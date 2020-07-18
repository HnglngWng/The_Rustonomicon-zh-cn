# 使用不安全(Working with Unsafe)

Rust通常只为我们提供了以范围和二进制方式讨论Unsafe Rust的工具.不幸的是，现实要复杂得多.例如,考虑以下玩具函数:

```Rust
fn index(idx: usize, arr: &[u8]) -> Option<u8> {
    if idx < arr.len() {
        unsafe {
            Some(*arr.get_unchecked(idx))
        }
    } else {
        None
    }
}
```

此函数安全且正确的.我们检查索引是否在边界内,如果是,则以未经检查的方式索引到数组中.我们说这样一个正确的不安全实现的函数是 *可靠的(sound)* ,这意味着安全代码无法通过它导致未定义行为(记住,这是Safe Rust的唯一基本属性).

但即使在这样一个微不足道的函数中,不安全块的范围也值得怀疑.考虑将`<`更改为`<=`:

```Rust
fn index(idx: usize, arr: &[u8]) -> Option<u8> {
    if idx <= arr.len() {
        unsafe {
            Some(*arr.get_unchecked(idx))
        }
    } else {
        None
    }
}
```

这个程序现在是 *不可靠的(unsound)* ,Safe Rust会导致未定义的行为,但 *我们只修改了安全代码(we only modified safe code)* .这是安全的根本问题:它是非局部的(non-local). 我们不安全操作的可靠性必然取决于其他"安全"操作所建立的状态.

安全性是模块化,从某种意义上说,选择不安全并不要求你考虑任意其他类型的不良.例如,对切片执行未经检查的索引并不意味着你突然需要担心切片为空(null)或包含未初始化的内存.没有什么从根本上改变.然而,安全性 *不是(isn't)* 模块化的,从某种意义上说,程序本质上是有状态的,并且你的不安全操作可能依赖于任意其他状态,

当我们合并实际的持久状态时,这种非局部性会变得更糟.考虑一个`Vec`的简单实现:

```Rust
use std::ptr;

// Note: This definition is naive. See the chapter on implementing Vec.
pub struct Vec<T> {
    ptr: *mut T,
    len: usize,
    cap: usize,
}

// Note this implementation does not correctly handle zero-sized types.
// See the chapter on implementing Vec.
impl<T> Vec<T> {
    pub fn push(&mut self, elem: T) {
        if self.len == self.cap {
            // not important for this example
            self.reallocate();
        }
        unsafe {
            ptr::write(self.ptr.offset(self.len as isize), elem);
            self.len += 1;
        }
    }
    # fn reallocate(&mut self) { }
}

# fn main() {}
```

此代码非常简单,可以合理地审核和非正式验证.现在考虑添加以下方法:

```Rust
fn make_room(&mut self) {
    // grow the capacity
    self.cap += 1;
}
```

此代码是100%的Safe Rust,但它也完全不健全.更改容量违反了`Vec`的不变量(该`cap`反映了Vec中分配的空间).这不是Vec其他东西可以防范的事情.它 *必须(has)* 信任容量字段,因为无法验证它.

因为它依赖于struct字段的不变量,所以这个`unsafe`的代码不仅会污染整个函数:它会污染整个 *模块(module)* .通常，限制不安全代码范围的唯一防弹(bullet-proof)方式是在隐私的模块边界.

然而,这非常 *有效(perfectly)* .`make_room`的存在对于Vec的健全性来说 *不是(not)* 问题,因为我们没有将其标记为公有.只有定义此函数的模块才能调用它.此外,`make_room`直接访问Vec的私有字段,因此它只能与Vec在同一模块中编写.

因此,我们可以编写一个完全安全的抽象,它依赖于复杂的不变量.这对Safe Rust和Unsafe Rust之间的关系 *至关重要(critical)* .

我们已经看到Unsafe代码必须信任 *某些(some)* Safe代码,但不应该信任 *泛型(generic)* Safe代码.出于类似的原因,隐私对于不安全的代码很重要:它使我们不必相信宇宙中的所有安全代码都会破坏我们的可信状态.

安全活着!
