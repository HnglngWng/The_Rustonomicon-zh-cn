# 解引用(Deref)

好的!我们已经实现了一个不错的最小堆栈.我们可以推入,我们可以弹出,我们可以自己清理.然而,我们理所当然地想要完整的功能.特别是,我们有一个合适的数组,但没有切片功能.这实际上很容易解决:我们可以实现`Deref<Target=[T]>`.这将神奇地使我们的Vec在各种条件下强迫并且表现得像切片.

我们需要的只是`slice::from_raw_parts`.它会为我们正确处理空切片.稍后,一旦我们设置了零大小的类型支持,它也将为那些工作.

```Rust
use std::ops::Deref;

impl<T> Deref for Vec<T> {
    type Target = [T];
    fn deref(&self) -> &[T] {
        unsafe {
            ::std::slice::from_raw_parts(self.ptr.as_ptr(), self.len)
        }
    }
}
```

也实现`DerefMut`:

```Rust
use std::ops::DerefMut;

impl<T> DerefMut for Vec<T> {
    fn deref_mut(&mut self) -> &mut [T] {
        unsafe {
            ::std::slice::from_raw_parts_mut(self.ptr.as_ptr(), self.len)
        }
    }
}
```

现在,我们有`len`,`first`,`last`,索引,切片,排序,`iter`,`iter_mut`以及切片提供的所有其他功能,甜!