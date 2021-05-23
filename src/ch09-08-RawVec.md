# RawVec(RawVec)

我们实际上已经达到了一个有趣的情况:我们已经复制了用于指定缓冲区并在Vec和IntoIter中释放其内存的逻辑.现在我们已经实现了它并确定了 *实际的(actual)* 逻辑复制,现在是执行一些逻辑压缩的好时机.

我们将抽象出`(ptr, cap)`对并给它们分配,增长和释放的逻辑:

```Rust
struct RawVec<T> {
    ptr: Unique<T>,
    cap: usize,
}

impl<T> RawVec<T> {
    fn new() -> Self {
        assert!(mem::size_of::<T>() != 0, "We're not ready to handle ZSTs");
        RawVec { ptr: Unique::dangling(), cap: 0 }
    }

    // unchanged from Vec
    fn grow(&mut self) {
        unsafe {
            let align = mem::align_of::<T>();
            let elem_size = mem::size_of::<T>();

            let (new_cap, ptr) = if self.cap == 0 {
                let ptr = Global.allocate(Layout::array::<T>(1).unwrap());
                (1, ptr)
            } else {
                let new_cap = 2 * self.cap;
                let c: NonNull<T> = self.ptr.into();
                let ptr = Global.grow(c.cast(),
                                      Layout::array::<T>(self.cap).unwrap(),
                                      Layout::array::<T>(new_cap).unwrap());
                (new_cap, ptr)
            };

            // If allocate or reallocate fail, oom
            if ptr.is_err() {
                handle_alloc_error(Layout::from_size_align_unchecked(
                    new_cap * elem_size,
                    mem::align_of::<T>(),
                ))
            }
            let ptr = ptr.unwrap();

            self.ptr = Unique::new_unchecked(ptr.as_ptr() as *mut _);
            self.cap = new_cap;
        }
    }
}


impl<T> Drop for RawVec<T> {
    fn drop(&mut self) {
        if self.cap != 0 {
            unsafe {
                let c: NonNull<T> = self.ptr.into();
                Global.deallocate(c.cast(),
                                  Layout::array::<T>(self.cap).unwrap());
            }
        }
    }
}
```

并改变Vec如下:

```Rust
pub struct Vec<T> {
    buf: RawVec<T>,
    len: usize,
}

impl<T> Vec<T> {
    fn ptr(&self) -> *mut T { self.buf.ptr.as_ptr() }

    fn cap(&self) -> usize { self.buf.cap }

    pub fn new() -> Self {
        Vec { buf: RawVec::new(), len: 0 }
    }

    // push/pop/insert/remove largely unchanged:
    // * `self.ptr.as_ptr() -> self.ptr()`
    // * `self.cap -> self.cap()`
    // * `self.grow -> self.buf.grow()`
}

impl<T> Drop for Vec<T> {
    fn drop(&mut self) {
        while let Some(_) = self.pop() {}
        // deallocation is handled by RawVec
    }
}
```

最后我们可以真正简化IntoIter:

```Rust
pub struct IntoIter<T> {
    _buf: RawVec<T>, // we don't actually care about this. Just need it to live.
    start: *const T,
    end: *const T,
}

// next and next_back literally unchanged since they never referred to the buf

impl<T> Drop for IntoIter<T> {
    fn drop(&mut self) {
        // only need to ensure all our elements are read;
        // buffer will clean itself up afterwards.
        for _ in &mut *self {}
    }
}

impl<T> Vec<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        unsafe {
            // need to use ptr::read to unsafely move the buf out since it's
            // not Copy, and Vec implements Drop (so we can't destructure it).
            let buf = ptr::read(&self.buf);
            let len = self.len;
            mem::forget(self);

            IntoIter {
                start: buf.ptr.as_ptr(),
                end: buf.ptr..as_ptr().offset(len as isize),
                _buf: buf,
            }
        }
    }
}
```

好多了.
