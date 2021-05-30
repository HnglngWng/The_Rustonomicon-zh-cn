# 异常安全(Exception Safety)

虽然程序应该谨慎使用展开(unwinding),但是有很多代码 *可能会(can)* 恐慌.如果你打开(unwrap)一个None,索引越界,或除以0,你的程序将会恐慌.在调试版本中,如果溢出,每个算术运算都会发生混乱.除非你非常小心并严格控制运行的代码,否则几乎所有东西都可以展开,你需要做好准备.

在更广泛的编程世界中,随时准备展开通常被称为 *异常安全(exception safety)* .在Rust中,有两个级别的异常安全性,人们可能会关注它们:

- 在不安全代码中,我们 *必须(must)* 异常安全,以免违反内存安全.我们称之为 *最小的(minimal)* 异常安全性.

- 在安全的代码中,对于程序做正确的事情来说, *最好(good)* 是异常安全.我们称之为 *最大(maximal)* 异常安全性.

与Rust中许多地方的情况一样,不安全代码必须准备好在展开时处理不好的安全代码.暂时创建不健全状态的代码必须小心,恐慌不会导致使用该状态.通常,这意味着确保在这些状态存在时仅运行非恐慌代码,或者在恐慌的情况下使守卫清理状态.这并不一定意味着,恐慌作证的状态是一个完全连贯的状态.我们只需保证它是一个 *安全的(safe)* 状态.

大多数不安全代码都是叶状的(leaf-like),因此很容易使之异常安全.它控制所有运行的代码,大多数代码都不会出现恐慌.但是,在重复调用调用者提供的(caller-provided)代码时,不安全代码使用临时未初始化数据的数组并不罕见.这样的代码需要小心并考虑异常安全性.

## Vec::push_all(Vec::push_all)

`Vec::push_all`是一个临时的技术(hack),可以通过切片可靠地高效扩展Vec而无需特化.这是一个简单的实现:

```Rust
impl<T: Clone> Vec<T> {
    fn push_all(&mut self, to_push: &[T]) {
        self.reserve(to_push.len());
        unsafe {
            // can't overflow because we just reserved this
            self.set_len(self.len() + to_push.len());

            for (i, x) in to_push.iter().enumerate() {
                self.ptr().add(i).write(x.clone());
            }
        }
    }
}
```

我们绕过`push`以避免冗余容量和对Vec的`len`检查,我们肯定知道它有容量.逻辑是完全正确的,只是我们的代码有一个微妙的问题:它不是异常安全的!`set_len`,`add`和`write`都很好;`clone`是我们忽视的(over-looked)恐慌炸弹.

克隆完全不受我们控制,完全可以自由恐慌.如果是这样,我们的函数将提前退出,Vec的长度设置得太大.如果查看或删除Vec,将读取未初始化的内存!

这种情况下的修复非常简单.如果我们想保证 *确实是* 我们克隆的值被删除,我们可以设置`len`每次循环迭代.如果我们只想保证无法观察到未初始化的内存,我们可以在循环后设置`len`.

## BinaryHeap::sift_up(BinaryHeap::sift_up)

将元素冒泡到堆中比扩展Vec要复杂一些.伪代码如下:

```Rust (pseudocode)
bubble_up(heap, index):
    while index != 0 && heap[index] < heap[parent(index)]:
        heap.swap(index, parent(index))
        index = parent(index)
```

这段代码逐字转录到Rust完全没问题,但是有一个恼人的性能特征:`self`元素一遍又一遍地无用地交换.我们宁愿拥有以下内容:

```Rust (pseudocode)
bubble_up(heap, index):
    let elem = heap[index]
    while index != 0 && elem < heap[parent(index)]:
        heap[index] = heap[parent(index)]
        index = parent(index)
    heap[index] = elem
```

此代码确保尽可能少地复制每个元素(实际上有必要将elem一般复制两次).但它现在暴露了一些异常安全问题!在任何时候,都存在一个值的两个副本.如果我们在这个函数中发生恐慌,那么将会出现双重删除(double-dropped)问题.不幸的是,我们也没有完全控制代码:比较是用户定义的!

与Vec不同,这里的修复并不容易.一种选择是将用户定义的代码和不安全的代码分成两个独立的阶段:

```Rust (pseudocode)
bubble_up(heap, index):
    let end_index = index;
    while end_index != 0 && heap[end_index] < heap[parent(end_index)]:
        end_index = parent(end_index)

    let elem = heap[index]
    while index != end_index:
        heap[index] = heap[parent(index)]
        index = parent(index)
    heap[index] = elem
```

如果用户定义的代码爆炸,那就不再是问题,因为我们还没有真正触及堆的状态.一旦我们开始搞乱堆，我们只处理我们信任的数据和函数,所以不用担心恐慌.

也许你对这个设计不满意.肯定是作弊!我们必须做 *两次(twice)* 复杂的堆遍历!好吧,让我们咬紧牙关.让我们 *真的(for reals)* 混合不可信和不安全的代码。

如果Rust有像Java中一样的`try`和`finally`,我们可以执行以下操作:

```Rust (pseudocode)
bubble_up(heap, index):
    let elem = heap[index]
    try:
        while index != 0 && elem < heap[parent(index)]:
            heap[index] = heap[parent(index)]
            index = parent(index)
    finally:
        heap[index] = elem
```

基本思路很简单:如果比较恐慌,我们就将松散的元素扔在逻辑上未初始化的索引中并挽救.任何观察堆的人都会看到一个可能 *不一致的(inconsistent)* 堆,但至少它不会导致任何双重删除!如果算法正常终止,则此操作恰好与我们如何完成的方式完全一致.

可悲的是,Rust没有这样的构造函数,所以我们需要自己动手!这样做的方法是将算法的状态存储在一个单独的结构中,其中包含"finally"逻辑的析构函数.无论我们是否恐慌,析构函数都会在我们之后运行和清理.

```Rust
struct Hole<'a, T: 'a> {
    data: &'a mut [T],
    /// `elt` is always `Some` from new until drop.
    elt: Option<T>,
    pos: usize,
}

impl<'a, T> Hole<'a, T> {
    fn new(data: &'a mut [T], pos: usize) -> Self {
        unsafe {
            let elt = ptr::read(&data[pos]);
            Hole {
                data: data,
                elt: Some(elt),
                pos: pos,
            }
        }
    }

    fn pos(&self) -> usize { self.pos }

    fn removed(&self) -> &T { self.elt.as_ref().unwrap() }

    unsafe fn get(&self, index: usize) -> &T { &self.data[index] }

    unsafe fn move_to(&mut self, index: usize) {
        let index_ptr: *const _ = &self.data[index];
        let hole_ptr = &mut self.data[self.pos];
        ptr::copy_nonoverlapping(index_ptr, hole_ptr, 1);
        self.pos = index;
    }
}

impl<'a, T> Drop for Hole<'a, T> {
    fn drop(&mut self) {
        // fill the hole again
        unsafe {
            let pos = self.pos;
            ptr::write(&mut self.data[pos], self.elt.take().unwrap());
        }
    }
}

impl<T: Ord> BinaryHeap<T> {
    fn sift_up(&mut self, pos: usize) {
        unsafe {
            // Take out the value at `pos` and create a hole.
            let mut hole = Hole::new(&mut self.data, pos);

            while hole.pos() != 0 {
                let parent = parent(hole.pos());
                if hole.removed() <= hole.get(parent) { break }
                hole.move_to(parent);
            }
            // Hole will be unconditionally filled here; panic or not!
        }
    }
}
```
