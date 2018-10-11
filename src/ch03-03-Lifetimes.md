# 生命周期(Lifetimes)

Rust整个 *生命周期(lifetimes)* 都会执行这些规则.生命周期实际上只是程序中某处作用域的名称.每个引用以及包含引用的任何内容都使用指定其有效作用域的生命周期进行标记.

在函数体中,Rust通常不会让你显式命名所涉及的生命周期.这是因为通常没有必要在局部上下文中谈论生命周期;Rust拥有所有信息,可以尽可能优化地处理所有事情.你经常会引入许多你必须编写的匿名作用域和临时作用域,以使你的代码正常工作.

但是,一旦跨越函数边界,就需要开始讨论生命周期.生命周期用撇号表示:`'a'`,`static`.为了试探生命周期,我们将假装我们实际上允许用生命周期来标记作用域,脱糖(desugar)从本章开始的例子.

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

然后我们继续计算字符串`s`,并返回对它的引用.由于函数的契约表明引用必须比`'a'`活得更长,这是我们为引用推断的生命周期.不幸的是,`s`是在作用域`'b`中定义的,所以唯一合理的方式就是,如果`'b`包含`'a`--这显然是假的,因为`'a`必须包含函数调用本身.因此,我们创建了一个引用,它的生命周期比它的引用对象要长,这就是我们说的引用不能做的第一件事.编译器理所当然地在我们面前爆炸了.

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

"糟糕!"

当然,编写此函数的正确方法如下:

```Rust
fn to_string(data: &u32) -> String {
    format!("{}", data)
}
```