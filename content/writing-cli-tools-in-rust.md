+++
title = "Writing CLI Tools in Rust: Fast, Safe, and User-Friendly"
date = 2026-01-23
description = "Rust combines memory safety, excellent performance, and a rich ecosystem to make building command-line tools straightforward and reliable. Here's why Rust excels for CLI development and how to build tools users actually want to use."
[taxonomies]
tags = ["rust", "cli", "command-line", "tooling", "development"]
categories = ["Engineering"]
+++

Command-line tools are the workhorses of developer productivity. From `git` to `grep`, from `curl` to `docker`, we rely on CLI tools every day. Rust has emerged as an exceptional language for building these tools, combining memory safety with performance that rivals C and C++. More importantly, Rust's ecosystem provides excellent libraries that make CLI development actually enjoyable.

<!-- more -->

## Why Rust for CLI Tools

**Performance**: CLI tools often process large amounts of data—parsing logs, transforming files, searching codebases. Rust compiles to native code with zero-cost abstractions. A Rust CLI tool performs comparably to C while being significantly easier to write correctly.

**Single binary distribution**: Rust produces statically-linked binaries by default. Users get a single executable with no runtime dependencies. No "install Python 3.9," no "requires Node 16," no dependency hell. Download the binary, add it to PATH, done.

**Memory safety without garbage collection**: CLI tools shouldn't crash or leak memory. Rust's ownership system prevents memory errors at compile time. No garbage collection means predictable performance and no pause times.

**Excellent error handling**: Rust's `Result` type makes error handling explicit and composable. CLI tools need good error messages—Rust's error handling ecosystem makes this natural.

**Rich ecosystem**: Libraries like `clap` for argument parsing, `tokio` for async I/O, `serde` for serialization, and `anyhow` for error handling provide production-ready building blocks.

**Cross-platform**: Write once, compile for Linux, macOS, Windows, and more. Rust's standard library abstracts platform differences. Cross-compilation is well-supported.

## The Essential Libraries

The Rust CLI ecosystem is mature and well-designed. These libraries handle common patterns:

**`clap`**: Argument parsing with derive macros. Define your CLI structure as Rust structs, get parsing, validation, and help text for free.

**`anyhow`**: Error handling with context. Add context to errors as they bubble up. Get useful error messages without boilerplate.

**`tokio`**: Async runtime for I/O-bound tools. Concurrent file processing, network requests, or database queries become straightforward.

**`serde`**: Serialization for JSON, YAML, TOML, etc. Parse config files or API responses with minimal code.

**`indicatif`**: Progress bars and spinners. Long-running operations should show progress—`indicatif` makes this trivial.

**`colored`**: Terminal colors. Make output readable with syntax highlighting or status colors.

**`env_logger`**: Logging with environment variable configuration. Users can enable debug logging via `RUST_LOG=debug`.

## Building a Real CLI Tool

Let's build a tool that searches for TODO comments in code, aggregates them, and outputs a report. This demonstrates real-world patterns: file traversal, parsing, error handling, and formatted output.

### Project Setup

```bash
cargo new todo-finder
cd todo-finder
```

Add dependencies in `Cargo.toml`:

```toml
[dependencies]
clap = { version = "4.4", features = ["derive"] }
anyhow = "1.0"
walkdir = "2.4"
regex = "1.10"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
colored = "2.1"
```

### Argument Parsing with Clap

Define your CLI interface as Rust types:

```rust
use clap::Parser;
use std::path::PathBuf;

#[derive(Parser, Debug)]
#[command(name = "todo-finder")]
#[command(about = "Find and report TODO comments in source code")]
struct Args {
    /// Directory to search
    #[arg(default_value = ".")]
    path: PathBuf,

    /// File extensions to search (e.g., "rs,js,py")
    #[arg(short, long, value_delimiter = ',')]
    extensions: Option<Vec<String>>,

    /// Output format: text, json
    #[arg(short, long, default_value = "text")]
    format: String,

    /// Case-insensitive search
    #[arg(short, long)]
    ignore_case: bool,
}

fn main() {
    let args = Args::parse();
    println!("{:?}", args);
}
```

Run it:

```bash
$ cargo run -- --help
Find and report TODO comments in source code

Usage: todo-finder [OPTIONS] [PATH]

Arguments:
  [PATH]  Directory to search [default: .]

Options:
  -e, --extensions <EXTENSIONS>  File extensions to search (e.g., "rs,js,py")
  -f, --format <FORMAT>          Output format: text, json [default: text]
  -i, --ignore-case              Case-insensitive search
  -h, --help                     Print help
```

You didn't write the help text manually—`clap` generated it from your struct. Changes to arguments automatically update the help.

### File Traversal and Filtering

Use `walkdir` to recursively traverse directories:

```rust
use walkdir::WalkDir;

fn find_files(path: &PathBuf, extensions: &Option<Vec<String>>) -> Vec<PathBuf> {
    WalkDir::new(path)
        .into_iter()
        .filter_map(|entry| entry.ok())
        .filter(|entry| entry.file_type().is_file())
        .filter(|entry| {
            extensions.as_ref().map_or(true, |exts| {
                entry
                    .path()
                    .extension()
                    .and_then(|ext| ext.to_str())
                    .map_or(false, |ext| exts.contains(&ext.to_string()))
            })
        })
        .map(|entry| entry.path().to_path_buf())
        .collect()
}
```

