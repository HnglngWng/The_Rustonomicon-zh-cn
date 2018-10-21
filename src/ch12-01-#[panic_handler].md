# #[panic_handler](#[panic_handler])

`#[panic_handler]`用于定义`panic!`的行为在`#![no_std]`应用程序中.`#[panic_handler]`属性必须应用于具有签名`fn(&PanicInfo) -> !`的函数,并且这样的函数必须在binary/dylib/cdylib crate的依赖图中出现 *一次(once)* . `PanicInfo`的API可以在API文档中找到.

鉴于`#![no_std]`应用程序没有 *标准(standard)* 输出和一些`#![no_std]`应用程序,例如嵌入式应用程序,需要不同的恐慌行为进行开发和发布它可能有助于恐慌crate,只包含`#[panic_handler]`的crate.这样,应用程序可以通过简单地链接到不同的恐慌crate来轻松交换恐慌行为.

下面显示了一个示例,其中应用程序具有不同的恐慌行为,具体取决于是使用开发(dev)配置文件(`cargo build`)还是使用发布(release)配置文件(`argo build --release`)进行编译.

```Rust
// crate: panic-semihosting -- log panic messages to the host stderr using semihosting

#![no_std]

use core::fmt::{Write, self};
use core::panic::PanicInfo;

struct HStderr {
    // ..
#     _0: (),
}
#
# impl HStderr {
#     fn new() -> HStderr { HStderr { _0: () } }
# }
#
# impl fmt::Write for HStderr {
#     fn write_str(&mut self, _: &str) -> fmt::Result { Ok(()) }
# }

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    let mut host_stderr = HStderr::new();

    // logs "panicked at '$reason', src/main.rs:27:4" to the host stderr
    writeln!(host_stderr, "{}", info).ok();

    loop {}
}
```

```Rust
// crate: panic-halt -- halt the thread on panic; messages are discarded

#![no_std]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

```Rust
// crate: app

#![no_std]

// dev profile
#[cfg(debug_assertions)]
extern crate panic_semihosting;

// release profile
#[cfg(not(debug_assertions))]
extern crate panic_halt;

// omitted: other `extern crate`s

fn main() {
    // ..
}
```