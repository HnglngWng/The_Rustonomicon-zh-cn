# 处理零大小类型(Handling Zero-Sized Types)

是时候了.我们要打击零大小的幽灵.Safe Rust *从不(never)* 需要关心这一点,但Vec非常关注原始指针和原始分配,这正是关注零大小类型的两件事.我们需要注意两件事:

- 如果为分配大小传入0,则原始分配器API具有未定义的行为.

- 原始指针偏移对于零大小类型是无操作(no-ops),这将破坏我们的C风格指针迭代器.

值得庆幸的是,我们抽象出指针迭代器并分别将处理分配到`RawValIter`和`RawVec`.多么神秘方便.

## 分配零大小的类型(Allocating Zero-Sized Types)

因此,如果分配器API不支持零大小的分配,那么我们将分配哪些内容作为分配?`Unique::dangling()`当然!几乎每个具有ZST的操作都是无操作,因为ZST只有一个值,因此不需要考虑存储或加载它们的状态.这实际上扩展到`ptr::read`和`ptr::write`:它们实际上根本不会查看指针.因此,我们永远不需要更改指针.

但请注意,我们之前对溢出前内存不足的依赖不再适用于零大小的类型.我们必须显式防范零大小类型的容量溢出.

由于我们目前的架构,所有这些都意味着编写3个守卫(guards),每个`RawVec`方法一个.

```Rust
impl<T> RawVec<T> {
    fn new() -> Self {
        // !0 is usize::MAX. This branch should be stripped at compile time.
        let cap = if mem::size_of::<T>() == 0 { !0 } else { 0 };

        // Unique::dangling() doubles as "unallocated" and "zero-sized allocation"
        RawVec { ptr: Unique::dangling(), cap: cap }
    }

    fn grow(&mut self) {
        unsafe {
            let elem_size = mem::size_of::<T>();

            // since we set the capacity to usize::MAX when elem_size is
            // 0, getting to here necessarily means the Vec is overfull.
            assert!(elem_size != 0, "capacity overflow");

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
        let elem_size = mem::size_of::<T>();

        // don't free zero-sized allocations, as they were never allocated.
        if self.cap != 0 && elem_size != 0 {
            unsafe {
                let c: NonNull<T> = self.ptr.into();
                Global.deallocate(c.cast(),
                                  Layout::array::<T>(self.cap).unwrap());
            }
        }
    }
}
```

仅此而已.我们现在支持推入和弹出零大小的类型.但是,我们的迭代器(切片Deref不提供)仍然被破坏.

## 迭代零大小的类型(Iterating Zero-Sized Types)

零大小的偏移是无操作(no-ops).这意味着我们当前的设计将始终将`start`和`end`初始化为相同的值,并且我们的迭代器将不会产生任何结果.当前的解决方案是将指针转换为整数,递增,然后将它们转换回来:

```Rust
impl<T> RawValIter<T> {
    unsafe fn new(slice: &[T]) -> Self {
        RawValIter {
            start: slice.as_ptr(),
            end: if mem::size_of::<T>() == 0 {
                ((slice.as_ptr() as usize) + slice.len()) as *const _
            } else if slice.len() == 0 {
                slice.as_ptr()
            } else {
                slice.as_ptr().offset(slice.len() as isize)
            },
        }
    }
}
```

现在我们有一个不同的bug.我们的迭代器现在 *永远(forever)* 运行,而不是我们的迭代器根本不运行.我们需要在迭代器中使用相同的技巧.此外,对于ZST,我们的size_hint计算代码将除以0.因为我们基本上将两个指针视为指向字节,所以我们只是将大小0映射为除以1.

```Rust
impl<T> Iterator for RawValIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                let result = ptr::read(self.start);
                self.start = if mem::size_of::<T>() == 0 {
                    (self.start as usize + 1) as *const _
                } else {
                    self.start.offset(1)
                };
                Some(result)
            }
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        let elem_size = mem::size_of::<T>();
        let len = (self.end as usize - self.start as usize)
                  / if elem_size == 0 { 1 } else { elem_size };
        (len, Some(len))
    }
}

impl<T> DoubleEndedIterator for RawValIter<T> {
    fn next_back(&mut self) -> Option<T> {
        if self.start == self.end {
            None
        } else {
            unsafe {
                self.end = if mem::size_of::<T>() == 0 {
                    (self.end as usize - 1) as *const _
                } else {
                    self.end.offset(-1)
                };
                Some(ptr::read(self.end))
            }
        }
    }
}
```

就是这样.迭代工作!
