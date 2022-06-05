随着Rust的兴起，许多开发人员越来越有兴趣在Linux内核中尝试Rust。

2019 年，Alex Gaynor 和 Geoffrey Thomas 在 Linux Security Summit 安全峰会上进行了演讲，他们介绍了 Rust 内核模块的一个原型，并提出了在内核中采用 Rust 的理由。此次演讲重点是在安全问题上，其中指出在 Android 和 Ubuntu 中，约有三分之二的内核漏洞被分配到 CVE 中，这些漏洞都是来自于内存安全问题。原则上，Rust 可以通过其 type system 和 borrow checker 所提供的更安全的 API 来完全避免这类错误。

在 2020 Linux Plumbers Conference 上，Thomas 、Gaynor、Rust 语言团队的联合领导者 Josh Triplett 以及其他一些对此感兴趣的开发者以“Barriers to in-tree Rust”为主题，讨论了想要把 Rust 引入到 Linux 内核项目中作为一种可选的开发语言还需要解决的一些问题。其中 in-tree 是 Linux 术语，意思是与内核源代码树本身一起存储并与之一起构建内核模块。

对此，Linux 之父 Linus Torvalds 也曾发表看法：Linux 最终不会用 Rust 编写，没有人会用 Rust 重写内核的 2500 万行 C，但是他也看到了 Rust 的优势，鼓励采用缓慢但稳定的方法将 Rust 引入 Linux，同时他表示将 Rust 接口用于驱动程序和其他非核心内核程序是有道理的。

Rust for Linux 这个项目的目的就是为了将 Rust 引入 Linux，让 Rust 成为 C 语言之后的第二语言。但它最初的目的是：实验性地支持Rust来写内核驱动。

以往，Linux 内核驱动的编写相对于应用其实是比较复杂的，具体复杂性主要表现在以下两个方面：

* 编写设备驱动必须了解Linux内核基础概念、工作机制、硬件原理等知识。
* 设备驱动中涉及内存和多线程并发时容易出现Bug，Linux驱动跟内核驱动工作在同一层次，一旦出现问题，很容易造成内核的整体崩溃

引入 Rust 从理想层面来看，一方面在代码抽象和跨平台方面比 C 更有效，另一方面代码质量会更高，有效减少内存和多线程并发类 Bug。

Rust for Linux 就是为了帮助实现这一目标，为 Linux 提供了 Rust 相关的基础设施和方便编写 Linux 驱动的安全抽象。