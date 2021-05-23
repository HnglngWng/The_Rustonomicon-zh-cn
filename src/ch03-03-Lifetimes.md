# 生命周期(Lifetimes)

Rust整个 *生命周期(lifetimes)* 强制执行这些规则.生命周期是引用必须对其有效的代码的命名区域. 这些区域可能相当复杂,因为它们对应于程序中的执行路径. 这些执行路径中甚至可能存在漏洞,因为只要在重新使用引用之前对其进行初始化,就有可能使其无效. 包含引用(或假装为引用)的类型也可以用生命周期进行标记,这样Rust 也可以防止它们失效.

在我们的大多数示例中,生命周期将与作用域一致. 这是因为我们的示例很简单. 它们不一致的更为复杂的情况如下所述.

在函数体中,Rust通常不会让你显式命名所涉及的生命周期.这是因为通常没有必要在局部上下文中谈论生命周期;Rust拥有所有信息,可以尽可能优化地处理所有事情.你经常会引入许多你必须编写的匿名作用域和临时作用域,以使你的代码正常工作.

但是,一旦跨越函数边界,就需要开始讨论生命周期.生命周期用撇号表示:`'a`,`'static`.为了试探生命周期,我们将假装我们实际上允许用生命周期来标记作用域,脱糖(desugar)从本章开始的例子.

最初,我们的例子使用了 *激进的(aggressive)* 糖--甚至是高果糖玉米糖浆--对作用域和生命周期,因为显式地写出所以内容 *非常嘈杂(extremely noisy)* (语法噪音--译注).所有Rust代码都依赖于激进的推断和"明显"事物的省略.

一个特别有趣的糖是每个`let`语句隐式引入一个作用域.在大多数情况下,这并不重要.然而,它对于相互引用的变量很重要.举一个简单的例子,让我们完全脱糖这段简单的Rust代码:

```Rust
let x = 0;
let y = &x;
let z = &y;
```

借用检查器总是试图最小化生命周期的范围,因此可能会脱糖为以下结果:

```Rust
// NOTE: `'a: {` and `&'b x` is not valid syntax!
'a: {
    let x: i32 = 0;
    'b: {
        // lifetime used is 'b because that's good enough.
        let y: &'b i32 = &'b x;
        'c: {
            // ditto on 'c
            let z: &'c &'b i32 = &'c y;
        }
    }
}
```

哇.这...很糟糕.让我们花一点时间来感谢Rust让这更容易了.

实际上,将引用传递到外部作用域会导致Rust推断出更长的生命周期:

```Rust
let x = 0;
let z;
let y = &x;
z = y;
```

```Rust
'a: {
    let x: i32 = 0;
    'b: {
        let z: &'b i32;
        'c: {
            // Must use 'b here because this reference is
            // being passed to that scope.
            let y: &'b i32 = &'b x;
            z = y;
        }
    }
}
```

# 示例:引用活得比引用对象更久(Example: references that outlive referents)

好吧，让我们看看之前的一些例子:

```Rust
fn as_str(data: &u32) -> &str {
    let s = format!("{}", data);
    &s
}
```

脱糖为:

```Rust
fn as_str<'a>(data: &'a u32) -> &'a str {
    'b: {
        let s = format!("{}", data);
        return &'a s;
    }
}
```

`as_str`的签名接受具有 *一定(some)* 生命周期的对u32的引用,并承诺它可以生成对str的引用,它可以活得 *一样长(just as long)* .我们已经可以看到为什么这个签名可能会有麻烦.这基本上意味着我们将在作用域内的某个地方找到一个str,对u32的引用开始的或者 *更早的(even earlier)* 某个地方.这个要求有点高.

然后我们继续计算字符串`s`,并返回对它的引用.由于函数的契约表明引用必须比`'a'`活得更长,这是我们为引用推断的生命周期.不幸的是,`s`是在作用域`'b`中定义的,所以唯一合理的方式就是,如果`'b`包含`'a`--这显然是假的,因为`'a`必须包含函数调用本身.因此,我们创建了一个引用,它的生命周期比它的引用对象要长,这就是我们说的引用不能做的第一件事.编译器理所当然地是我们失败了.

为了更清楚地说明这一点,我们可以扩展示例:

```Rust
fn as_str<'a>(data: &'a u32) -> &'a str {
    'b: {
        let s = format!("{}", data);
        return &'a s
    }
}

fn main() {
    'c: {
        let x: u32 = 0;
        'd: {
            // An anonymous scope is introduced because the borrow does not
            // need to last for the whole scope x is valid for. The return
            // of as_str must find a str somewhere before this function
            // call. Obviously not happening.
            println!("{}", as_str::<'d>(&'d x));
        }
    }
}
```

糟糕!

当然,编写此函数的正确方法如下:

