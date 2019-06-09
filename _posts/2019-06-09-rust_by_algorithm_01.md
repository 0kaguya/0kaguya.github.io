---
title: 「Rust × 算法」： 01 - 几个排序算法的实验
date: 2019-6-9
permalink: /algorithm/basic/rust_by_algorithm_01/
tags:
    - algorithm
    - rust
---

Rust 非常适合学习各种算法。比起 Python 之类的语言，Rust 属于系统编程语言，所以更愿意暴露复杂度和切实的细节，方便深入学习；比起C、C++，Rust又具有更人性化的基础设施，可以轻松地编写、组织和管理代码。当然，Java、Kotlin 和 Scala 这些 JVM 上的通用编程语言应该也很合适；不过我并不太了解。这一步的比较就纯粹是个人喜好这一维度上的考量了。

这一系列的笔记介绍我用 Rust 进行算法学习的实时情况。没有具体的算法内容教学；如果有「XX是什么」之类的基础疑问，阅读 Wiki 上的详细说明和对应的伪代码应该有所帮助。同样，这也不是 Rust 的入门教程。不了解这门语言的话，非常建议读一读官方的入门材料，难度不大，可以在安装好 Rust toolchain 之后通过`rustup doc --book`打开。

第一篇介绍对一些排序算法的实现。可以假设这是一个空的 Cargo 项目，代码位于`src/lib.rs`文件内。

## 搭起框架

普通地来说，常见的排序算法应用于可以乱序读写的顺序数据结构；同时，结构内的各个元素之间要有可以互相比较的方法。把这些思考用一个 trait 具体地表示的话就是：

``` rust
trait Sort
    where Self: Index<usize> + IndexMut<usize>,
          Self::Output: PartialOrd,  {
}
```

`Index`和`IndexMut`描述取下标的操作，两者分别是读和写。Rust 的各种运算符在标准库的`std::ops`下用不同的 trait 描述，实现了这些 trait 就可以使用对应的操作。`Self::Output`定义在`Index`中，是取下标操作的返回值的类型，即数据结构中所存储的元素的类型。它是`PartialOrd`的，也就是可以出现在不等式的两侧、进行大小比较。`PartialOrd`也描述运算符，但不在`std::ops`下，而是在`std::cmp`中。

先为标准库的`Vec`实现排序 trait 。当然，动态数组是满足上面所述的条件的。

``` rust
impl<T> Sort for Vec<T>
    where T: PartialOrd, {
}
```

这个 trait 里什么都没有。不管如何，先写个简单的插入排序进去吧。简单地把`self`看作普通的动态数组`Vec`，代码如下。

``` rust
trait Sort where ... {
    fn insertion_sort(&mut self) {
        for i in 1 .. self.len() {
            if self[i] < self[i-1] {
                let mut j = i;
                let src = self[i];
                while j > 0 && src < self[j-1] {
                    self[j] = self[j-1];
                    j = j - 1;
                }
                self[j] = src;
            }
        }
    }
}
```

写完后编译；然而编译器报错了。摘录如下：

``` text
...
error[E0599]: no method named `len` found for type `&mut Self` in the current scope
 --> src/lib.rs:
  |
  |         for i in 1_usize .. self.len() - 1 {
  |                                  ^^^
  |
...
error[E0277]: the size for values of type `<Self as std::ops::Index<usize>>::Output` cannot be known at compilation time
  --> src/lib.rs:
   |
   |                 let src = self[i];
   |                     ^^^   ------- help: consider borrowing here: `&self[i]`
   |                     |
   |                     doesn't have a size known at compile-time
   |
   = help: the trait `std::marker::Sized` is not implemented for `<Self as std::ops::Index<usize>>::Output`

...
```

编译器抱怨 `self`没有方法`len`，以及不能对元素进行简单的数据操作因为大小不可知。后一则报错的提示很明确；根据提示，为`Self::Output`加上`Sized`的 trait 进行约束：

``` rust
trait Sort
    where Self: Index<usize> + IndexMut<usize>,
          Self::Output: PartialOrd + Sized, {
    fn insertion_sort(...) { ... }
}
```

重新编译，第二则相关的报错就消失了。但缺少方法`self::len`的错误还存在着。

我们知道，动态数组`Vec`确实有方法`len`。但这个方法不是由某个 trait 提供，而是单独实现的，所以不能通过加约束解决问题。因此，只能在实例化 trait 的时候自行实现一个提供同样功能的函数了。为了不和数据结构原本的`len`方法冲突，这里改名为`size`。

