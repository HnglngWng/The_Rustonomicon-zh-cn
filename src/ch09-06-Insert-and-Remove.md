# 插入和移除(Insert and Remove)

切片 *未(not)* 提供的东西是`insert`和`remove`,所以接下来做.

插入需要将目标索引处的所有元素向右移动(shift)一个.要做到这一点,我们需要使用`ptr::copy`,这是C的`memmove`我们的版本.这会将一些内存从一个位置复制到另一个位置,正确处理源和目标重叠的情况(这肯定会发生在这里).

如果我们在索引`i`处插入,我们希望使用旧len将`[i .. len]`移动到`[i + 1 .. len+1]`.

```Rust
pub fn insert(&mut self, index: usize, elem: T) {
    // Note: `<=` because it's valid to insert after everything
    // which would be equivalent to push.
    assert!(index <= self.len, "index out of bounds");
    if self.cap == self.len { self.grow(); }

    unsafe {
        // ptr::copy(src, dest, len): "copy from src to dest len elems"
        ptr::copy(self.ptr.as_ptr().add(index),
                  self.ptr.as_ptr().add(index + 1),
                  self.len - index);
        ptr::write(self.ptr.as_ptr().add(index), elem);
        self.len += 1;
    }
}
```

移除行为以相反的方式.我们需要使用 *新的(new)* len将所有元素从`[i+1 .. len+1]`移动到`[i .. len]`.

```Rust
pub fn remove(&mut self, index: usize) -> T {
    // Note: `<` because it's *not* valid to remove after everything
    assert!(index < self.len, "index out of bounds");
    unsafe {
        self.len -= 1;
        let result = ptr::read(self.ptr.as_ptr().offset(index as isize));
        ptr::copy(self.ptr.as_ptr().add(index + 1),
                  self.ptr.as_ptr().add(index),
                  self.len - index);
        result
    }
}
```