This filters files by extension if specified, otherwise includes all files. Iterator combinators make the logic readable.

### Parsing TODO Comments

Define a structure for TODO items:

```rust
use serde::Serialize;

#[derive(Debug, Serialize)]
struct Todo {
    file: String,
    line: usize,
    text: String,
}
```

Search files with regex:

```rust
use regex::Regex;
use std::fs;

fn find_todos(
    files: &[PathBuf],
    ignore_case: bool,
) -> anyhow::Result<Vec<Todo>> {
    let pattern = if ignore_case {
        r"(?i)//\s*TODO:?\s*(.+)"
    } else {
        r"//\s*TODO:?\s*(.+)"
    };
    let re = Regex::new(pattern)?;
    let mut todos = Vec::new();

    for file_path in files {
        let content = fs::read_to_string(file_path)?;

        for (line_num, line) in content.lines().enumerate() {
            if let Some(captures) = re.captures(line) {
                if let Some(text) = captures.get(1) {
                    todos.push(Todo {
                        file: file_path.display().to_string(),
                        line: line_num + 1,
                        text: text.as_str().trim().to_string(),
                    });
                }
            }
        }
    }

    Ok(todos)
}
```

This uses `anyhow::Result` for error handling. The `?` operator propagates errors. Regex compilation errors, file read errors—all handled uniformly.

### Output Formatting

Support multiple output formats:

```rust
use colored::*;

fn print_todos(todos: &[Todo], format: &str) -> anyhow::Result<()> {
    match format {
        "json" => {
            println!("{}", serde_json::to_string_pretty(todos)?);
        }
        "text" => {
            println!("{}", format!("Found {} TODOs:\n", todos.len()).bold());
            for todo in todos {
                println!(
                    "{}:{} - {}",
                    todo.file.cyan(),
                    todo.line.to_string().yellow(),
                    todo.text
                );
            }
        }
        _ => anyhow::bail!("Unknown format: {}", format),
    }
    Ok(())
}
```

Text format uses colors for readability. JSON format enables piping to other tools or scripts.

### Putting It Together

```rust
fn main() -> anyhow::Result<()> {
    let args = Args::parse();

    let files = find_files(&args.path, &args.extensions);
    let todos = find_todos(&files, args.ignore_case)?;
    print_todos(&todos, &args.format)?;

    Ok(())
}
```

Run it:

```bash
$ todo-finder src -e rs
Found 3 TODOs:

src/main.rs:42 - Implement better error messages
src/parser.rs:18 - Add support for FIXME comments
src/output.rs:7 - Support CSV output format
```

With JSON output:

```bash
$ todo-finder src -e rs -f json | jq '.[] | .file'
"src/main.rs"
"src/parser.rs"
"src/output.rs"
```

The tool composes with Unix pipelines because it follows conventions: plain text to stdout, errors to stderr, JSON when requested.

## Error Handling Patterns

Rust forces you to handle errors. This makes CLI tools robust:

```rust
use anyhow::{Context, Result};

fn load_config(path: &PathBuf) -> Result<Config> {
    let content = fs::read_to_string(path)
        .with_context(|| format!("Failed to read config file: {}", path.display()))?;

    let config: Config = serde_json::from_str(&content)
        .with_context(|| format!("Failed to parse config file: {}", path.display()))?;

    Ok(config)
}
```

If reading fails, users see: "Failed to read config file: /path/to/config.json: No such file or directory"

If parsing fails: "Failed to parse config file: /path/to/config.json: missing field 'name' at line 5"

Context stacks as errors bubble up. Users get actionable error messages without you writing custom error types.

## Async I/O for Performance

Some CLI tools are I/O bound—making HTTP requests, querying databases, reading many files. Tokio enables concurrent I/O:

```rust
use tokio;
use std::fs;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let args = Args::parse();

    let files = find_files(&args.path, &args.extensions);

    let mut tasks = Vec::new();
    for file_path in files {
        tasks.push(tokio::spawn(async move {
            tokio::fs::read_to_string(&file_path).await
        }));
    }

    let contents = futures::future::join_all(tasks).await;

    // Process contents...

    Ok(())
}
```

Files are read concurrently. For large codebases, this is dramatically faster than sequential I/O.

For network-bound tools:

```rust
use reqwest;

async fn fetch_urls(urls: &[String]) -> Result<Vec<String>> {
    let client = reqwest::Client::new();

    let tasks: Vec<_> = urls
        .iter()
        .map(|url| {
            let client = client.clone();
            let url = url.clone();
            tokio::spawn(async move {
                client.get(&url).send().await?.text().await
            })
        })
        .collect();

    let results = futures::future::join_all(tasks).await;

    results
        .into_iter()
        .map(|r| r?.map_err(Into::into))
        .collect()
}
```

Multiple URLs fetched concurrently. The tool stays responsive and maximizes throughput.

## Progress Indicators

Long-running operations need progress feedback:

