---
title: 0x03 - Cargo
published: 2024-09-19 12:00:00
description: '深入探索 Cargo 使用'
image: ''
tags: ['Rust', 'Basic skills']
category: 'Rust'
draft: true
---

>   在 Rust 中，Crate 被翻译成包，因此 package 可以认为是工程、软件包。

rust 有两种类型的包：

-   库包；
-   二进制包；

cargo new 创建的即是一个新的项目，可执行的二进制包，--bin 是默认参数，可以省略。

cargo run 会自动帮我们在 debug 模式下进行编译，然后运行。在使用 --release 参数的情况下，程序会做大量的性能优化。

mod 关键字：用于声明一个模块，告诉编译器去包含指定的模块文件。

use 关键字：用于将某个模块中的类型或函数引入当前作用域。

**mod 与创建 lib.rs 的区别**

-   **使用** mod**：** 当你在 main.rs 中使用 mod 声明模块时，这些模块仅在当前二进制包（crate）内可见。main.rs 是整个模块树的根，你需要在其中声明所有的顶级模块。
-   **创建** lib.rs**：** 当你创建 lib.rs 文件时，你实际上在定义一个库包。模块的层次结构从 lib.rs 开始，这使得你的代码可以作为库被其他包使用。

**为什么创建** lib.rs **可以避免在** main.rs **中写很多** mod**？**

当你在 lib.rs 中声明模块后，main.rs 可以直接使用这些模块，而不需要在 main.rs 中再次声明 mod。这样可以使 main.rs 更加简洁，只关注程序的入口逻辑。

## 基础

-   cargo new {project} --bin --vcs none：创建一个 binary 或者 lib 类型的项目，并不会自动进行 git 初始化；
-   cargo build：默认采用 --debug 模式进行编译目标项目；
-   cargo run：默认在采用 --debug 模式编译后并自动运行；
-   cargo build --release：使用 --release 模式编译目标项目，rustc 会对代码自动进行性能优化。

当你从 remote 拉取一个新项目后，执行 cargo build 即可自动拉取相应的依赖并构建。

cargo 通过 Cargo.toml 配置文件管理依赖及其相关的版本（可以类比 Python 中的 pyproject.toml ），同时在编译的时候会自动生成一个 Cargo.lock 文件（可以类比使用 uv 作为Python 包管理工具产生的 uv.lock），其中包含了项目所使用依赖的更详细信息。

如果需要更新依赖的版本，需执行 cargo update，否则会使用 Cargo.toml 中声明的版本。

### Cargo项目目录惯例

```
.
├── Cargo.lock
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── main.rs
│   └── bin/
│       ├── named-executable.rs
│       ├── another-executable.rs
│       └── multi-file-executable/
│           ├── main.rs
│           └── some_module.rs
├── benches/
│   ├── large-input.rs
│   └── multi-file-bench/
│       ├── main.rs
│       └── bench_module.rs
├── examples/
│   ├── simple.rs
│   └── multi-file-example/
│       ├── main.rs
│       └── ex_module.rs
└── tests/
    ├── some-integration-tests.rs
    └── multi-file-test/
        ├── main.rs
        └── test_module.rs
```

-   `Cargo.toml`, `Cargo.lock` 存储在包的根目录;
-   项目源文件在 `src` 目录中;
-   默认的库文件是 `src/lib.rs`;
-   默认的可执行文件是 `src/main.rs`;
    -   其他可执行的文件在 `src/bin/`;
-   Benchmarks 在 benches 目录中；
-   示例在 examples 目录中；
-   集成测试在 tests 目录中；

如果 binary，example，bench 或者集成测试包含了多个源文件，可以放在相应子目录下并添加 main.rs 文件，可执行文件的名称与文件夹名称保持一致。

