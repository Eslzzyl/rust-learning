本章将讨论 Cargo 的一些更为高级的功能：

- 使用发布配置来自定义构建
- 将库发布到 [crates.io](https://crates.io/)
- 使用工作空间来组织更大的项目
- 从 [crates.io](https://crates.io/) 安装二进制文件
- 使用自定义的命令来扩展 Cargo

有关 Cargo 的文档，请查阅 http://doc.rust-lang.org/cargo/

## 14.1 采用发布配置自定义构建

**发布配置**（*release profiles*）是一个术语，指的是那些预定义的、可定制的、带有不同选项的配置。

Cargo 有两个主要的配置：

- 运行 `cargo build` 时采用的 `dev` 配置
- 运行 `cargo build --release` 的 `release` 配置。

`dev` 配置是适合开发的默认配置，`release` 配置则适合发布。

这两个配置在执行`cargo build`时会出现在输出中。

当项目的 *Cargo.toml* 文件中没有任何 `[profile.*]` 部分的时候，Cargo 会对每一个配置都采用默认设置。通过增加任何希望定制的配置对应的 `[profile.*]` 部分，我们可以选择覆盖任意默认设置的子集。

下面是 `dev` 和 `release` 配置的 `opt-level` 设置的默认值：

```toml
[profile.dev]
opt-level = 0

[profile.release]
opt-level = 3
```

`opt-level` 设置控制优化等级。这个配置的值从 0 到 3。越高的优化级别需要越多的时间编译。

如果我们想要修改上面的配置，就要手动在 *Cargo.toml* 文件中增加配置行。注意上面的配置是默认的，所以不会出现在文件中。

对于每个配置的设置和其默认值的完整列表，查看 [Cargo 的文档](https://doc.rust-lang.org/cargo/reference/profiles.html)。

## 14.2 将项目发布到 Crates.io

crates.io 用来分发包的源代码，所以它主要托管开源代码。

### 编写有用的文档注释

文档注释可被用于生成HTML文档。他们意在让对库感兴趣的程序员理解如何 **使用** 这个 crate，而不是它是如何被 **实现** 的。

文档注释使用三斜杠 `///`，支持 Markdown。文档注释就位于被注释项的前面。

```rust
/// Adds one to the number given.
///
/// # Examples
///
/// ```
/// let arg = 5;
/// let answer = my_crate::add_one(arg);
///
/// assert_eq!(6, answer);
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

要生成HTML文档，执行`cargo doc`命令。这将调用`rustdoc`并将生成的文档放入`target/doc`目录。

#### 常用文档注释部分

- **Examples**：这个函数的用法示例。

- **Panics**：这个函数可能会 `panic!` 的场景。
- **Errors**：如果这个函数返回 `Result`，此部分描述可能会出现何种错误以及什么情况会造成这些错误，这有助于调用者编写代码来采用不同的方式处理不同的错误。
- **Safety**：如果这个函数使用 `unsafe` 代码（见十九章），这一部分应该会涉及到期望函数调用者支持的确保 `unsafe` 块中代码正常工作的不变条件（invariants）。

#### 文档注释作为测试

`cargo test` 会像测试那样运行文档中的示例代码。

#### 注释包含项的结构

还有一种风格的文档注释：`//!`，对于描述 crate 和模块特别有用。使用他们描述其容器整体的目的来帮助 crate 用户理解你的代码组织。

（解释有点复杂，没看懂，用到时再查）

### 使用 `pub use` 导出合适的公有 API

本节尚未完成。2020-05-15

### 创建 Crates.io 账号

发布 crate 之前，需要在 crates.io 上注册账号并获取一个 API token。

一旦获取 API token，就可以用`cargo login`登录：

```bash
cargo login xxxxxxxxxx
```

这个命令会通知 Cargo 你的 API token 并将其储存在本地的 *~/.cargo/credentials* 文件中。注意这个 token 不应该与其他人共享。如果因为任何原因与他人共享了这个信息，应该立即到 crates.io 重新生成 token。

### 发布新 crate 之前

假设你已经有了一个待发布的 crate。

在发布之前，需要在 crate 的 *Cargo.toml* 文件的`[package]`部分增加一些元数据。

首先 crate 需要一个唯一的名称。虽然在本地开发 crate 时，可以使用任何你喜欢的名称，不过 crates.io 上的 crate 名称遵守先到先得的分配原则。一旦某个 crate 名称被使用，其他人就不能再发布这个名称的 crate 了。

其次要为 crate 添加一些描述，通常是一两句话因为它会出现在 crate 的搜索结果中和 crate 页面里。

还需要一条 license 信息，可以直接使用[这里](https://spdx.org/licenses/)给出的标识符，此种情况下使用`license`字段直接指定；也可以使用自己的 license，此种情况下需要把自己的 license 放进一个文件，并将文件加入项目，接着使用`licesne-file`字段来指定文件名。

Rust 自身的 license 是 `MIT OR Apache-2.0`。所以许多人选择同一个 license 作为他们自己 crate 的 license。

![开源许可说明图](http://www.ruanyifeng.com/blogimg/asset/201105/free_software_licenses.png)

于是，已经准备好发布的项目的 *Cargo.toml* 文件可能看起来像这样：

```toml
[package]
name = "guessing_game"
version = "0.1.0"
edition = "2021"
description = "A fun game where you guess what number the computer has chosen."
license = "MIT OR Apache-2.0"

[dependencies]
```

### 发布到 Crates.io

注意！crate 的发布是永久性的，已经发布的版本无法覆盖，其代码也无法删除。crates.io 的一个主要目标是作为一个存储代码的永久文档服务器，这样所有依赖其中 crate 的项目都能一直正常工作。

如果你发现你发布的 crate 中包含某些秘密信息，比如密码，你能做的就是立即修改这些密码。指望把密码从 crates.io 抹去是不可能的。

要发布代码，运行`cargo publish`命令。

### 发布现存 crate 的新版本

当你要发布新版本时，需要修改 *Cargo.toml* 中 `version` 所指定的值。

修改版本时，版本号的原则请参阅 [语义化版本 2.0.0 | Semantic Versioning (semver.org)](https://semver.org/lang/zh-CN/)

修改好后，运行 `cargo publish` 即可。

### 使用 `cargo yank` 撤回版本

虽然你永远无法删除已经发布版本的代码，但你仍然可以阻止其他用户使用某个版本的代码。这个术语叫做 **撤回**（*yanking*）。

撤回某个版本会阻止新项目依赖此版本，但所有现存的依赖此版本的项目仍然能够下载和依赖这个版本。

要撤回一个版本，使用 `cargo yank` 并同时指定版本：

```bash
cargo yank --vers 1.0.1
```

如果想返回，可以撤销撤销：

```bash
cargo yank --vers 1.0.1 --undo
```

## 14.3 Cargo 工作空间

## 14.4 使用 `cargo install` 从 Crates.io 安装二进制文件

## 14.5 Cargo 自定义扩展命令

