# 最终代码(The Final Code)

```Rust
#![feature(ptr_internals)]
#![feature(allocator_api)]

use std::ptr::{Unique, NonNull, self};
use std::mem;
use std::ops::{Deref, DerefMut};
use std::marker::PhantomData;
use std::alloc::{Alloc, GlobalAlloc, Layout, Global, handle_alloc_error};

struct RawVec<T> {
    ptr: Unique<T>,
    cap: usize,
}

impl<T> RawVec<T> {
    fn new() -> Self {
        // !0 is usize::MAX. This branch should be stripped at compile time.
        let cap = if mem::size_of::<T>() == 0 { !0 } else { 0 };

        // Unique::empty() doubles as "unallocated" and "zero-sized allocation"
        RawVec { ptr: Unique::empty(), cap: cap }
    }

    fn grow(&mut self) {
        unsafe {
            let elem_size = mem::size_of::<T>();

            // since we set the capacity to usize::MAX when elem_size is
            // 0, getting to here necessarily means the Vec is overfull.
            assert!(elem_size != 0, "capacity overflow");

            let (new_cap, ptr) = if self.cap == 0 {
                let ptr = Global.alloc(Layout::array::<T>(1).unwrap());
                (1, ptr)
            } else {
                let new_cap = 2 * self.cap;
                let c: NonNull<T> = self.ptr.into();
                let ptr = Global.realloc(c.cast(),
                                         Layout::array::<T>(self.cap).unwrap(),
                                         Layout::array::<T>(new_cap).unwrap().size());
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
        let elem_size = mem::size_of::<T>();
        if self.cap != 0 && elem_size != 0 {
            unsafe {
                let c: NonNull<T> = self.ptr.into();
                Global.dealloc(c.cast(),
                               Layout::array::<T>(self.cap).unwrap());
            }
        }
    }
}

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
    pub fn push(&mut self, elem: T) {
        if self.len == self.cap() { self.buf.grow(); }

        unsafe {
            ptr::write(self.ptr().offset(self.len as isize), elem);
        }

        // Can't fail, we'll OOM first.
        self.len += 1;
    }

    pub fn pop(&mut self) -> Option<T> {
        if self.len == 0 {
            None
        } else {
            self.len -= 1;
            unsafe {
                Some(ptr::read(self.ptr().offset(self.len as isize)))
            }
        }
    }

    pub fn insert(&mut self, index: usize, elem: T) {
        assert!(index <= self.len, "index out of bounds");
        if self.cap() == self.len { self.buf.grow(); }

        unsafe {
            if index < self.len {
                ptr::copy(self.ptr().offset(index as isize),
                          self.ptr().offset(index as isize + 1),
                          self.len - index);
            }
            ptr::write(self.ptr().offset(index as isize), elem);
            self.len += 1;
        }
    }

    pub fn remove(&mut self, index: usize) -> T {
        assert!(index < self.len, "index out of bounds");
        unsafe {
            self.len -= 1;
            let result = ptr::read(self.ptr().offset(index as isize));
            ptr::copy(self.ptr().offset(index as isize + 1),
                      self.ptr().offset(index as isize),
                      self.len - index);
            result
        }
    }

    pub fn into_iter(self) -> IntoIter<T> {
        unsafe {
            let iter = RawValIter::new(&self);
            let buf = ptr::read(&self.buf);
            mem::forget(self);

            IntoIter {
                iter: iter,
                _buf: buf,
            }
        }
    }

    pub fn drain(&mut self) -> Drain<T> {
        unsafe {
            let iter = RawValIter::new(&self);

            // this is a mem::forget safety thing. If Drain is forgotten, we just
            // leak the whole Vec's contents. Also we need to do this *eventually*
            // anyway, so why not do it now?
            self.len = 0;

            Drain {
                iter: iter,
                vec: PhantomData,
            }
        }
    }
}

impl<T> Drop for Vec<T> {
    fn drop(&mut self) {
        while let Some(_) = self.pop() {}
        // allocation is handled by RawVec
    }
}

impl<T> Deref for Vec<T> {
    type Target = [T];
    fn deref(&self) -> &[T] {
        unsafe {
            ::std::slice::from_raw_parts(self.ptr(), self.len)
        }
    }
}

impl<T> DerefMut for Vec<T> {
    fn deref_mut(&mut self) -> &mut [T] {
        unsafe {
            ::std::slice::from_raw_parts_mut(self.ptr(), self.len)
        }
    }
}
```