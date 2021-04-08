# Report

2018011365 张鹤潇

### 编程作业

在模板代码的基础上新增 `sys_spawn` 系统调用，我将其实现为 `fork` 和 `exec` 的组合。这种实现方式在效率上虽然有欠缺，但已足以通过测例。

> 实际上，根据 `fork` 和 `exec` 的实现，编写不复制父进程内存空间的 `spawn` 也并不复杂。

为了保证前向兼容，我将前四个 lab 完成的功能移植到内核中。

### 问答题

#### 1

可以使用 Copy on Write 技术：`fork` 生成子进程时，子进程的代码段、数据段、堆栈都指向父进程的物理空间，只在相应内存页被子进程或父进程修改时才拷贝生成新页面。

#### 2

> 论文的一个中文版本：https://www.infoq.cn/article/bygiwi-fxhtnvsoheunw

- 即使实现了 COW，与 `spawn` 相比，`fork+exec` 仍然带来较大的时间成本。
- `fork` 最初是为单线程系统而设计的，它不是线程安全的。
- 因为现代操作系统的复杂性，`fork` 已经不再简洁了，许多系统调用 flag 会影响 `fork` 的行为。

#### 3

假设 `spawn` 的接口为 `fn spawn(path: &str, args: &[&str]) -> isize`

重写后的伪代码如下：

```rust
// a.rs
fn main(argc: usize, argv: &[&str]) {
    let a = get_a();
    spawn("b.rs", &[&a.to_string()]);
    println!("a = {}", a);
    exit(0);
}

// b.rs
fn main(argc: usize, argv: &[&str]) {
    assert!(argc == 1);
    let a = argv[0].parse().unwrap();
    let b = get_b();
    println!("a + b = {}", a + b);
    exit(0);
}
```

子程序 `b.rs` 的功能很简单，没有必要独立出来。`fork` 的存在提高了多进程编程的自由度，有利于编写更紧凑的程序。

> 上述例子略作修改后可在 Ubuntu 上直接运行：
>
> ```rust
> // a.rs
> use std::process::Command;
> 
> fn main() {
>     let a = 10;
>     Command::new("./b")
>         .arg(&a.to_string())
>         .spawn()
>         .expect("failed to spawn");
>     println!("a = {}", a);
> }
> 
> // b.rs
> use std::env;
> 
> fn main() {
>     let args: Vec<String> = env::args().collect();
>     assert!(args.len() == 2);
>     let a = args[1].parse::<i32>().unwrap();
>     let b = 20;
>     println!("a + b = {}", a + b);
> }
> 
> ```
>
> 运行方式：
>
> ```sh
> rustc b.rs -o b
> rustc a.rs -o a
> ./a
> ```
#### 4

进程的状态大致分为：就绪、等待 、运行、退出 (包括 Zombie 态)

- `fork` 创建了一个就绪进程
- `exec` 由运行中的程序发起，此后程序仍处于运行态
- `wait` 使程序从运行态转换到等待状态
- `exit` 使运行中的程序退出。