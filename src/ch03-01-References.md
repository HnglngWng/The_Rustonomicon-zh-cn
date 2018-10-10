# 引用(References)

有两种引用:

- 共享引用:&

- 可变引用:&mut

遵守以下规则:

- 引用不能活得超过它的引用对象

- 可变引用不能别名

仅此而已.这是引用遵循的整个模型.

当然,我们应该定义 *别名(aliased)* 的含义.

```Rust
error[E0425]: cannot find value `aliased` in this scope
 --> <rust.rs>:2:20
  |
2 |     println!("{}", aliased);
  |                    ^^^^^^^ not found in this scope

error: aborting due to previous error
```

不幸的是,Rust实际上没有定义其别名模型.🙀

当我们等待Rust开发人员指定他们语言的语义时,让我们使用下一节来讨论一般的别名,以及它为何重要.