# 外部函数接口(Foreign Function Interface)

# 介绍(Introduction)

本指南将使用[snappy](https://github.com/google/snappy)压缩/解压缩库作为编写外部代码绑定的介绍.Rust目前无法直接调用C++库,但snappy包含一个C接口(在[`snappy-c.h`](https://github.com/google/snappy/blob/master/snappy-c.h)中有记录).

## 关于libc的说明(A note about libc)

其中许多示例使用[`libc`crate](https://crates.io/crates/libc),其中提供了C类型的各种类型定义等.如果你自己尝试这些示例,则需要将`libc`添加到`Cargo.toml`:

```Toml
[dependencies]
libc = "0.2.0"
```

并添加`extern crate libc;`到你crate的根.

# 调用外部函数(Calling foreign functions)

以下是调用外部函数的最小示例,如果安装了snappy,它将进行编译:

```Rust
extern crate libc;
use libc::size_t;

#[link(name = "snappy")]
extern {
    fn snappy_max_compressed_length(source_length: size_t) -> size_t;
}

fn main() {
    let x = unsafe { snappy_max_compressed_length(100) };
    println!("max compressed length of a 100 byte buffer: {}", x);
}
```

`extern`块是外部库中的函数签名列表,在本例中是平台的C ABI.`#[link(...)]`属性用于指示链接器链接到snappy库,以便解析符号.

外部函数被认为是不安全的,因此对它们的调用需要用`unsafe {}`包装作为对编译器的承诺,即真正包含的所有内容都是安全的.C库经常暴露非线程安全的接口,几乎所有接受指针参数的函数对所有可能的输入都无效,因为指针可能悬空,原始指针不属于Rust的安全内存模型.

将参数类型声明为外部函数时,Rust编译器无法检查声明是否正确,因此正确指定它是在运行时保持绑定正确的一部分.

`extern`块可以扩展为覆盖整个snappy API:

```Rust
extern crate libc;
use libc::{c_int, size_t};

#[link(name = "snappy")]
extern {
    fn snappy_compress(input: *const u8,
                       input_length: size_t,
                       compressed: *mut u8,
                       compressed_length: *mut size_t) -> c_int;
    fn snappy_uncompress(compressed: *const u8,
                         compressed_length: size_t,
                         uncompressed: *mut u8,
                         uncompressed_length: *mut size_t) -> c_int;
    fn snappy_max_compressed_length(source_length: size_t) -> size_t;
    fn snappy_uncompressed_length(compressed: *const u8,
                                  compressed_length: size_t,
                                  result: *mut size_t) -> c_int;
    fn snappy_validate_compressed_buffer(compressed: *const u8,
                                         compressed_length: size_t) -> c_int;
}
# fn main() {}
```

# 创建一个安全接口(Creating a safe interface)

原始C API需要被包装以提供内存安全性并使用更高级别的概念,如向量.库可以选择仅公开安全的高级接口,并隐藏不安全的内部细节.

包装期望缓冲区的函数涉及使用`slice::raw`模块来操作Rust向量作为指向内存的指针.Rust的向量保证是一个连续的内存块.长度是当前包含的元素数,容量是分配的内存元素的总大小.长度小于或等于容量.

```Rust
# extern crate libc;
# use libc::{c_int, size_t};
# unsafe fn snappy_validate_compressed_buffer(_: *const u8, _: size_t) -> c_int { 0 }
# fn main() {}
pub fn validate_compressed_buffer(src: &[u8]) -> bool {
    unsafe {
        snappy_validate_compressed_buffer(src.as_ptr(), src.len() as size_t) == 0
    }
}
```

上面的`validate_compressed_buffer`包装器使用了一个`unsafe`块,但它通过从函数签名中取消`unsafe`来保证调用它对所有输入都是安全的.

`snappy_compress`和`snappy_uncompress`函数更复杂,因为必须分配缓冲区来保存输出.

`snappy_max_compressed_length`函数可用于分配具有最大所需容量的向量来保存压缩输出.然后可以将向量作为输出参数传递给`snappy_compress`函数.还会传递输出参数以在压缩后检索的真实长度用于设置长度.

```Rust
# extern crate libc;
# use libc::{size_t, c_int};
# unsafe fn snappy_compress(a: *const u8, b: size_t, c: *mut u8,
#                           d: *mut size_t) -> c_int { 0 }
# unsafe fn snappy_max_compressed_length(a: size_t) -> size_t { a }
# fn main() {}
pub fn compress(src: &[u8]) -> Vec<u8> {
    unsafe {
        let srclen = src.len() as size_t;
        let psrc = src.as_ptr();

        let mut dstlen = snappy_max_compressed_length(srclen);
        let mut dst = Vec::with_capacity(dstlen as usize);
        let pdst = dst.as_mut_ptr();

        snappy_compress(psrc, srclen, pdst, &mut dstlen);
        dst.set_len(dstlen as usize);
        dst
    }
}
```

解压缩是类似的,因为snappy将未压缩的大小存储为压缩格式的一部分,`snappy_uncompressed_length`将检索所需的确切缓冲区大小.

```Rust
# extern crate libc;
# use libc::{size_t, c_int};
# unsafe fn snappy_uncompress(compressed: *const u8,
#                             compressed_length: size_t,
#                             uncompressed: *mut u8,
#                             uncompressed_length: *mut size_t) -> c_int { 0 }
# unsafe fn snappy_uncompressed_length(compressed: *const u8,
#                                      compressed_length: size_t,
#                                      result: *mut size_t) -> c_int { 0 }
# fn main() {}
pub fn uncompress(src: &[u8]) -> Option<Vec<u8>> {
    unsafe {
        let srclen = src.len() as size_t;
        let psrc = src.as_ptr();

        let mut dstlen: size_t = 0;
        snappy_uncompressed_length(psrc, srclen, &mut dstlen);

        let mut dst = Vec::with_capacity(dstlen as usize);
        let pdst = dst.as_mut_ptr();

        if snappy_uncompress(psrc, srclen, pdst, &mut dstlen) == 0 {
            dst.set_len(dstlen as usize);
            Some(dst)
        } else {
            None // SNAPPY_INVALID_INPUT
        }
    }
}
```

然后,我们可以添加一些测试来展示如何使用它们.

```Rust
# extern crate libc;
# use libc::{c_int, size_t};
# unsafe fn snappy_compress(input: *const u8,
#                           input_length: size_t,
#                           compressed: *mut u8,
#                           compressed_length: *mut size_t)
#                           -> c_int { 0 }
# unsafe fn snappy_uncompress(compressed: *const u8,
#                             compressed_length: size_t,
#                             uncompressed: *mut u8,
#                             uncompressed_length: *mut size_t)
#                             -> c_int { 0 }
# unsafe fn snappy_max_compressed_length(source_length: size_t) -> size_t { 0 }
# unsafe fn snappy_uncompressed_length(compressed: *const u8,
#                                      compressed_length: size_t,
#                                      result: *mut size_t)
#                                      -> c_int { 0 }
# unsafe fn snappy_validate_compressed_buffer(compressed: *const u8,
#                                             compressed_length: size_t)
#                                             -> c_int { 0 }
# fn main() { }

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid() {
        let d = vec![0xde, 0xad, 0xd0, 0x0d];
        let c: &[u8] = &compress(&d);
        assert!(validate_compressed_buffer(c));
        assert!(uncompress(c) == Some(d));
    }

    #[test]
    fn invalid() {
        let d = vec![0, 0, 0, 0];
        assert!(!validate_compressed_buffer(&d));
        assert!(uncompress(&d).is_none());
    }

    #[test]
    fn empty() {
        let d = vec![];
        assert!(!validate_compressed_buffer(&d));
        assert!(uncompress(&d).is_none());
        let c = compress(&d);
        assert!(validate_compressed_buffer(&c));
        assert!(uncompress(&c) == Some(d));
    }
}
```

# 析构函数(Destructors)

外部库经常将资源的所有权交给调用代码.当发生这种情况时,我们必须使用Rust的析构函数来提供安全性并保证释放这些资源(特别是在恐慌的情况下).

有关析构函数的更多信息,请参阅Drop trait.

# 从C代码到Rust函数的回调(Callbacks from C code to Rust functions)

某些外部库需要使用回调来将其当前状态或中间数据报告给调用者.可以将Rust中定义的函数传递给外部库.对此的要求是回调函数被标记为`extern`,具有正确的调用约定,以使其可以从C代码调用.

然后可以通过对C库的注册调用发送回调函数,然后从那里调用.

一个基本的例子是:

Rust代码:

```Rust
extern fn callback(a: i32) {
    println!("I'm called from C with value {0}", a);
}

#[link(name = "extlib")]
extern {
   fn register_callback(cb: extern fn(i32)) -> i32;
   fn trigger_callback();
}

fn main() {
    unsafe {
        register_callback(callback);
        trigger_callback(); // Triggers the callback.
    }
}
```

C代码:

```Rust
typedef void (*rust_callback)(int32_t);
rust_callback cb;

int32_t register_callback(rust_callback callback) {
    cb = callback;
    return 1;
}

void trigger_callback() {
  cb(7); // Will call callback(7) in Rust.
}
```

在这个例子中,Rust的`main()`将在C中调用`trigger_callback()`,这反过来会回调Rust中的`callback()`.

## 将回调定向到Rust对象(Targeting callbacks to Rust objects)

前一个例子展示了如何从C代码调用全局函数.但是,通常希望回调针对特殊的Rust对象.这可以是表示相应C对象的包装器的对象.

这可以通过将指向对象的原始指针传递给C库来实现.然后,C库可以在通知中包含指向Rust对象的指针.这将允许回调不安全地访问引用的Rust对象.

Rust代码:

```Rust
#[repr(C)]
struct RustObject {
    a: i32,
    // Other members...
}

extern "C" fn callback(target: *mut RustObject, a: i32) {
    println!("I'm called from C with value {0}", a);
    unsafe {
        // Update the value in RustObject with the value received from the callback:
        (*target).a = a;
    }
}

#[link(name = "extlib")]
extern {
   fn register_callback(target: *mut RustObject,
                        cb: extern fn(*mut RustObject, i32)) -> i32;
   fn trigger_callback();
}

fn main() {
    // Create the object that will be referenced in the callback:
    let mut rust_object = Box::new(RustObject { a: 5 });

    unsafe {
        register_callback(&mut *rust_object, callback);
        trigger_callback();
    }
}
```

C代码:

```Rust
typedef void (*rust_callback)(void*, int32_t);
void* cb_target;
rust_callback cb;

int32_t register_callback(void* callback_target, rust_callback callback) {
    cb_target = callback_target;
    cb = callback;
    return 1;
}

void trigger_callback() {
  cb(cb_target, 7); // Will call callback(&rustObject, 7) in Rust.
}
```

## 异步回调(Asynchronous callbacks)

在前面给出的示例中,回调被调用为对外部C库的函数调用的直接反应.对当前线程的控制从Rust切换到C到Rust以执行回调,但最后在调用触发回调的函数的同一线程上执行回调.

当外部库产生自己的线程并从那里调用回调时,事情变得更加复杂.在这些情况下,对回调内部的Rust数据结构的访问尤其不安全,必须使用正确的同步机制.除了像互斥锁这样的经典同步机制之外,Rust中的一种可能性是使用通道(在`std::sync::mpsc`中)将数据从调用回调的C线程转发到Rust线程.

如果异步回调针对Rust地址空间中的特殊对象,那么在相应的Rust对象被销毁之后,C库也不再需要执行回调.这可以通过在对象的析构函数中取消注册回调并以确保在注销后不执行回调的方式设计库来实现.

# 链接(Linking)

`extern`块上的`link`属性提供了基本的构建块,用于指示rustc如何链接到本机库.今天有两种可接受的链接属性形式:

- `#[link(name ="foo")]`

- `#[link(name ="foo",kind ="bar")]`

在这两种情况下,`foo`是我们要链接到的本机库的名称,而在第二种情况下,`bar`是编译器链接到的本机库的类型.目前有三种已知类型的本机库:

- 动态(Dynamic)--`#[link(name ="readline")]`

- 静态(Static)--`#[link(name ="my_build_dependency",kind ="static")]`

- 框架(Frameworks)--`#[link(name ="CoreFoundation",kind ="framework")]`

请注意,框架仅适用于macOS目标.

不同`kind`的值旨在区分本机库如何参与链接.从链接角度来看,Rust编译器会创建两种工件:partial(rlib/staticlib)和final(dylib/binary).原生动态库和框架依赖关系传播到最终工件边界,而静态库依赖关系根本不传播,因为静态库直接集成到后续工件中.

有关如何使用此模型的一些示例如下:

- 本机构建依赖项.在编写一些Rust代码时,有时需要一些C/C++粘合剂,但是以库格式分发C/C++代码是一种负担.在这种情况下,代码将存档到`libfoo.a`中,然后Rust crate将通过`#[link(name ="foo",kind ="static")]`声明依赖项.

无论crate的输出风格如何,本机静态库都将包含在输出中,这意味着不需要分发本机静态库.

- 正常的动态依赖.常见的系统库(如`readline`)可在大量系统上使用,并且通常无法找到这些库的静态副本.当此依赖项包含在Rust crate中时,部分目标(如rlibs)将不会链接到库,但是当rlib包含在最终目标(如二进制文件)中时,将链接本机库.

在macOS上,框架的行为与动态库的语义相同.

# 不安全块(Unsafe blocks)

某些操作(如解引用原始指针或调用已标记为不安全的函数)仅允许在不安全的块内.不安全块隔离不安全,并且是对编译器的承诺,即不安全不会泄漏出块.

另一方面,不安全的函数向世界宣传它.一个不安全的函数写成这样:

```Rust
unsafe fn kaboom(ptr: *const i32) -> i32 { *ptr }
```

只能从`unsafe`块或其他`unsafe`函数调用此函数.

# 访问外部全局变量(Accessing foreign globals)

```Rust
extern crate libc;

#[link(name = "readline")]
extern {
    static rl_readline_version: libc::c_int;
}

fn main() {
    println!("You have readline version {} installed.",
             unsafe { rl_readline_version as i32 });
}
```

或者,你可能需要更改外部接口提供的全局状态.要做到这一点,静态可以用`mut`声明,所以我们可以改变它们.

```Rust
extern crate libc;

use std::ffi::CString;
use std::ptr;

#[link(name = "readline")]
extern {
    static mut rl_prompt: *const libc::c_char;
}

fn main() {
    let prompt = CString::new("[my-awesome-shell] $").unwrap();
    unsafe {
        rl_prompt = prompt.as_ptr();

        println!("{:?}", rl_prompt);

        rl_prompt = ptr::null();
    }
}
```

请注意,与静态`mut`的所有交互都是不安全的,包括读写.处理全局可变状态需要非常谨慎.

# 外部调用约定(Foreign calling conventions)

大多数外部代码都公开了一个C ABI,而Rust在调用外部函数时默认使用平台的C调用约定.一些外部函数,尤其是Windows API,使用其他调用约定.Rust提供了一种告诉编译器使用哪种约定的方法:

```Rust
extern crate libc;

#[cfg(all(target_os = "win32", target_arch = "x86"))]
#[link(name = "kernel32")]
#[allow(non_snake_case)]
extern "stdcall" {
    fn SetEnvironmentVariableA(n: *const u8, v: *const u8) -> libc::c_int;
}
# fn main() { }
```

这适用于整个`extern`块.支持的ABI约束列表如下:

- `stdcall`

- `aapcs`

- `cdecl`

- `fastcall`

- `vectorcall`这当前隐藏在`abi_vectorcall`门后面,可能会有变化.

- `Rust`

- `rust-intrinsic`

- `system`

- `C`

- `win64`

- `sysv64`

此列表中的大多数abis都是不言自明的,但`system` abi可能看起来有点奇怪.此约束选择适当的ABI用于与目标库进行互操作.例如,在带有x86架构的win32上,这意味着所使用的abi将是`stdcall`.但是,在x86_64上,windows使用`C`调用约定,因此将使用`C`.这意味着在我们之前的示例中,我们可以使用`extern "system" { ... }`来为所有Windows系统定义块,而不仅仅是x86.

# 与外部代码的互操作性(Interoperability with foreign code)

Rust保证只有在应用`#[repr(C)]`属性时,结构的布局才能与平台在C中的表示兼容.`#[repr(C,packed)]`可用于布局没有填充的struct成员.`#[repr(C)]`也可以应用于枚举.

Rust拥有的boxes(`Box<T>`)使用不可为空的指针作为指向包含对象的句柄.但是,它们不应手动创建,因为它们由内部分配器管理.可以安全地假设引用是直接指向该类型的非可空指针.但是,打破借用检查或可变性规则并不能保证是安全的,所以如果需要,则更喜欢使用原始指针(`*`),因为编译器无法对它们做出尽可能多的假设.

向量和字符串共享相同的基本内存布局,`vec`和`str`模块中提供了工具,用于使用C API.但是,字符串不会以`\0`结尾.如果你需要一个NUL终止的字符串来与C互操作,你应该使用在`std::ffi`模块中的`CString`类型.

[crate.io上的`libc` crate](https://crates.io/crates/libc)包含`libc`模块中C标准库的类型别名和函数定义,默认情况下是针对`libc`和`libm`的Rust链接.

# 可变参数函数(Variadic functions)

在C中,函数可以是"variadic",这意味着它们接受可变数量的参数.这可以在Rust中通过在外部函数声明的参数列表中指定`...`来实现:

```Rust
extern {
    fn foo(x: i32, ...);
}

fn main() {
    unsafe {
        foo(10, 20, 30, 40, 50);
    }
}
```

普通的Rust函数 *不能(not)* 是可变参数的:

```Rust
// This will not compile

fn foo(x: i32, ...) { }
```

# "可空指针优化"(The "nullable pointer optimization")

某些Rust类型被定义为永远不为`null`.这包括引用(`&T`,`&mut T`),boxes(`Box<T>`)和函数指针(`extern "abi" fn()`).当与C接口时,经常使用可能为`null`的指针,这似乎需要一些凌乱的`transmute`和/或不安全的代码来处理与Rust类型之间的转换.但是,该语言提供了一种解决方法.

作为一种特殊情况,如果`enum`恰好包含两个变体,其中一个变体不包含任何数据,另一个包含上面列出的一个非可空类型的字段,则枚举符合"可空指针优化(nullable pointer optimization)"的条件.这意味着判别式不需要额外的空间;相反,通过将`null`值放入非可空字段来表示空变体.这称为"优化",但与其他优化不同,它保证适用于符合条件的类型.

利用可空指针优化的最常见类型是`Option<T>`,其中`None`对应于`null`.所以`Option<extern "C" fn(c_int) -> c_int>`是使用C ABI(对应于C类型`int (*)(int)`)表示可空函数指针的正确方法.

这是一个人为的例子.假设一些C库有一个用于注册回调的工具,在某些情况下会调用它.回调传递一个函数指针和一个整数,它应该以整数作为参数运行函数.因此,我们在两个方向上都有跨越FFI边界的函数指针.

```Rust
extern crate libc;
use libc::c_int;

# #[cfg(hidden)]
extern "C" {
    /// Registers the callback.
    fn register(cb: Option<extern "C" fn(Option<extern "C" fn(c_int) -> c_int>, c_int) -> c_int>);
}
# unsafe fn register(_: Option<extern "C" fn(Option<extern "C" fn(c_int) -> c_int>,
#                                            c_int) -> c_int>)
# {}

/// This fairly useless function receives a function pointer and an integer
/// from C, and returns the result of calling the function with the integer.
/// In case no function is provided, it squares the integer by default.
extern "C" fn apply(process: Option<extern "C" fn(c_int) -> c_int>, int: c_int) -> c_int {
    match process {
        Some(f) => f(int),
        None    => int * int
    }
}

fn main() {
    unsafe {
        register(Some(apply));
    }
}
```

C端的代码如下所示:

```Rust
void register(void (*f)(void (*)(int), int)) {
    ...
}
```

不需要`transmute`.

# 从C调用Rust代码(Calling Rust code from C)

你可能希望以某种方式编译Rust代码,以便可以从C调用它.这很简单,但需要一些东西:

```Rust
#[no_mangle]
pub extern "C" fn hello_rust() -> *const u8 {
    "Hello, world!\0".as_ptr()
}
# fn main() {}
```

`extern "C"`使得该函数遵循C调用约定,如上面"外部调用约定"中所讨论的.`no_mangle`属性关闭了Rust的名称修改,因此更容易链接到.

# FFI和恐慌(FFI and panics)

在与FFI合作时要注意`panic!`是很重要的.`panic!`跨越FFI边界是未定义行为.如果你编写的代码可能会出现恐慌,你应该在一个带有`catch_unwind`的闭包中运行它:

```Rust
use std::panic::catch_unwind;

#[no_mangle]
pub extern fn oh_no() -> i32 {
    let result = catch_unwind(|| {
        panic!("Oops!");
    });
    match result {
        Ok(_) => 0,
        Err(_) => 1,
    }
}

fn main() {}
```

请注意,`catch_unwind`只会捕获展开的(unwinding)恐慌,而不是那些中止该过程的人.有关更多信息,请参阅`catch_unwind`的文档.

# 表示不透明的结构(Representing opaque structs)

有时候,C库想要提供指向某个东西的指针,但是不要让你知道它想要的东西的内部细节.最简单的方法是使用`void *`参数:

```C
void foo(void *arg);
void bar(void *arg);
```

我们可以使用`c_void`类型在Rust中表示:

```Rust
extern crate libc;

extern "C" {
    pub fn foo(arg: *mut libc::c_void);
    pub fn bar(arg: *mut libc::c_void);
}
# fn main() {}
```

这是处理这种情况的完美有效方式.但是,我们可以做得更好.为了解决这个问题,一些C库将改为创建一个`struct`,其中struct的细节和内存布局是私有的.这提供了一些类型的安全性.这些结构称为"不透明的(opaque)".这是C中的一个例子:

```C
struct Foo; /* Foo is a structure, but its contents are not part of the public interface */
struct Bar;
void foo(struct Foo *arg);
void bar(struct Bar *arg);
```

要在Rust中执行此操作,让我们创建自己的opaque类型:

```Rust
#[repr(C)] pub struct Foo { _private: [u8; 0] }
#[repr(C)] pub struct Bar { _private: [u8; 0] }

extern "C" {
    pub fn foo(arg: *mut Foo);
    pub fn bar(arg: *mut Bar);
}
# fn main() {}
```

通过包含私有字段而不包含构造函数,我们创建了一个opaque类型,我们无法在此模块之外进行实例化.(没有字段的结构可以被任何人实例化.)我们还想在FFI中使用这种类型,所以我们必须添加`#[repr(C)]`.为避免在FFI中使用`()`时受到警告,我们改为使用一个空数组,它与空的(empty)类型一样,但与FFI兼容.

但是因为我们的`Foo`和`Bar`类型不同,我们将在它们之间获得类型安全性,所以我们不会意外地将指向`Foo`的指针传递给`bar()`.

请注意,使用空枚举作为FFI类型是一个非常糟糕的主意.编译器依赖于空枚举是无人居住的,因此处理类型`&Empty`的值是一个巨大的步枪,可能导致错误的程序行为(通过触发未定义行为).
