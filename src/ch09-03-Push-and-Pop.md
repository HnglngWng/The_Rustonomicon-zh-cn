# 推入和弹出(Push and Pop)

好的.我们可以初始化.我们可以分配.让我们实际实现一些函数!让我们从`push`开始.它需要做的就是检查我们是否已经完全增长,无条件地写入下一个索引,然后增加我们的长度.

要做写,我们必须小心不要计算我们想要写入的内存.在最坏的情况下,它是来自分配器的真正未初始化的内存.最好的情况,只是我们弹出的一些旧值.无论哪种方式,我们不能只是索引内存并解引用它,因为这会将内存计算为T的有效实例.更糟糕的是,`foo[idx] = x`会尝试在`foo[idx]`的旧值上调用`drop`.

正确的方法是使用`ptr::write`,它只是用我们提供的值的位来盲目地覆盖目标地址.不涉及求值.

对于`push`,如果旧len(在调用push之前)为0,那么我们要写入第0个索引.所以我们应该用老len偏移.

```Rust
pub fn push(&mut self, elem: T) {
    if self.len == self.cap { self.grow(); }

    unsafe {
        ptr::write(self.ptr.as_ptr().offset(self.len as isize), elem);
    }

    // Can't fail, we'll OOM first.
    self.len += 1;
}
```

简单!`pop`怎么样?虽然这次我们想要访问的索引被初始化,但是Rust不会让我们解引用内存的位置来移出值,因为这会使内存为未初始化状态!为此,我们需要`ptr::read`,它只复制目标地址中的位并将其解释为类型T的值.这将使该地址的内存逻辑上未初始化,即使实际上有一个非常好的实例T在那里的.

对于`pop`,如果旧len为1,我们想要读出第0个索引.所以我们应该用新的len偏移.

```Rust
pub fn pop(&mut self) -> Option<T> {
    if self.len == 0 {
        None
    } else {
        self.len -= 1;
        unsafe {
            Some(ptr::read(self.ptr.as_ptr().offset(self.len as isize)))
        }
    }
}
```