```rust
use indicatif::{ProgressBar, ProgressStyle};

fn process_files(files: &[PathBuf]) -> Result<()> {
    let pb = ProgressBar::new(files.len() as u64);
    pb.set_style(
        ProgressStyle::default_bar()
            .template("{spinner:.green} [{bar:40.cyan/blue}] {pos}/{len} {msg}")
            .unwrap()
            .progress_chars("#>-"),
    );

    for file_path in files {
        pb.set_message(format!("Processing {}", file_path.display()));
        process_file(file_path)?;
        pb.inc(1);
    }

    pb.finish_with_message("Done!");
    Ok(())
}
```

Users see:

```
⠁ [######################>-----------------] 42/100 Processing src/main.rs
```

This transforms the experience of waiting for a task to complete.

## Testing CLI Tools

Rust's testing story extends to CLI tools:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::fs;
    use tempfile::TempDir;

    #[test]
    fn test_find_todos() {
        let dir = TempDir::new().unwrap();
        let file_path = dir.path().join("test.rs");

        fs::write(
            &file_path,
            "// TODO: fix this\nfn main() {}\n// TODO: and this",
        )
        .unwrap();

        let files = vec![file_path];
        let todos = find_todos(&files, false).unwrap();

        assert_eq!(todos.len(), 2);
        assert_eq!(todos[0].text, "fix this");
        assert_eq!(todos[1].text, "and this");
    }
}
```

Use `tempfile` for isolated test directories. Test parsing logic, output formatting, error cases—all without hitting real filesystems.

For integration tests:

```rust
use assert_cmd::Command;

#[test]
fn test_cli_output() {
    let mut cmd = Command::cargo_bin("todo-finder").unwrap();
    cmd.arg("test-fixtures")
        .arg("-f")
        .arg("json")
        .assert()
        .success();
}
```

`assert_cmd` runs your compiled binary. Test the actual user experience, including argument parsing and output formatting.

## Distribution and Installation

Rust's tooling makes distribution straightforward.

**Cross-compilation**:

```bash
# Install target
rustup target add x86_64-unknown-linux-musl

# Build static binary
cargo build --release --target x86_64-unknown-linux-musl
```

The resulting binary runs on any Linux x86_64 system without dependencies.

**GitHub Releases**: Use `cross` for easy cross-compilation to multiple platforms:

```bash
cross build --release --target x86_64-apple-darwin
cross build --release --target x86_64-pc-windows-gnu
cross build --release --target x86_64-unknown-linux-musl
```

Upload binaries to GitHub Releases. Users download for their platform.

**Homebrew**: Create a Homebrew formula for macOS users:

```ruby
class TodoFinder < Formula
  desc "Find and report TODO comments in source code"
  homepage "https://github.com/username/todo-finder"
  url "https://github.com/username/todo-finder/releases/download/v1.0.0/todo-finder-v1.0.0.tar.gz"
  sha256 "..."

  def install
    bin.install "todo-finder"
  end
end
```

**Cargo**: Publish to crates.io. Users install with `cargo install todo-finder`.

**Docker**: For containerized environments:

```dockerfile
FROM rust:1.75 as builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/todo-finder /usr/local/bin/
ENTRYPOINT ["todo-finder"]
```

## Real-World Examples

**ripgrep**: Fast text search tool. Outperforms `grep` while adding features. Demonstrates Rust's performance potential.

**bat**: `cat` clone with syntax highlighting and Git integration. Shows how Rust tools can enhance Unix utilities.

**fd**: Modern `find` replacement. Simpler syntax, faster execution, respects `.gitignore`. User-friendly defaults.

**exa**: Improved `ls` with colors, Git awareness, tree view. Makes common tasks more pleasant.

**tokei**: Code statistics tool. Counts lines of code across languages. Fast enough for massive codebases.

All written in Rust. All distributed as single binaries. All dramatically faster than their alternatives while being safer and more maintainable.

## Why This Matters

CLI tools are infrastructure. Developers use them hundreds of times daily. Slow tools waste time. Buggy tools break workflows. Poorly designed tools frustrate users.

Rust lets you build tools that are:
- **Fast**: Native performance without garbage collection pauses
- **Reliable**: Memory safety prevents crashes and undefined behavior
- **Maintainable**: Strong types and good error handling make changes safe
- **Distributable**: Single binaries with no runtime dependencies
- **Cross-platform**: Write once, compile for all major platforms

The ecosystem provides production-quality libraries. You're not building from scratch—you're composing well-tested components.

Modern development involves managing complexity across distributed systems, multiple languages, and vast codebases. CLI tools automate repetitive tasks, enforce standards, and integrate disparate systems. Building them in Rust means they'll be fast, correct, and pleasant to use.

You don't need to rewrite `grep` to benefit from Rust. Small tools that automate your specific workflows—build checkers, log parsers, deployment scripts, code generators—all become better in Rust. Start with a simple tool. Replace a slow Python script. Build something you'll use daily.

The best tools disappear into your workflow. They're fast enough you don't think about performance. Reliable enough you don't think about crashes. Well-designed enough you don't think about options. Rust gives you the foundation to build tools like that.
