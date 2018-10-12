# 生命周期省略(Lifetime Elision)

为了使常见的模式更符合人体工程学,Rust允许在函数签名中 *省略(elided)* 生命周期.

*生命周期位置(lifetime position)* 是你可以在类型中编写生命周期的任何地方:

```Rust
&'a T
&'a mut T
T<'a>
```

生命周期位置可以表现为"输入(input)"或"输出(output)":

- 对于`fn`定义,输入指的是`fn`定义中形式参数的类型,而输出指的是结果类型.所以`fn foo(s:& str) ->(&str, &str)`在输入位置省略了一个生命周期,在输出位置省略了两个生命周期.请注意,`fn`方法定义的输入位置不包括方法的`impl`头中出现的生命周期(对于默认方法,也不包括trait头中出现的生命周期).

- 在将来,应该可以以相同的方式省略`impl`头.

省略规则如下:

- 输入位置中的每个省略的生命周期变为不同的生命周期参数.

- 如果只有一个输入生命周期位置(省略或未省略),该生命周期分配给 *所有(all)* 省略的输出生命周期.

- 如果有多个输入生命周期位置,但其中一个是`&self`或`&mut self`,`self`的生命周期被分配给 *所有(all)* 省略的输出生命周期.

- 否则,省略输出生命周期是错误的.

例子:

```Rust
fn print(s: &str);                                      // elided
fn print<'a>(s: &'a str);                               // expanded

fn debug(lvl: usize, s: &str);                          // elided
fn debug<'a>(lvl: usize, s: &'a str);                   // expanded

fn substr(s: &str, until: usize) -> &str;               // elided
fn substr<'a>(s: &'a str, until: usize) -> &'a str;     // expanded

fn get_str() -> &str;                                   // ILLEGAL

fn frob(s: &str, t: &str) -> &str;                      // ILLEGAL

fn get_mut(&mut self) -> &mut T;                        // elided
fn get_mut<'a>(&'a mut self) -> &'a mut T;              // expanded

fn args<T: ToCStr>(&mut self, args: &[T]) -> &mut Command                  // elided
fn args<'a, 'b, T: ToCStr>(&'a mut self, args: &'b [T]) -> &'a mut Command // expanded

fn new(buf: &mut [u8]) -> BufWriter;                    // elided
fn new<'a>(buf: &'a mut [u8]) -> BufWriter<'a>          // expanded
```