---
layout: post
title: "Rust 语言入门"
# subtitle: "starrocks topn 排序算法解析"
author: "KodiakAS"
header-img: "img/database-bg.png"
header-mask: 0.3
tags:
  - rust
---

# 1.安装及基本命令


## 1.1 安装

[Getting started](https://www.rust-lang.org/learn/get-started)

WSL：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Linux or Mac:

```bash
curl https://sh.rustup.rs -sSf | sh
```

## 1.2 基本命令

更新rust版本 `rustup update`

卸载rust `rustup self uninstall`

打开本地文档 `rustup doc`

**Cargo -- Rust 的包管理工具**

新建项目 `cargo new project_name`

编译项目 `cargo build [--release]`

编译项目并运行 `cargo run`

仅检查但不产生二进制文件 `cargo check`

# 2.基础知识


## 2.1 Crate

Crate是Rust中的库，Rust标准库中的prelude模块会默认导入，其它需要显式导入

使用外部Crate时需要在Rust项目的配置文件`Cargo.toml` 中配置依赖项

```rust
[dependencies]
rand = "0.8.3"
```

此版本号会在第一次执行`cargo build`时被写入`Cargo.lock` ，编译时根据`Cargo.lock` 中的版本号在线拉取依赖库，每次编译时均会使用此版本的库

使用`cargo update` 命令会将`Cargo.lock` 中的各依赖项版本号更新（只会更新版本号的第三位，防止语义变化），但不会修改`Cargo.toml` 中的内容

## 2.2 基本语法示例

```rust
use std::cmp::Ordering;
use std::io;
use rand::{thread_rng, Rng}; // 外部依赖需在Cargo.toml中配置

fn main() {
    let secret_number = thread_rng().gen_range(1..101);
    println!("Take a guess"); // 使用宏 Println!

    loop {
        let mut guess = String::new(); // 创建空字符串实例
        io::stdin()
            .read_line(&mut guess) // 返回枚举类型，Ok或Err
            .expect("Can not read line."); // 处理异常，接收Err时退出并打印信息

        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num, // match的每个分支称为一个arm
            Err(_) => {
                println!("Please type a number!");
                continue;
            }
        };

        println!("Your guess is {}", guess);
        match guess.cmp(&secret_number) {
            Ordering::Less => println!("To small"),
            Ordering::Greater => println!("To big"),
            Ordering::Equal => {
                println!("You win");
                break;
            }
        }
    }
}
```

# 3.概念


## 3.1 变量与可变性

Rust中变量默认是不可变的，声明变量时加上`mut`关键字声明其为可变

### 常量

- 常量使用`const` 关键字声明，必须标注数据类型，一旦声明永远不可变
- 常量可以在任何作用域内声明，包括全局作用域
- 常量只可以绑定到常量表达式，无法绑定到函数调用结果或只能在运行时计算出的值
- 程序运行期间，常量在其声明的作用域内一直有效

```rust
const MAX_THREADS: u32 = 100;
```

### Shadowing

可以用相同的名字声明新的变量，新变量会隐藏掉之前的同名变量，声明同名变量可以使用不同类型

```rust
let x = 5;
let x = x + 1;
let x = "    "
```

## 3.2 标量数据类型

Rust是一种编译型静态语言，编译时必须知道所有变量的类型，基于使用的值，编译器通常可以推断出它们数据类型，但如果可能的类型比较多（如将String转换为整数的parse方法），就必须添加类型标注，否则编译时会报错

```rust
let guess: u32 = "100".parse().expect("Not a number")
```

### 整数类型

![int](/img/in-post/rust/int.png)

### 数字字面值

![number_literal](/img/in-post/rust/number_literals.png)

> 整数溢出

u8的范围是0-255，如果把一个u8的值设置为256

- debug模式下编译，Rust会检查整数溢出，程序运行时会panic
- release模式下编译，不会检查可能导致panic的整数溢出，如果运行时发生溢出，会执行环绕操作，即256变成0，257变成1，程序不会panic

### 浮点类型

Rust有两种基础浮点类型，`f32`和`f64` ，默认使用`f64`

### 布尔类型

`true`和`false`，占用一个字节

### 字符类型

Rust中的char类型用来描述语言中最基础的单个字符，字面值使用单引号，占用4字节

char类型是Unicode标量值，可以表示中日韩文，空白字符，emoji等内容

## 3.3 复合类型

### Tuple

可以将多个不同类型的值放在一个类型里，长度是固定的，声明后无法修改

```rust
let tup: (i32, f64, u8) = (500, 0.5, 1);
```

#### 获取tuple元素值

```rust
let x, y, z = tup;
```

#### 访问tuple元素

```rust
println!("{}, {}, {}", tup.0, tup.1, tup.2);
```

### 数组

可以将多个同类型的元素放在一个类型里，长度固定

```rust
let a = [1, 2, 3, 4, 5];
```

#### 数组的用途

将数据存放在栈上而非堆上，或者想保证有固定数量的元素

一般使用Vector（标准库提供，长度可变）

#### 数组的类型

```rust
let a: [i32, 5] = [1, 2, 3, 4, 5];
```

#### 另一种声明数组的方法

```rust
let a = [3; 5];
// let a = [3, 3, 3, 3, 3];
```

#### 访问数组元素

```rust
let first = a[0];
```

若索引超出数组范围，编译时会通过，运行时会panic，但Rust不允许访问相应地址的内存

## 3.4 函数

### 函数的声明

```rust
fn main() {
    demo_function(5); // argument
}

fn demo_function(x: i32) { // parameter
    println!("{}", x);
}
```

声明函数的时候函数参数的类型必须指明

### 函数体中的语句和表达式

函数体由一系列语句组成，可选的由一个表达式结束

Rust是一个基于表达式的语言，语句是执行一些动作的指令，表达式会计算产生一个值

```rust
fn main() {
    let y = 6;
}
```

函数的定义，变量的声明等都是语句，语句不返回值，所以不能使用`let`将一个语句赋给一个变量

示例中的"6"是一个表达式，表达式都对应一个值，可以是语句的一部分，调用函数，调用宏等都是表达式

```rust
fn main() {
        let x = 5;
        let y = {
                let x = 1;
                x + 3 // 不加";"时是一个表达式，加上则变成一个语句，返回值是一个空的tuple
        };
        println!("y = {}", y);
}
```

### 函数的返回值

返回值就是函数体里最后一个表达式的值，若要提前返回，需要使用`return`关键字，并指定一个值

```rust
fn plus_five(x: i32) -> i32 {
        x + 5 // 表达式
}

fn main() {
        let x = plus_five(1);
        println!("x = {}", x);
}
```

## 3.5 控制流 if else

根据条件执行不同分支(arm)，条件必须是bool型

```rust
fn main() {
        let number = 6;

        if number % 4 == 0 {
                println!("number is divisible by 4");
        } else if number % 3 == 0 {
                println!("number is divisible by 3");
        }	else if number % 2 == 0 {
                println!("number is divisible by 2");
        } else {
                println!("number is not divisible by 4, 3, 2");
        }
}
```

如果使用了多于一个`else if`，最好使用`match`重构代码

### 在 let 语句中使用 if

因为if是一个表达式，所以可以放在let语句中等号的右边

```rust
fn main() {
        let condition = true;
        let number = if condition {5} else {6}; // if和else块中返回值类型必须相同
        println!("number is {}", number);
}
```

## 3.6 控制流 循环

### loop

反复执行一段代码，直到被中断

```rust
fn main() {
        let mut counter = 0;

        let result = loop {
                counter += 1;
                if counter == 10 {
                        break counter * 2;
                }
        };

        println!("the result is {}", result);
}
```

### while

每次循环前都判断一次条件

```rust
fn main() {
        let mut number = 3;
        
        while number != 0 {
                println!("{}", number);
                number = number - 1;
        }

        println!("done");
}
```

### for 循环遍历集合

```rust
fn main() {
        let a = [10, 20, 30, 40, 50];
        for element in a.iter() {
                println!("{}", element);
        }
}
```

### Range

指定一个开始数字和一个结束数字，Range可以生成它的之间的数字（左闭右开），`rev`方法可以反转Range

```rust
fn main() {
        for number in (1..4).rev() {
                println!("{}", number);
        }
        println!("done");
}
```

## 4.1 所有权

所有权

# WIP