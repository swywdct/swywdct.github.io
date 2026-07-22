---
title: 'Rust 三个 CLI 练手项目'
description: '使用 Rust 实现批量重命名和 Markdown 转 HTML 两个命令行工具。'
pubDate: '2026-07-23'
---

## 一、批量重命名文件

使用 clap 进行命令行参数解析，walkdir进行文件夹递归扫描，regax进行正则匹配。
支持匹配名称替换，添加前后缀，添加序号，可预览结果。

```rust
use std::path::PathBuf;
use anyhow::Ok;
use clap::Parser;
use walkdir::WalkDir;
use regex::Regex;
use std::path::Path;
use std::fs;

#[derive(Parser, Debug)]
#[command(version, about = "批量重命名文件")]
struct Args {
    /// 要扫描的目录
    directory: PathBuf,

    /// 匹配文件名的正则表达式
    #[arg(long)]
    pattern: String,

    /// 替换模版
    #[arg(long)]
    replace: Option<String>,

    #[arg(long, default_value = "")]
    prefix: String,

    #[arg(long, default_value = "")]
    suffix: String,

    /// 是否给结果添加连续序号
    #[arg(long)]
    number: bool,

    /// 只打印结果，不改名
    #[arg(long)]
    dry_run: bool,
    
}
/// 搜索所有文件，添加到files内，进行排序，保证可重复性。
fn collect_files(root: &std::path::Path) -> anyhow::Result<Vec<std::path::PathBuf> >{
    let mut files = Vec::new();

    for entry in WalkDir::new(root) {
        let entry = entry?;
        if entry.file_type().is_file() {
            files.push(entry.into_path());
        } 
    }
    files.sort();
    Ok(files)
}

/// - `path`: 原始文件的完整路径（用于提取文件名和扩展名）。
/// - `re`: 编译好的正则表达式，用于匹配文件名中需要替换的部分。
/// - `replace`: 可选的替换文本。如果为 `Some(text)`，则将匹配到的部分替换为 `text`；
///              如果为 `None`，则不执行替换。
/// - `prefix`: 要添加到文件名前的前缀字符串（可为空）。
/// - `suffix`: 要添加到文件名后的后缀字符串（可为空）。
/// - `number`: 可选的序号值。如果为 `Some(n)`，则文件名主体将被格式化为序号（如 `001`）；
///              如果为 `None`，则保持原文件名主体不变。
///
/// # 返回
/// 返回一个新的 `String`，即按照上述规则组合后的文件名（不含路径）。
fn renamed_name(
    path: &Path,
    re: &Regex,
    replace: Option<&str>,
    prefix: &str,
    suffix: &str,
    number: Option<usize>,
) -> Option<String> {
    let old = path.file_name()?.to_str()?;
    if !re.is_match(old) { //正则匹配
        return None;
    }
    let base = match replace {
        Some(template) => re.replace(old, template).into_owned(),
        None => old.to_owned(),
    };

    //不为None时，加三位数前缀，如001_base.xxx
    let numbered = number.map_or(base.clone(), |n| format!("{n:03}_{base}"));
    Some(format!("{prefix}{numbered}{suffix}"))
}

/// 判断后重命名，并打印操作
fn apply_rename(old: &Path, new_name:& str, dry_run: bool) -> anyhow::Result<()> {
    let new_path = old.parent().unwrap_or(Path::new(".")).join(new_name);
    if new_path.exists() && new_path != old {
        anyhow::bail!("目标已存在：{}",new_path.display());
    }
    println!("{} -> {}", old.display() , new_path.display() );
    if !dry_run && new_path != old {
        fs::rename(old,new_path)?;
    }
    Ok(())
}


fn main() -> anyhow::Result<()> {
    let args = Args::parse();
    let re = regex::Regex::new(&args.pattern)?;
    let files = collect_files(&args.directory)?;
    let mut index = 1; //前缀序号

    for path in files {
        let number = args.number.then_some(index);
        if let Some(name) = renamed_name(
            &path, &re, args.replace.as_deref(),
            &args.prefix, &args.suffix, number,
        ) {
            apply_rename(&path, &name, args.dry_run)?;
            index += 1;
        }
    }
    Ok(())
}
```