``` rust
trait Sort
    where Self: Index<usize> + IndexMut<usize>,
          Self::Output: PartialOrd + Sized, {
    fn size(&self) -> usize;

    fn insertion_sort(...) { ... } // 改 .len() 为 .size()
}

impl<T> Sort for Vec<T>
    where T: PartialOrd + Sized, {
    fn size(&self) -> usize {
        self.len()
    }
}
```

解决了所有的问题吗？ 并没有，编译器接着抛出了新的错误：

``` text
error[E0507]: cannot move out of borrowed content
  --> src/lib.rs:
   |
   |                 let src = self[i];
   |                           ^^^^^^^
   |                           |
   |                           cannot move out of borrowed content
   |                           help: consider borrowing here: `&self[i]`

error[E0507]: cannot move out of borrowed content
```

不同于其他语言，Rust 的`=`在赋值时默认使用移动语义，只有实现了`Copy` trait 的类型会在移动时复制，从而看起来和别的语言相似。这里`=`语句表达的意义是硬生生地把元素从结构里剜出来，当然不可接受。要解决这一问题，可以为`Self::Output`类型添加`Copy`的 trait 约束，也可以添加`Clone`的 trait 约束，从而可以调用`clone`显示地表达复制的语义。这里采取后一种，因为`Copy` trait 受到非常多的限制。

在添加`Clone`的约束后，之前`Sized`的类型约束可以可以删去；因为`Clone`是更精确的表达方式，它覆盖了`Sized`。现在整个文件看起来是这样子的：

``` rust
//! src/lib.rs
use std::ops::{Index, IndexMut};
use std::cmp::PartialOrd;

trait Sort
    where Self: Index<usize> + IndexMut<usize>,
          Self::Output: PartialOrd + Clone, {
    fn size(&self) -> usize;

    fn insertion_sort(&mut self) {
        for i in 1 .. self.size() {
            if self[i] < self[i-1] {
                let mut j = i;
                let src = self[i].clone();
                while j > 0 && src < self[j-1] {
                    self[j] = self[j-1].clone();
                    j = j - 1;
                }
                self[j] = src;
            }
        }
    }
}

impl<T> Sort for Vec<T>
    where T: PartialOrd + Clone, {
    fn size(&self) -> usize {
        self.len()
    }
}
```

### *使用写好的代码：简单的命令行交互

娱乐性的小节。快速浏览请跳至下一部分。

要显式地快速地看到写好的代码的执行效果的话，可以写一个简单的命令行交互程序。把包括入口函数`main`的代码文件放在`src/bin`文件夹下就可以通过`cargo run`命令编译执行了。代码如下：

``` rust
//! src/bin/sort.rs
use sort::Sort; // `sort`是项目名
use std::io::{self, Write};

fn main() {
    print!("Enter some number: ");
    io::stdout().flush()
        .unwrap();

    let mut buf = String::new();
    io::stdin()
        .read_line(&mut buf)
        .unwrap();

    let mut vec = buf
        .split_whitespace()
        .filter_map(|s| s.parse().ok())
        .collect::<Vec<i32>>();
    vec.insertion_sort();

    println!("Sorted numbers: {}", vec.iter()
             .map(i32::to_string)
             .collect::<Vec<_>>()
             .join(" "));
}
```

### 单元测试

写了代码之后该如何验证？简单的方法是做一些测试。比如说，这里添加一项简单的单元测试：

``` rust
#[cfg(test)]
mod test {
    use super::*;

    #[test]
    fn simple_test_insertion_sort_1() {
        let mut v = vec![3,2,4,1,5];
        v.insertion_sort();
        assert_eq!(v, vec![1,2,3,4,5]);
    }

    ...
}
```

可以手动构造更多更复杂的测试，不过更简洁的方法是随机生成数据。生成随机数需要添加外部依赖`rand`，也就是在`Cargo.toml`中`[dependencies]`下添加一行：

``` toml
[dependencies]
rand = "0.6"
```

然后写一个生成随机动态数组的函数：

``` rust
use rand::{thread_rng, Rng};
use rand::distributions::{Distribution, Standard};

#[allow(dead_code)]
fn random_vec<T>(size: usize) -> Vec<T>
    where Standard: Distribution<T>, {
    thread_rng().sample_iter(&Standard).take(size).collect()
}
```

