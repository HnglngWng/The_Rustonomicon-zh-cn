# 强制转换(Coercions)

在某些情况下,类型可以被隐含地强制改变.这些变化通常只会 *削弱(weakening)* 类型,主要集中在指针和生命周期.它们主要用于使Rust在更多情况下"正常工作(just work)",并且在很大程度上是无害的.

这是所有种类的强制转换:

以下类型之间允许强制转换:

- 传递性(Transitivity):`T_1`到`T_3`,其中`T_1`强制转换为`T_2`,`T_2`强制转换为`T_3`

- 指针弱化(Pointer Weakening):

  - `&mut T`到`&T`
  
  - `*mut T`到`*const T`
  
  - `&T`到`*const T`
  
  - `&mut T`到`*mut T`

- 无大小(Unsizing):如果`T`实现`CoerceUnsized<U>`,则`T`到`U`

- 解引用强制(Deref coercion):如果`T`解引用为`U`(即`T: Deref<Target=U>`),则类型`&T`的表达式`&x`到类型`&U`的表达式`&*x`

`CoerceUnsized<Pointer<U>> for Pointer<T> where T: Unsize<U>`是针对所有指针类型实现的(包括智能指针,如Box和Rc).Unsize仅自动实现,并启用以下转换:

- `[T; n]` => `[T]`

- `T` => `dyn Trait` 其中 `T: Trait`

- `Foo<..., T, ...>` => `Foo<..., U, ...>`其中:

  - `T: Unsize<U>`
  
  - `Foo`是一个结构
  
  - 只有`Foo`的最后一个字段有涉及`T`的类型
  
  - `T`不是任何其他字段的类型的一部分
  
  - `Bar<T>: Unsize<Bar<U>>`,如果`Foo`的最后一个字段的类型为`Bar<T>`

强制转换发生在 *强制地点(coercion site)* .显式键入的任何位置都会导致对其类型的强制转换.如果需要进行推断,则不会进行强制转换.详尽地说,表达式`e`到类型`U`的强制地点是:

- let语句,静态和常量:`let x: U = e`

- 函数参数:`takes_a_U(e)`

- 将返回的任何表达式:`fn foo() -> U { e }`

- 结构字面量:`Foo { some_u: e}`

- 数组字面量:`let x: [U; 10] = [e, ..]`

- 元组字面量:`let x: (U, ..) = (e, ..)`

- 块中的最后一个表达式:`let x: U = { ..; e }`

请注意,匹配traits时不执行强制转换(接收者(receivers)除外,见下文).如果有某种类型`U`的impl并且`T`强制转为`U`,那么这不构成`T`的实现.例如,以下将不会通过类型检查,即使`t`可以强制转换为`&T`并且存在`&T`的impl:

```Rust
trait Trait {}

fn foo<X: Trait>(t: X) {}

impl<'a> Trait for &'a i32 {}

fn main() {
    let t: &mut i32 = &mut 0;
    foo(t);
}
```

```Rust
error[E0277]: the trait bound `&mut i32: Trait` is not satisfied
 --> src/main.rs:9:5
  |
9 |     foo(t);
  |     ^^^ the trait `Trait` is not implemented for `&mut i32`
  |
  = help: the following implementations were found:
            <&'a i32 as Trait>
note: required by `foo`
 --> src/main.rs:3:1
  |
3 | fn foo<X: Trait>(t: X) {}
  | ^^^^^^^^^^^^^^^^^^^^^^

error: aborting due to previous error
```
