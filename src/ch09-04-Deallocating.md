# 释放分配(Deallocating)

接下来我们应该实现Drop,这样我们就不会泄漏大量资源最简单的方法是调用`pop`直到它产生None,然后释放我们的缓冲区.请注意,如果`T: !Drop`,则不需要调用`pop`.理论上我们可以询问Rust是否`T` `need_drop`并省略对`pop`的调用.然而在实践中LLVM *非常(really)* 擅长移除像这样的简单的无副作用代码,所以除非你注意到它没有被剥离(在这种情况下它是),否则我不会打扰.

当`self.cap == 0`时,我们不能调用`alloc::dealloc`,因为在这种情况下我们实际上没有分配任何内存.

```Rust
impl<T> Drop for Vec<T> {
    fn drop(&mut self) {
        if self.cap != 0 {
            while let Some(_) = self.pop() { }
            let layout = Layout::array::<T>(self.cap).unwrap();
            unsafe {
                alloc::dealloc(self.ptr.as_ptr() as *mut u8, layout);
            }
        }
    }
}
```
