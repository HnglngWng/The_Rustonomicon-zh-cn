# IntoIter(IntoIter)

让我们继续编写迭代器.由于Deref的魔力,`iter`和`iter_mut`已经为我们写了.然而,有两个有趣的迭代器,Vec提供的切片不能:`into_iter`和`drain`.

IntoIter通过值消费Vec,因此可以通过值生成其元素.为了实现这一点,IntoIter需要控制Vec的分配.

IntoIter也需要DoubleEnded,以便从两端读取.从后面读取可以实现为调用`pop`,但从前面读取更难.我们可以调用`remove(0)`,但这会非常昂贵.相反,我们将只使用ptr::read从Vec的任一端复制值,而根本不改变缓冲区.

为此,我们将使用非常常见的C习语来进行数组迭代.我们会制作两个指针;一个指向数组的开头,一个指向一个结尾的元素.当我们想要一端的元素时,我们将读出该末端指向的值并将指针移过一个.当两个指针相等时,我们知道我们已经完成了.

注意,读取和偏移的顺序对于`next`和`next_back`是相反的.对于`next_back`,指针总是在它想要读取的元素之后,而对于`next`,指针总是在它想要读取的元素.要了解这是为什么,请考虑除了一个元素之外的每个元素都已经产生的情况.

该数组看起来像这样:

```Rust
          S  E
[X, X, X, O, X, X, X]
```

如果E直接指向它想要接下来产生的元素,那么它将与没有更多元素产生的情况无法区分.

虽然我们在迭代期间实际上并不关心它,但我们还需要保留Vec的分配信息,以便在删除IntoIter后释放它.

所以我们将使用以下结构:

```Rust
pub struct IntoIter<T> {
    buf: Unique<T>,
    cap: usize,
    start: *const T,
    end: *const T,
}
```

这就是我们最终的初始化:

```Rust
impl<T> Vec<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        // Can't destructure Vec since it's Drop
        let ptr = self.ptr;
        let cap = self.cap;
        let len = self.len;

        // Make sure not to drop Vec since that will free the buffer
        mem::forget(self);

        unsafe {
            IntoIter {
                buf: ptr,
                cap: cap,
                start: ptr.as_ptr(),
                end: if cap == 0 {
                    // can't offset off this pointer, it's not allocated!
                    ptr.as_ptr()
                } else {
                    ptr.as_ptr().offset(len as isize)
                },
            }
        }
    }
}
```

这是向前迭代:

```Rust
impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                let result = ptr::read(self.start);
                self.start = self.start.offset(1);
                Some(result)
            }
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        let len = (self.end as usize - self.start as usize)
                  / mem::size_of::<T>();
        (len, Some(len))
    }
}
```

这是向后迭代:

```Rust
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                self.end = self.end.offset(-1);
                Some(ptr::read(self.end))
            }
        }
    }
}
```

因为IntoIter获取其分配的所有权,所以需要实现Drop来释放它.但是,它还希望实现Drop以删除它所包含的未生成的任何元素.

```Rust
impl<T> Drop for IntoIter<T> {
    fn drop(&mut self) {
        if self.cap != 0 {
            // drop any remaining elements
            for _ in &mut *self {}

            unsafe {
                let c: NonNull<T> = self.buf.into();
                Global.deallocate(c.cast(),
                                  Layout::array::<T>(self.cap).unwrap());
            }
        }
    }
}
```