相应的需要写一个判断是否排好序的函数。

``` rust
trait Sort where ... {
    ...
    fn is_sorted(&self) -> bool {
        for i in 1_usize .. self.size() {
           if self[i-1] > self[i] {
               return false
           }
        }
        return true
    }
}
```

用这两个函数就可以写随机测试了。

``` rust
...
#[cfg(test)]
mod test {
    ...
    #[test]
    fn random_test_sample() {
        let mut v = random_vec::<f64>(10000);
        v.insertion_sort();
        assert!(v.is_sorted());
    }
}
```

``` text
$ cargo test
...
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

$
```

### 性能测试

除了算法的正确性，算法的性能指标也很重要。性能测试的方法类似单元测试。

``` rust
#![feature(test)]

#[cfg(test)]
mod bench {
    extern crate test;
    use test::Bencher;

    use super::*;

    #[bench]
    fn insertion_sort(b: &mut Bencher) {
        b.iter(|| {
            random_vec::<i32>(1000).insertion_sort()
        })
    }
}
```

闭包内的代码会被多次执行以保证测试结果准确稳定。注意由于多次执行的缘故，太大的数据会变得很慢。对于这个简单实现的插入排序来说，数组的规模超过 10000 就开始吃力了。

``` text
$ cargo bench
...
test bench::insertion_sort   ... bench:     281,999 ns/iter (+/- 5,439)
$
```

## 更多的排序算法

有了上述的基础设施，去实现和测试各种不同的排序算法就很轻松了。

### 冒泡排序

冒泡排序和插入排序有同样的时间复杂度，但使用了更多的读写，所以可能比插入排序慢。

``` rust
trait Sort where ... {
    ...
    fn bubble_sort(&mut self) {
        for i in (0 .. self.size()).rev() {
            for j in 1 .. i + 1 {
                if self[j-1] > self[j] {
                    swap(&self[j-1], self[j])
                }
            }
        }
    }
}
```

其中`swap!`是简单的三变量交换，用简单的宏实现。可以改成在`unsafe`代码块中使用`std::mem::swap`实现同样的功能，但效率没有区别，没有必要。

``` rust
macro_rules! swap { 
    ($x:expr, $y:expr) => {
        {
            let tmp = $x.clone();
            $x = $y.clone();
            $y = tmp;
        }
    }
}
```

性能测试，数据是`random_vec::<i32>(1000)`，插入排序和冒泡排序的对比是这样的：

``` text
test bench::insertion_sort   ... bench:     281,812 ns/iter (+/- 3,123)
test bench::bubble_sort      ... bench:   1,128,302 ns/iter (+/- 44,081)
```

冒泡排序很恐怖。

### 希尔排序

希尔排序很有趣。对于不同的间隔序列，希尔排序有不同的效率，这样可以写更多测试来对比了。

首先实现一个希尔排序。和别的排序API不同，调用希尔排序需要传入一个间隔序列。要追求统一的话可以封装一层传入高效间隔序列的实现。

``` rust
fn shell_sort_with(&mut self, gaps: &[usize]) {
    for &gap in gaps {
        for b in 0 .. gap {
            for i in 1 .. self.size() / gap {
                if self[i * gap + b] < self[(i - 1) * gap + b] {
                    let mut j = i;
                    let src = self[i * gap + b].clone();
                    while j > 0 && src < self[(j - 1) * gap + b] {
                        self[j * gap + b] = self[(j - 1) * gap + b].clone();
                        j -= 1;
                    }
                    self[j * gap + b] = src;
                }
            }
        }
    }
}
```

为了便于调节和对比，使用统一的函数生成样例。

``` rust
fn demo() -> Vec<i32> {
    random_vec::<i32>(10000)
}
```

分别选择`1,`、`5,3,1`、`109,41,19,5,1`作为间隔序列，得到的结果如下：

``` text
test bench::insertion_sort_1 ... bench:  27,734,230 ns/iter (+/- 4,995,983)
test bench::shell_sort_1     ... bench:  42,148,417 ns/iter (+/- 7,194,261)
test bench::shell_sort_2     ... bench:   8,326,559 ns/iter (+/- 434,494)
test bench::shell_sort_3     ... bench:   1,388,850 ns/iter (+/- 14,348)
```

（待补充）
