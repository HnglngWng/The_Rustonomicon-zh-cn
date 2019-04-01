# Rust中的数据表示(Data Representation in Rust)

低级编程关注数据布局.这是一个大问题.它也普遍影响语言的其余部分,因此我们将首先深入研究在Rust中数据是如何表示的.

理想情况下,本章与[Reference的类型布局部分](https://github.com/rust-lang-nursery/nomicon/blob/master/reference/type-layout.html)是一致的,并且是冗余的. 本书第一次编写时,reference完全失修,Rustonomicon试图作为reference的部分替代品. 现在不再是这种情况,所以理想情况下可以删除整章.
我们将本章保留一段时间,但理想情况下,你应该为Reference贡献任何新的事实或改进.