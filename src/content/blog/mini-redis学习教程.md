# 从零读懂 Tokio mini-redis：用 Rust 构建一个异步 Redis

> 本文是一篇面向学习者的源码导读，基于当前仓库中的 `mini-redis` 实现编写。
> 项目不是生产级 Redis，而是 Tokio 官方用于演示异步 Rust 编程模式的教学项目。
[https://github.com/tokio-rs/mini-redis][1]
## 1. 这篇教程要解决什么问题

如果只看一个 TCP echo server，很难同时理解异步运行时、协议编解码、共享状态、消息广播、优雅停机和异步测试。mini-redis 把这些概念放进了一个规模适中的项目中：

- 客户端通过 TCP 发送 Redis RESP 协议数据。
- 服务端读取字节，解析为 `Frame`。
- `Frame` 被转换为 `Command`，再交给具体命令执行。
- `GET` 和 `SET` 访问共享的内存数据库。
- `SET` 可以带 TTL，后台任务负责过期删除。
- `PUBLISH` 和 `SUBSCRIBE` 通过 Tokio channel 实现消息分发。
- 服务端限制连接数，并在收到 Ctrl-C 后等待现有连接收尾。

读完后，你应该能回答这些问题：

1. TCP 为什么需要自己处理“半个消息”和“多个消息粘在一起”的情况？
2. `Frame::check`、`Frame::parse`、`Connection::read_frame` 分别负责什么？
3. 一个 `SET foo bar` 命令如何从客户端方法走到数据库？
4. 为什么这里使用 `std::sync::Mutex`，而不是 `tokio::sync::Mutex`？
5. 多个订阅者如何收到同一个频道的消息？
6. 一个连接关闭时，服务端如何释放连接许可并完成优雅停机？

---

## 2. 项目概览

### 2.1 项目能力

当前项目支持以下命令：

| 命令                              | 作用                     |
| ------------------------------- | ---------------------- |
| `PING [message]`                | 无参数返回 `PONG`，有参数原样返回消息 |
| `GET key`                       | 读取键值，不存在时返回空值          |
| `SET key value`                 | 写入键值                   |
| `SET key value EX seconds`      | 写入键值并按秒设置 TTL          |
| `SET key value PX milliseconds` | 写入键值并按毫秒设置 TTL         |
| `PUBLISH channel message`       | 向频道发布消息                |
| `SUBSCRIBE channel ...`         | 订阅一个或多个频道              |
| `UNSUBSCRIBE channel ...`       | 在订阅模式下取消订阅             |

它没有实现 Redis 的完整命令集、持久化、集群和生产级安全能力。这个限制是刻意的：项目的重点是 Tokio 的异步编程模式，而不是复制 Redis 的全部功能。

### 2.2 目录结构

```text
mini-redis/
├── Cargo.toml
├── README.md
├── src/
│   ├── lib.rs                  # 库入口、公共类型和模块声明
│   ├── frame.rs                # RESP 帧定义、校验和解析
│   ├── connection.rs           # TCP 字节流与 Frame 之间的适配层
│   ├── parse.rs                # 从数组帧中读取字符串、字节和整数
│   ├── db.rs                   # 共享内存数据库、TTL 和 Pub/Sub 状态
│   ├── server.rs               # TCP 监听、连接任务、优雅停机
│   ├── shutdown.rs             # 每个连接的关闭信号状态
│   ├── cmd/                    # 命令解析与执行
│   │   ├── mod.rs
│   │   ├── get.rs
│   │   ├── set.rs
│   │   ├── ping.rs
│   │   ├── publish.rs
│   │   ├── subscribe.rs
│   │   └── unknown.rs
│   ├── clients/                # 异步、缓冲和阻塞客户端
│   │   ├── client.rs
│   │   ├── buffered_client.rs
│   │   ├── blocking_client.rs
│   │   └── mod.rs
│   └── bin/
│       ├── server.rs           # mini-redis-server
│       └── cli.rs              # mini-redis-cli
├── examples/
│   ├── hello_world.rs
│   ├── pub.rs
│   ├── sub.rs
│   └── chat.rs
└── tests/                      # 协议、客户端、服务端和 TTL 测试
```

### 2.3 一次普通请求的调用链

以 `GET hello` 为例：

```text
Client::get("hello")
    │
    ├─ Get::new("hello").into_frame()
    │      └─ Array([Bulk("get"), Bulk("hello")])
    │
    ├─ Connection::write_frame()
    │      └─ 编码为 RESP 字节并写入 TcpStream
    │
    └─ 服务端 Handler::run()
           ├─ Connection::read_frame()
           ├─ Command::from_frame()
           ├─ Get::apply(&db, &mut connection)
           ├─ Db::get("hello")
           └─ Connection::write_frame()
                  └─ 返回 Bulk(value) 或 Null
```

整个设计的关键是分层：协议层不关心命令语义，命令层不关心 TCP 粘包，数据库层不关心 RESP 编码。

---

## 3. 准备环境并运行项目

### 3.1 环境要求

需要安装：

- Rust stable 工具链
- Cargo

确认环境：

```bash
rustc --version
cargo --version
```

### 3.2 启动服务端

在项目根目录执行：

```bash
RUST_LOG=debug cargo run --bin mini-redis-server
```

默认监听 `127.0.0.1:6379`。服务端使用 `tracing` 输出结构化日志，`RUST_LOG=debug` 会显示连接和命令处理信息。

也可以指定端口：

```bash
RUST_LOG=debug cargo run --bin mini-redis-server -- --port 6380
```

### 3.3 使用 CLI

另开终端：

```bash
cargo run --bin mini-redis-cli -- ping
cargo run --bin mini-redis-cli -- ping hello

cargo run --bin mini-redis-cli -- set foo bar
cargo run --bin mini-redis-cli -- get foo

# 过期时间的单位是毫秒
cargo run --bin mini-redis-cli -- set short-lived value 1000
```

注意 Cargo 的参数分隔符 `--`：前一个 `--` 表示后面的参数传给二进制程序，而不是传给 Cargo。

### 3.4 运行示例

```bash
cargo run --example hello_world
```

发布订阅需要三个终端：

```bash
# 终端 1
RUST_LOG=debug cargo run --bin mini-redis-server

# 终端 2
cargo run --example sub

# 终端 3
cargo run --example pub
```

`sub` 会订阅 `foo`，`pub` 会向 `foo` 发布 `bar`。

### 3.5 运行测试

```bash
cargo test
```

项目的测试覆盖了三种层次：

- `tests/frame_validation.rs`：验证 RESP 帧的协议校验。
- `tests/server.rs`：直接使用原始 TCP 字节测试服务端，包括 TTL 和 Pub/Sub。
- `tests/client.rs`、`tests/buffered_client.rs`：从客户端 API 角度测试功能。

---

## 4. Cargo 配置：依赖为什么这样选

`Cargo.toml` 里的依赖可以按职责分组：

| 依赖                   | 用途                                     |
| -------------------- | -------------------------------------- |
| `tokio`              | 异步运行时、TCP、channel、同步原语、定时器和信号          |
| `bytes`              | 高效处理字节缓冲和共享字节数据                        |
| `tokio-stream`       | 把异步接收器包装成 Stream，并使用 `StreamMap` 合并多个流 |
| `async-stream`       | 用宏方便地创建异步 Stream                       |
| `atoi`               | 直接从字节解析十进制整数                           |
| `clap`               | 使用派生宏解析 CLI 参数                         |
| `tracing`            | 结构化日志与 span                            |
| `tracing-subscriber` | 配置日志过滤和输出                              |

项目还提供可选的 `otel` feature，用于将 tracing 数据接入 OpenTelemetry。它不是理解核心代码的前置条件。

---

## 5. RESP 协议：TCP 上的消息长什么样

### 5.1 TCP 只提供字节流

TCP 不保留应用层消息边界。一次 `write_all` 不保证对应一次 `read`：

- 一个请求可能被拆成多次读取。
- 多个请求可能一次性被读取到缓冲区。
- 请求可能只收到一半，连接仍然保持正常。

因此服务端不能简单地调用一次 `read`，然后假设得到完整请求。`Connection` 必须维护一个读缓冲区，并持续读取直到完整帧出现。

### 5.2 RESP 的常用类型

mini-redis 使用 Redis 的 RESP 风格编码：

```text
+OK\r\n                         # Simple String
-ERR unknown command\r\n       # Error
:42\r\n                        # Integer
$5\r\nhello\r\n                # Bulk String
$-1\r\n                       # Null Bulk String
*2\r\n$3\r\nGET\r\n$5\r\nfoo\r\n  # Array
```

客户端发送 `SET foo bar`：

```text
*3\r\n
$3\r\n
SET\r\n
$3\r\n
foo\r\n
$3\r\n
bar\r\n
```

拼成连续字节后就是：

```text
*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n
```

### 5.3 `Frame`：协议的中间表示

`src/frame.rs` 定义：

```rust
pub enum Frame {
    Simple(String),
    Error(String),
    Integer(u64),
    Bulk(Bytes),
    Null,
    Array(Vec<Frame>),
}
```

这一步很重要。命令实现不需要到处处理 `b'*'`、`\r\n` 和长度字段，只需要处理结构化数据：

```rust
Frame::Array(vec![
    Frame::Bulk(Bytes::from_static(b"set")),
    Frame::Bulk(Bytes::from_static(b"foo")),
    Frame::Bulk(Bytes::from_static(b"bar")),
])
```

`Bulk(Bytes)` 使用 `Bytes` 而不是 `String`，因为 Redis value 可以是任意字节，不一定是合法 UTF-8。只有命令名、key 和 channel 这类需要文本语义的字段，才会在 `parse.rs` 中转换为字符串。

### 5.4 为什么先 `check` 再 `parse`

`Frame::check` 只检查一帧是否完整和格式合法，不立即构造完整的 `Frame` 对象。`Frame::parse` 在确认数据完整后才真正分配和构建结构。

两步解析的收益：

1. 不完整数据是网络中的正常状态，不应被当作协议错误。
2. 在半帧到达时，不需要创建临时对象。
3. `check` 返回后，游标位置可以直接给出完整帧的字节长度。

`check` 区分两类错误：

- `Incomplete`：继续从 socket 读取。
- `Other(...)`：协议格式错误，关闭当前连接。

比如 `$-1\r\n` 是合法的 Null，而 `$-2\r\n` 是非法的负长度。对应测试见 `tests/frame_validation.rs`。

### 5.5 `Connection::read_frame` 如何处理半帧

`src/connection.rs` 中的核心逻辑可以概括为：

```rust
loop {
    if let Some(frame) = self.parse_frame()? {
        return Ok(Some(frame));
    }

    if self.stream.read_buf(&mut self.buffer).await? == 0 {
        if self.buffer.is_empty() {
            return Ok(None);
        }
        return Err("connection reset by peer".into());
    }
}
```

`BytesMut` 保存跨多次读取的数据。`parse_frame` 创建 `Cursor`，先调用 `Frame::check`：

- 成功：记录游标位置 `len`，调用 `Frame::parse`，然后 `self.buffer.advance(len)` 丢弃已消费的字节。
- `Incomplete`：返回 `Ok(None)`，外层继续读 socket。
- 其他错误：结束当前连接。

如果一次读入了两条命令，第一条解析后只丢弃第一帧，第二条仍留在 `BytesMut` 中，下一次 `read_frame` 会继续处理它。

### 5.6 `Connection::write_frame` 如何编码

写出流程与读入相反：

1. 数组先写 `*长度\r\n`。
2. 逐项调用 `write_value`。
3. Simple、Error、Integer、Bulk 和 Null 分别写对应前缀。
4. 所有内容写入 `BufWriter<TcpStream>`。
5. 最后 `flush`，确保响应真正发到 socket。

这里的 `BufWriter` 可以减少大量小写入导致的系统调用。需要注意，当前实现只在顶层处理数组，作为数组元素的嵌套数组暂未支持。

---

## 6. 从 Frame 到 Command：类型化命令解析

### 6.1 `Parse` 是一个小型游标

RESP 命令通常是数组：第一个元素是命令名，后面是参数。`src/parse.rs` 把 `Vec<Frame>` 包装成迭代器，并提供：

- `next_string()`：读取文本字段并校验 UTF-8。
- `next_bytes()`：读取任意字节字段。
- `next_int()`：读取整数帧或解析数字字符串。
- `finish()`：确认数组中没有多余参数。

这让命令模块不用自己维护数组下标，也能统一处理参数类型错误和参数不足。

### 6.2 `Command::from_frame`

`src/cmd/mod.rs` 的分发流程是：

```rust
let mut parse = Parse::new(frame)?;
let command_name = parse.next_string()?.to_lowercase();

let command = match command_name.as_str() {
    "get" => Command::Get(Get::parse_frames(&mut parse)?),
    "set" => Command::Set(Set::parse_frames(&mut parse)?),
    "ping" => Command::Ping(Ping::parse_frames(&mut parse)?),
    "publish" => Command::Publish(Publish::parse_frames(&mut parse)?),
    "subscribe" => Command::Subscribe(Subscribe::parse_frames(&mut parse)?),
    "unsubscribe" => Command::Unsubscribe(Unsubscribe::parse_frames(&mut parse)?),
    _ => Command::Unknown(Unknown::new(command_name)),
};

parse.finish()?;
Ok(command)
```

命令名转小写，所以 `GET`、`Get` 和 `get` 都可以被识别。已知命令必须通过 `finish()`，例如 `GET key extra` 会因为多余参数而报协议错误。

未知命令是一个特殊情况：它直接构造 `Unknown`，不要求继续消费剩余参数，因为服务端只需要返回“未知命令”。

### 6.3 命令的统一形状

每个命令文件通常都有三类方法：

```text
parse_frames(&mut Parse) -> CommandType
apply(&self, db, connection, ...) -> Result
into_frame(self) -> Frame
```

- `parse_frames`：服务端方向，把收到的 Frame 变成命令对象。
- `apply`：服务端方向，执行命令并写回响应。
- `into_frame`：客户端方向，把命令对象编码成请求 Frame。

这使同一命令的协议格式集中在一个模块中，客户端和服务端不会各自复制一份格式逻辑。

### 6.4 `GET` 和 `SET`

`GET::apply` 调用 `db.get`：

- 找到值：返回 `Frame::Bulk(value)`。
- 找不到值：返回 `Frame::Null`，编码为 `$-1\r\n`。

`SET::apply` 调用 `db.set`，成功后返回 `Frame::Simple("OK")`，编码为 `+OK\r\n`。

客户端的 `Client::get` 再把响应映射回 Rust 类型：

```text
Simple 或 Bulk -> Some(Bytes)
Null            -> None
其他 Frame      -> 错误
```

### 6.5 TTL 的协议表示

客户端 `set_expires` 使用 `PX` 和毫秒：

```text
SET key value PX 1000
```

`Set::into_frame` 将 `Duration` 转换为 `as_millis()` 的十进制字符串。服务端解析 `PX` 后，把毫秒转换为 `Duration`。

服务端协议解析也支持直接发送 `EX seconds`，但项目提供的 `Client::set_expires` 和 CLI 为了保留更细的时间精度，统一使用 `PX`。例如原始 RESP 请求可以写成：

```text
*5\r\n$3\r\nSET\r\n$5\r\nhello\r\n$5\r\nworld\r\n$2\r\nEX\r\n:1\r\n
```

项目没有在 `GET` 时做主动惰性删除，而是由数据库后台任务负责清理。因此 `GET` 的结果依赖后台任务是否已经处理到期点，但测试可以用 Tokio 的时间控制来稳定验证行为。

---

## 7. 服务端：监听器、连接任务和连接数限制

### 7.1 `server::run` 的总体结构

`src/server.rs` 的 `run` 接收两个参数：

```rust
pub async fn run(listener: TcpListener, shutdown: impl Future)
```

调用者可以传入 `tokio::signal::ctrl_c()`，也可以在测试中传入其他 future。

服务端初始化：

```text
DbDropGuard
Semaphore(MAX_CONNECTIONS = 250)
broadcast::Sender<()>       # 通知所有连接关闭
mpsc::Sender<()>            # 等待所有连接完成
```

然后使用 `tokio::select!` 并发等待两件事：

- `server.run()`：持续接受连接。
- `shutdown` future：外部关闭信号完成。

谁先完成，谁触发下一步处理。

### 7.2 每连接一个 Tokio task

监听器接受 socket 后，创建 `Handler`：

```text
Handler {
    db: Db,
    connection: Connection,
    shutdown: Shutdown,
    _shutdown_complete: mpsc::Sender<()>,
}
```

随后：

```rust
tokio::spawn(async move {
    if let Err(err) = handler.run().await {
        error!(cause = ?err, "connection error");
    }
    drop(permit);
});
```

每个连接独立运行，所以一个客户端的协议错误只会终止该连接，不会直接杀掉整个服务器。共享数据库句柄是廉价 clone，因为 `Db` 内部是 `Arc<Shared>`。

### 7.3 Semaphore 限制并发连接

监听循环在 `accept` 之前获取 semaphore permit：

```rust
let permit = self.limit_connections
    .clone()
    .acquire_owned()
    .await
    .unwrap();
```

连接处理 task 结束时 permit 被 drop，自动归还 semaphore。

这意味着达到 250 个连接后，服务端会暂停接受新连接，直到已有连接结束。这个模式适用于限制 socket、数据库连接或其他有限资源的占用。

### 7.4 accept 错误的指数退避

`Listener::accept` 对暂时性 accept 错误进行重试，等待时间按 1、2、4、8、16、32、64 秒增长。超过阈值后放弃并让服务端停止。

指数退避的目的不是让所有错误都消失，而是处理例如操作系统短暂达到文件描述符上限这类可能自行恢复的情况，同时避免错误循环持续占满 CPU。

---

## 8. 共享数据库：Arc、Mutex、Bytes 和 TTL

### 8.1 数据结构

数据库的核心状态是：

```rust
struct State {
    entries: HashMap<String, Entry>,
    pub_sub: HashMap<String, broadcast::Sender<Bytes>>,
    expirations: BTreeSet<(Instant, String)>,
    shutdown: bool,
}

struct Entry {
    data: Bytes,
    expires_at: Option<Instant>,
}
```

这里有两个独立命名空间：

- `entries`：key-value 数据。
- `pub_sub`：频道到广播发送器的映射。

频道名和 key 名相同并不冲突，例如 `SET foo value` 与 `PUBLISH foo message` 互不影响。

### 8.2 为什么使用 `Arc<Shared>`

服务端每个连接都需要访问同一个数据库。`Db` 是一个可 clone 的句柄：

```text
Db
└── Arc<Shared>
    ├── Mutex<State>
    └── Notify
```

clone `Db` 只增加引用计数，不复制 HashMap。连接 task、后台 TTL task 和服务端本身共享同一份状态。

### 8.3 为什么是 `std::sync::Mutex`

项目特别展示了一个很容易被误解的原则：异步代码不等于所有锁都必须用 Tokio mutex。

这里临界区只做同步内存操作：查 HashMap、插入 HashMap、操作 BTreeSet、发送 broadcast。锁内部没有 `.await`，而且临界区很短，所以 `std::sync::Mutex` 更简单，也通常更合适。

应该考虑 `tokio::sync::Mutex` 的场景是：必须跨 `.await` 持有锁，或者需要把等待锁的过程纳入异步调度。即使如此，也要仔细评估是否真的需要跨 await 持锁，因为那会扩大争用范围。

绝不能在异步 task 中长时间持有同步锁并做阻塞工作。如果临界区很重，应考虑缩短临界区或使用 `spawn_blocking`。

### 8.4 为什么使用 `Bytes`

`Bytes` 很适合网络数据：

- 可以保存任意字节，不要求 UTF-8。
- clone 通常是浅拷贝，多个地方可以共享底层数据。
- 在连接读写、数据库存储和 broadcast 之间传递时减少不必要复制。

比如 `Db::get` 只 clone `entry.data` 的 `Bytes` 句柄，而不是复制整个 value。

### 8.5 用 BTreeSet 管理过期时间

每个带 TTL 的 key 都插入：

```text
(expires_at, key)
```

使用元组而不是只存 `Instant`，是因为多个 key 可能同时到期。`String` 作为第二排序字段，用来打破相同时间的平局。

`BTreeSet` 的最小元素就是最早到期的 key，后台任务只需要关注它，不必每次扫描全部 HashMap。

更新 key 时，项目会：

1. 插入新 `Entry`。
2. 如果旧值有 TTL，从 `expirations` 移除旧的 `(when, key)`。
3. 如果新值有 TTL，插入新的 `(when, key)`。
4. 如果新 TTL 成为最早到期项，通知后台任务重新计算等待时间。

这个“删除旧索引，再插入新索引”的顺序很重要，否则覆盖同一个 key 时可能留下过期索引。

### 8.6 TTL 后台任务

`Db::new` 创建共享状态后立即 spawn：

```rust
tokio::spawn(purge_expired_tasks(shared.clone()));
```

后台任务的状态机可以表示为：

```text
检查 shutdown
    │
    ├─ 有已到期 key：删除后继续检查
    │
    ├─ 有未来 key：sleep_until(最早到期时间)
    │              或被 Notify 提前唤醒
    │
    └─ 没有到期 key：等待 Notify
```

为什么需要 `Notify`？如果任务已经在等待一个较晚的 key，而此时新插入了一个更早到期的 key，`Db::set` 会调用 `notify_one()`，让任务重新检查排序后的过期列表。

为什么需要 `DbDropGuard`？后台任务本身持有 `Arc<Shared>`，不能只依赖引用计数自动结束。`DbDropGuard` 在服务器退出时显式设置 `shutdown = true` 并通知后台任务，让它能够退出循环。

---

## 9. Pub/Sub：broadcast 与 StreamMap 的组合

### 9.1 发布端

每个频道对应一个 `broadcast::Sender<Bytes>`：

```text
pub_sub: HashMap<频道名, broadcast::Sender<消息>>
```

`Db::publish` 找到频道 sender 后调用 `send(value)`。返回值是当前 receiver 数量；没有频道或没有订阅者时返回 0。

这个数字只是发布瞬间的订阅者数量，不保证每个订阅者最终一定读到消息，因为客户端可能在读取前断开。

### 9.2 订阅端

一个客户端可以订阅多个频道。每个频道有独立的 `broadcast::Receiver<Bytes>`，服务端把每个 receiver 包装成 Stream，再放入 `StreamMap<String, Messages>`：

```text
StreamMap {
    "foo"   -> foo 的消息流
    "bar"   -> bar 的消息流
}
```

`subscriptions.next()` 会从任意一个活跃流取出：

```text
(channel_name, message)
```

服务端随后编码为：

```text
*3\r\n
$7\r\nmessage\r\n
$3\r\nfoo\r\n
$5\r\nhello\r\n
```

### 9.3 订阅模式是一个不同的连接状态

普通连接循环只需要等待“客户端请求”：

```text
read_frame -> parse command -> apply -> response
```

进入订阅模式后，服务端同时等待三类事件：

```rust
tokio::select! {
    Some((channel, msg)) = subscriptions.next() => { ... }
    frame = dst.read_frame() => { ... }
    _ = shutdown.recv() => { ... }
}
```

也就是：

1. 订阅频道有新消息。
2. 客户端发送新的订阅或取消订阅命令。
3. 服务端收到关闭信号。

这正是 `select!` 适合表达的场景：多个异步事件源竞争，谁先就绪就先处理谁。

### 9.4 动态订阅和取消订阅

收到新的 `SUBSCRIBE` 后，频道名先放入 `self.channels`，主循环下一次迭代再建立真实订阅并发送确认帧。

收到 `UNSUBSCRIBE` 时：

- 指定频道：从 `StreamMap` 移除对应流。
- 不指定频道：复制当前所有频道名，逐个移除。

每次变更都会发送确认数组，例如：

```text
*3\r\n$9\r\nsubscribe\r\n$5\r\nhello\r\n:1\r\n
*3\r\n$11\r\nunsubscribe\r\n$5\r\nhello\r\n:0\r\n
```

项目在 `broadcast::Receiver::Lagged` 时选择跳过丢失的旧消息并继续接收。这体现了广播 channel 的语义：它更适合实时通知，而不是要求每条消息都可靠持久化的队列。

### 9.5 类型状态防止错误调用

异步客户端调用 `subscribe` 后，原来的 `Client` 被消耗，返回 `Subscriber`：

```rust
let client = Client::connect("127.0.0.1:6379").await?;
let mut subscriber = client.subscribe(vec!["foo".into()]).await?;
```

这样进入订阅状态后，普通 `get`、`set` 方法在类型层面就不可用。这是 Rust 类型系统表达协议状态的一种很实用的方式。

---

## 10. 优雅停机：广播通知加 channel 关闭检测

服务端的停机过程分成两层：

### 10.1 通知所有连接

主服务收到 `shutdown` future 后，drop `notify_shutdown` sender。每个 Handler 都持有一个 receiver，并在读取请求时使用：

```rust
tokio::select! {
    res = self.connection.read_frame() => res?,
    _ = self.shutdown.recv() => return Ok(()),
}
```

连接完成当前安全状态后退出。订阅模式也会监听相同信号。

### 10.2 等待所有连接结束

每个 `Handler` 持有 `shutdown_complete_tx` 的 clone。主监听器先 drop 自己的 sender，然后等待 receiver 收到 channel 关闭：

```rust
drop(shutdown_complete_tx);
let _ = shutdown_complete_rx.recv().await;
```

当所有 Handler 被 drop，最后一个 sender 消失，receiver 返回 `None`，服务端才真正退出。

这里不是等待固定时间，也不是强制 abort 所有 task，而是利用 channel 的生命周期表达“所有工作已完成”。

### 10.3 `Shutdown` 为什么需要布尔状态

`Shutdown` 包装 broadcast receiver，并记录 `is_shutdown`：

- 第一次 `recv()` 等待广播信号。
- 收到后设置 `is_shutdown = true`。
- 后续调用立即返回，不重复等待。

这让同一个连接在普通模式和订阅模式之间切换时仍能正确感知已经收到的停机信号。

---

## 11. 客户端实现：异步、缓冲和阻塞三种接口

### 11.1 异步 `Client`

`src/clients/client.rs` 持有一个 `Connection`。它的 API 与服务端命令对应：

```rust
let mut client = Client::connect("127.0.0.1:6379").await?;

client.set("foo", "bar".into()).await?;
let value = client.get("foo").await?;
let pong = client.ping(None).await?;
let count = client.publish("news", "hello".into()).await?;
```

当前设计要求 `&mut self`，因为一条 TCP 连接上的请求和响应必须按顺序配对，且项目没有实现同一连接上的并发 pipelining。

典型方法都遵循：

```text
构造命令对象
    -> into_frame()
    -> connection.write_frame()
    -> connection.read_frame()
    -> 校验响应类型
    -> 转换成 Rust 返回值
```

### 11.2 `BufferedClient`：用消息传递共享一条连接

如果多个 Tokio task 想共享一个 `Client`，直接 clone 不行，因为底层连接的读写必须串行，且方法需要可变借用。

`BufferedClient` 的解决方案是：

```text
调用者 task
    │ Command + oneshot::Sender
    ▼
mpsc channel (容量 32)
    │
    ▼
专用连接 task
    │ 调用真实 Client
    │
    └── oneshot::Sender 返回结果
```

每次调用创建一个 `oneshot`：

```rust
let (tx, rx) = oneshot::channel();
self.tx.send((command, tx)).await?;
rx.await??
```

`BufferedClient` 自身可以 clone，因此多个任务共享的是发送句柄，而不是直接共享 socket。专用 task 依次从 `mpsc` 中取请求，保证一条连接上永远只有一个顺序执行者。

这是一种常见的 actor 风格：状态和资源由一个 task 独占，其他 task 通过消息访问它。

### 11.3 `BlockingClient`：在同步 API 外包一层 runtime

`BlockingClient` 内部仍然使用异步 `Client`，只是创建一个 current-thread Tokio runtime，并在每个同步方法中调用：

```rust
self.rt.block_on(self.inner.get(key))
```

因此它适合没有 Tokio runtime 的普通同步程序：

```rust
let mut client = BlockingClient::connect("127.0.0.1:6379")?;
client.set("foo", "bar".into())?;
println!("{:?}", client.get("foo")?);
```

要注意：不要在已经运行 Tokio runtime 的异步 task 内调用这种阻塞 API，否则可能造成运行时嵌套或阻塞线程。异步程序优先使用 `Client`，同步程序再使用 `BlockingClient`。

---

## 12. CLI 和示例：如何作为阅读入口

### 12.1 `src/bin/server.rs`

服务端二进制主要做三件事：

1. 初始化 tracing。
2. 绑定 `127.0.0.1:6379` 或命令行指定端口。
3. 调用 `server::run(listener, tokio::signal::ctrl_c())`。

复杂的连接处理放在库模块中，这样测试可以直接调用 `server::run`，而不是启动外部进程。

### 12.2 `src/bin/cli.rs`

CLI 使用 Clap 派生 `Parser` 和 `Subcommand`，将命令行输入映射为 Rust enum。执行时再调用异步 `Client`。

它还体现了一个实用的错误边界：

- CLI 参数错误由 Clap 或自定义解析器报告。
- 网络错误和协议错误通过项目统一的 `mini_redis::Result` 返回。
- 二进制入口只负责展示结果，不把协议细节泄漏给用户。

### 12.3 `examples/hello_world.rs`

这是最适合第一遍运行的示例：

```rust
let mut client = Client::connect("127.0.0.1:6379").await?;
client.set("hello", "world".into()).await?;
let result = client.get("hello").await?;
```

建议在阅读完整代码前，先运行它，然后对照 `Client::set`、`Client::get`、`Set::into_frame` 和 `Get::apply` 追踪请求。

---

## 13. 测试设计：从字节级到 API 级

### 13.1 原始 TCP 测试

`tests/server.rs` 不使用项目客户端，而是直接写入 RESP 字节：

```rust
stream
    .write_all(b"*2\r\n$3\r\nGET\r\n$5\r\nhello\r\n")
    .await?;
```

这样可以独立验证服务端的协议层和命令执行层。如果客户端和服务端共用同一套编码逻辑，客户端测试可能掩盖双方同时存在的编码错误；原始字节测试能够减少这种风险。

### 13.2 客户端 API 测试

`tests/client.rs` 使用 `Client` 验证用户实际会调用的接口，例如：

- 无参数和带消息的 `PING`。
- `SET` 后 `GET`。
- 单频道和多频道订阅。
- 取消全部订阅。

### 13.3 可控时间测试

TTL 测试调用：

```rust
tokio::time::pause();
tokio::time::advance(Duration::from_secs(1)).await;
```

时间是异步测试中的非确定性来源。直接 `sleep` 会让测试变慢，也可能因机器负载产生抖动。暂停 Tokio 时间后手动推进，可以让测试快速且可重复。

### 13.4 BufferedClient 测试

`tests/buffered_client.rs` 验证缓冲客户端仍然能正确执行 `SET` 和 `GET`。进一步学习时，可以补充多个 task 并发调用同一个 `BufferedClient` 的测试，以验证 mpsc 排队和 oneshot 返回路径。

---

## 14. 建议的源码阅读顺序

为了避免一开始陷入细节，推荐按下面顺序阅读：

1. `examples/hello_world.rs`：先看用户如何使用客户端。
2. `src/clients/client.rs`：看客户端如何构造请求和解析响应。
3. `src/cmd/get.rs`、`src/cmd/set.rs`：看命令的双向转换。
4. `src/frame.rs`：理解 RESP 数据结构和解析规则。
5. `src/connection.rs`：理解 TCP 缓冲、半帧和多帧。
6. `src/parse.rs`、`src/cmd/mod.rs`：理解 Frame 到 Command 的分发。
7. `src/server.rs`：把服务端主循环串起来。
8. `src/db.rs`：学习共享状态和 TTL 后台任务。
9. `src/cmd/subscribe.rs`：学习多个异步流和订阅状态机。
10. `src/shutdown.rs`：最后理解停机信号如何贯穿连接生命周期。

阅读每个模块时可以问自己两个问题：

- 这个模块输入和输出的抽象分别是什么？
- 它是否需要知道 TCP、协议、命令或数据库的内部细节？

如果一个模块知道了太多层，通常说明边界正在变差。

---

## 15. 一次请求的完整时序

下面用 `SET hello world` 展示各层协作：

```text
客户端                    服务端 Handler                  数据库
  │                             │                            │
  │ Set::into_frame             │                            │
  │ Array(Bulk, Bulk, Bulk)     │                            │
  │                             │                            │
  │ write_frame                 │                            │
  │ -- RESP bytes ------------> │                            │
  │                             │ read_frame                 │
  │                             │ Frame::parse               │
  │                             │ Command::from_frame        │
  │                             │ Set::parse_frames          │
  │                             │                            │
  │                             │ Db::set -----------------> │
  │                             │                            │ entries.insert
  │                             │                            │ TTL index update
  │                             │                            │
  │                             │ <----- Simple("OK")       │
  │ <------ +OK\r\n ------------ │                            │
  │                             │                            │
  │ read_frame                  │                            │
  │ Frame::Simple("OK")        │                            │
```

这里没有跨 await 持有数据库 mutex：命令调用 `Db::set` 时完成短暂同步操作，之后才异步写响应。这是并发性能和正确性的关键点之一。

---

## 16. 容易踩坑的地方

### 16.1 把 TCP read 当成消息读取

错误思路：一次 `read` 得到一个完整 Redis 命令。正确思路是维护缓冲区，并通过协议长度和终止符判断完整帧。

### 16.2 把所有字段都解析成 String

value 和 Pub/Sub message 可以是任意二进制数据。只有命令名、key、频道名等需要文本解释的字段才应调用 `next_string()`。

### 16.3 在 mutex 内 await

这会让其他 task 长时间无法访问状态，严重时会形成死锁或吞吐下降。mini-redis 的设计是先在锁内完成同步状态变更，再释放锁，最后调用 `Notify`。

### 16.4 忘记清理旧 TTL

覆盖一个已有过期时间的 key 时，如果只更新 `HashMap` 而不移除旧的 BTreeSet 索引，过期任务会访问错误的索引，或者积累无效记录。

### 16.5 让订阅者阻塞发布者

Pub/Sub 使用 broadcast channel，并设置容量。慢订阅者可能丢失旧消息，但不会让发布者无限等待。这是实时通知和可靠消息队列之间的语义取舍。

### 16.6 把 BlockingClient 用在异步上下文

阻塞接口适合同步代码。在 Tokio task 中使用它会阻塞执行线程，应改用异步 `Client`，或将同步工作放到专用阻塞线程。

### 16.7 把项目当作生产 Redis

当前实现没有持久化、认证、ACL、集群、高级数据结构、完整 RESP 版本支持和生产级资源治理。它最适合作为学习 Tokio 和网络服务设计的样例。

---

## 17. 可以尝试的练习

建议按难度逐步完成：

### 练习一：增加 `DEL`

需要修改：

1. `src/cmd/mod.rs` 增加 `Del` 变体。
2. 新建 `src/cmd/del.rs`。
3. `Db` 增加删除 key 的方法。
4. 客户端增加 `del` 方法。
5. 为不存在 key、存在 key 和多 key 添加测试。

思考：返回值是删除数量，应该用 `Frame::Integer`。

### 练习二：增加 `INCR`

需要处理：

- value 是否能解析为整数。
- 读、加一、写回是否需要在同一个锁临界区内完成。
- 并发连接同时执行时如何避免丢失更新。

这个练习能让你理解“先 `GET` 再 `SET`”为什么不具备原子性。

### 练习三：实现更严格的参数验证

可以为已知命令增加：

- 明确的参数数量检查。
- 更清晰的错误响应。
- 对 key 和 channel 的长度限制。
- 对 bulk string 最大长度的限制。

这也是从教学项目走向可暴露服务时必须考虑的安全边界。

### 练习四：补全 `BufferedClient`

当前缓冲客户端只实现了 `GET` 和 `SET`。可以把 `PING`、`PUBLISH` 或 TTL 版本加入 command enum，并让响应类型变成更通用的枚举，而不是只使用 `Result<Option<Bytes>>`。

### 练习五：增加协议单元测试

覆盖：

- 空数组。
- 多层数组。
- 半帧输入。
- 多帧连续输入。
- 非 UTF-8 bulk value。
- 非法长度和缺失 `\r\n`。

---

## 18. 总结：这个项目真正值得学的是什么

mini-redis 的价值不在于实现了多少 Redis 命令，而在于它把一组真实服务中经常同时出现的问题连接起来：

1. 用 `Frame` 隔离协议表示和业务命令。
2. 用 `BytesMut` 正确处理 TCP 的流式输入。
3. 用 `Arc` 共享状态，用短临界区的 `std::sync::Mutex` 保证一致性。
4. 用 `BTreeSet + Notify + sleep_until` 实现高效 TTL 清理。
5. 用 `broadcast` 和 `StreamMap` 实现多频道消息分发。
6. 用 Tokio task 和 semaphore 管理并发连接。
7. 用 broadcast shutdown 和 channel 生命周期实现优雅停机。
8. 用类型状态把“普通连接”和“订阅连接”区分开。
9. 用异步客户端、消息传递客户端和阻塞客户端适配不同调用场景。
10. 用原始协议测试、API 测试和可控时间测试降低异步系统的不确定性。

当你能够沿着 `Client::set -> Connection -> Frame -> Command -> Db -> Frame -> Client` 这条链路独立追踪一次请求时，就已经掌握了阅读这个项目的大部分方法。接下来可以把同样的思路迁移到 HTTP 服务、数据库连接池、消息代理或其他 Tokio 应用中。

---

## 附录：常用命令速查

```bash
# 格式检查
cargo fmt -- --check

# 编译检查
cargo check

# 运行全部测试
cargo test

# 启动服务端
RUST_LOG=debug cargo run --bin mini-redis-server

# CLI 操作
cargo run --bin mini-redis-cli -- ping
cargo run --bin mini-redis-cli -- set foo bar
cargo run --bin mini-redis-cli -- get foo
cargo run --bin mini-redis-cli -- publish foo hello
cargo run --bin mini-redis-cli -- subscribe foo

# 运行示例
cargo run --example hello_world
cargo run --example pub
cargo run --example sub
```

[1]:	https://github.com/tokio-rs/mini-redis