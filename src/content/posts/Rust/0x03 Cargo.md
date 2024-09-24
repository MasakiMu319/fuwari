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