```Rust
fn to_string(data: &u32) -> String {
    format!("{}", data)
}
```

我们必须在函数内部生成一个拥有的值来返回它!我们唯一能够返回`&'a str`的方式就是如果它位于`&'a u32`的一个字段种,显然不是这样.

(实际上我们也可以只返回一个字符串字面量,作为全局可以认为它位于栈的底部;虽然这限制了我们的实现 *只是一点点(just a bit)* .)

# 示例:别名可变引用(Example: aliasing a mutable reference)

另一个例子:

```Rust
let mut data = vec![1, 2, 3];
let x = &data[0];
data.push(4);
println!("{}", x);
```

```Rust
'a: {
    let mut data: Vec<i32> = vec![1, 2, 3];
    'b: {
        // 'b is as big as we need this borrow to be
        // (just need to get to `println!`)
        let x: &'b i32 = Index::index::<'b>(&'b data, 0);
        'c: {
            // Temporary scope because we don't need the
            // &mut to last any longer.
            Vec::push(&'c mut data, 4);
        }
        println!("{}", x);
    }
}
```

这里的问题有点微妙和有趣.我们希望Rust拒绝此程序,原因如下:当我们尝试获取对`data`的可变引用以`push`时,我们有一个活着的对`data`的衍生物的共享引用`x`.这将创建一个别名的可变引用,这将违反引用的 *第二个(second)* 规则.

然而,这 *根本不是(not at all)* Rust认为这个程序是坏的的原因.Rust不理解`x`是对数据子路径的引用.它完全不了解`Vec`.它看到的是`x`必须活为`'b`,被打印出来.`Index::index`的签名随后要求我们对`data`的引用必须存活为`'b`.当我们尝试调用`push`时,它会看到我们尝试创建一个`&'c mut data`.Rust知道`'c`包含在`'b`中,并拒绝我们的程序,因为`&'b data`数据必须仍然活着!

在这里,我们看到生命周期系统比我们实际感兴趣的保留引用语义要粗糙得多.在大多数情况下, *这是完全可以的(that's totally ok)* ,因为它使我们不用整天向编译器解释我们的程序.然而,它确实意味着一些完全正确的与Rust的 *真正(true)* 语义相关的程序被拒绝,因为生命周期太过愚蠢.

# 生命周期所覆盖的区域(The area covered by a lifetime)

生命周期(有时称为 *借用(borrow)* )从它被创建的地方到最后一次使用是 *活的(alive)* . 借来的东西需要比借用活得久. 这看起来很简单,但是有一点点微妙之处.

以下代码片段可以编译,因为在打印`x`之后不再需要它,因此它是悬空的还是别名的都也没关系(即使变量`x` *技术上* 存在到作用域的最后).

```Rust
let mut data = vec![1, 2, 3];
let x = &data[0];
println!("{}", x);
// This is OK, x is no longer needed
data.push(4);
```

但是,如果该值有析构函数,则该析构函数将在作用域的末尾运行. 运行析构函数被认为是一次使用--显然是最后一次. 因此,这将 *不* 编译.

```Rust
#[derive(Debug)]
struct X<'a>(&'a i32);

impl Drop for X<'_> {
    fn drop(&mut self) {}
}

let mut data = vec![1, 2, 3];
let x = X(&data[0]);
println!("{:?}", x);
data.push(4);
// Here, the destructor is run and therefore this'll fail to compile.
```

一种使编译器确信`x`不再有效的方法是在`data.push(4)`之前使用`drop(x)`.

此外,借用可能有多种可能的最后使用,例如在条件的每个分支中.

```Rust
# fn some_condition() -> bool { true }
let mut data = vec![1, 2, 3];
let x = &data[0];

if some_condition() {
    println!("{}", x); // This is the last use of `x` in this branch
    data.push(4);      // So we can push here
} else {
    // There's no use of `x` in here, so effectively the last use is the
    // creation of x at the top of the example.
    data.push(5);
}
```

而生命周期在其中可能会有一个停顿. 或者,你可以把它看成是两个不同的借用,只是绑定在同一个局部变量上. 这通常在循环周围发生(在循环结束时写入变量的新值,并在下一次迭代的顶部最后一次使用它).

```Rust
let mut data = vec![1, 2, 3];
// This mut allows us to change where the reference points to
let mut x = &data[0];

println!("{}", x); // Last use of this borrow
data.push(4);
x = &data[3]; // We start a new borrow here
println!("{}", x);
```

从历史上看,Rust一直保持借用直到作用域结束,因此这些示例可能无法使用较旧的编译器进行编译. 此外,仍然有一些角落的情况,Rust无法适当地缩短借用的存活部分,即使它看起来应该也无法编译. 这些将随着时间的推移而解决.