## 二.markdown2html格式转换
使用clap进行命令行参数解析，pulldown\_cmark用来转化格式，tera创建模版
```rust
use std::path::PathBuf;
use clap::Parser;
use pulldown_cmark::{html, Parser as Ps};
use std::fs;
use tera::{Context, Tera};

#[derive(Parser, Debug)]
struct Args {
    input: PathBuf,

    #[arg(short, long)]
    output: PathBuf,

    #[arg(short,long,default_value = " ")]
    title: String,
}
/// 格式转化
fn markdown_to_html(path: &std::path::Path) -> anyhow::Result<String> {
    let markdown = fs::read_to_string(path)?;
    let parser = Ps::new(&markdown);
    let mut body = String::new();
    html::push_html(&mut body, parser); //md2html
    Ok(body)
}

/// 添加前缀模版
fn render_page(title: &str, body :&str) -> anyhow::Result<String> {
    let tera = Tera::new("templates/**/*")?;
    let mut context = Context::new();
    context.insert("title", title);
    context.insert("body", body);

    Ok(tera.render("page.html",&context)?)

}

fn main() -> anyhow::Result<()> {
    let args = Args::parse();
    let body = markdown_to_html(&args.input)?;
    let page = render_page(&args.title, &body)?;
    fs::write(&args.output, page)?;

    println!("已生成 {}", args.output.display());

    Ok(())
}
```
Eg.
```md
# aaa #
## bb ##
### cc ##
abcdefg
- aa
- bb
- cc

1. a
2. b
3. c

*asdasdasd*
```
to
```html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title> </title>
</head>
<body>
  <main class="markdown-body"><h1>aaa</h1>
<h2>bb</h2>
<h3>cc</h3>
<p>abcdefg</p>
<ul>
<li>aa</li>
<li>bb</li>
<li>cc</li>
</ul>
<ol>
<li>a</li>
<li>b</li>
<li>c</li>
</ol>
<p><em>asdasdasd</em></p>
</main>
</body>
</html>
```

## 三.端口扫描

多线程扫描目的ip的给定端口区间是否打开。
使用ProgressBar, ProgressStyle显示进度条，tokio.TcpStream做异步TCP

```rust
use clap::Parser;
use std::net::Ipv4Addr;
use indicatif::{ProgressBar, ProgressStyle};
use std::sync::Arc;
use std::time::Duration;
use tokio::net::TcpStream;
use tokio::sync::Semaphore;

#[derive(Parser, Debug)]
struct Args {
    host: Ipv4Addr,
    #[arg(long, default_value_t = 1)]
    start: u16,
    #[arg(long, default_value_t = 1024)]
    end: u16,
    #[arg(long, default_value_t = 100)]
    concurrency: usize,
    #[arg(long, default_value_t = 500)]
    timeout_ms: u64,
}
/// 原子化操作扫描
async fn scan(host:std::net::Ipv4Addr, start: u16, end: u16,concurrency: usize, timeout_ms:u64){
    let total = u64::from(end.saturating_add(start)) +1 ; //安全加法，不会溢出panic
    let bar = ProgressBar::new(total);                             //显示进度条
    bar.set_style(ProgressStyle::with_template("{bar:40}{pos}/{len}").unwrap());     //长度为40
    let limit = Arc::new(Semaphore::new(concurrency.max(1)));//线程最小为1
    let mut tasks = Vec::new();

    for port in start..=end {
        let permit = Arc::clone(&limit).acquire_owned().await.unwrap(); // 获得许可
        let bar = bar.clone(); 
        tasks.push(tokio::spawn(async move {            // 推入新的异步任务
            let addr = format!("{host}:{port}");    // 拼接
            let result = tokio::time::timeout( 
                Duration::from_millis(timeout_ms), TcpStream::connect(&addr)
            ).await;
            if matches!(result, Ok(Ok(_))) { //内外层全部Ok
                println!("open {addr}");
            }
            bar.inc(1); // 进度条增加1
            drop(permit); // 显式释放
        }));
    }
    for task in tasks { // 等待所有任务完成
        let _ = task.await;
    }
    bar.finish_with_message("scan finish");
}

#[tokio::main]
async fn main() {
    let args = Args::parse();
    
    if args.start > args.end { 
        eprintln!("start must <= end");
        std::process::exit(2);
    }
    scan(args.host, args.start, args.end, args.concurrency, args.timeout_ms).await;
}
```
