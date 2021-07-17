+++
title = "A Half Hour to Learn Rust"
date = 2020-03-10T07:18:06+08:00
images = []
tags = ["transalte"]
categories = ["rust"]
draft = false
+++

> 原文[A half hour to learm rust](https://fasterthanli.me/blog/2020/a-half-hour-to-learn-rust/) 

为了提高编程语言的流利程度，必须阅读很多代码。 但是，如果逆不知道代码的含义，该怎么读呢？

在本文中，不专注于一个或两个概念，我将尝试通过尽可能多的Rust代码，解释其中关键字和符号包含的含义。

准备好了么？开始吧！

让我们介绍下变量绑定:

```rust
let x; // 声明 "x"
x = 42; // 将42赋值给"x"
```
当然这可以写在一行:

```rust
let x = 42;
```
您可以使用`：`显式指定变量的类型，即类型注释:

```rust
let x: i32; // `i32` 是有符号32位整型
x = 42;

// 还有 i8, i16, i32, i64, i128
//  同样的还有表示无符号整型的 u8, u16, u32, u64, u128 
```
同样的，可以写成一行：

```rust
let x: i32 = 42;
```

如果你声明一个变量并在使用之后才对它初始化，编译器会对使用前未初始化的变量报错。

```rust
let x;
foobar(x); // error: borrow of possibly-uninitialized variable: `x`
x = 42;
```

但是，这样做完全可以：

```rust
let x;
x = 42;
foobar(x); // the type of `x` will be inferred from here
```

下划线`_` 是一个特殊名称的变量，或者说，是匿名变量。这基本意味着不再使用，无意义的值：

```rust
// 这什么含义都没有，因为42是个常数
let _ = 42;

// 这是调用了一个函数，但是不关心或者不需要返回值
let _ = get_thing();
```

用下划线开始的变量名是合法的，需要注意的是编译器对它们未使用不会警告：

```rust
// 我们最终可能会使用`_x`，但是我们的代码仍在开发中
// 我们现在只是想摆脱编译器警告。
let _x = 42;
```

可以重复声明相同标识符的变量，只是后面的会覆盖掉前面的变量：

```rust
let x = 13;
let x = x + 3;
// 在这行后使用`x`引用的是第二个`x`，第一个 `x` 将不存在
```

Rust has tuples, which you can think of as “fixed-length collections of values of different types”.

```rust
let pair = ('a', 17);
pair.0; // this is 'a'
pair.1; // this is 17
```

If we really we wanted to annotate the type of `pair`, we would write:

```rust
let pair: (char, i32) = ('a', 17);
```

Tuples can be *destructured* when doing an assignment, which means they're broken down into their individual fields:

```rust
let (some_char, some_int) = ('a', 17);
// now, `some_char` is 'a', and `some_int` is 17
```

This is especially useful when a function returns a tuple:

```rust
let (left, right) = slice.split_at(middle);
```

Of course, when destructuring a tuple, `_` can be used to throw away part of it:

```rust
let (_, right) = slice.split_at(middle);
```

The semi-colon marks the end of a statement:

```rust
let x = 3;
let y = 5;
let z = y + x;
```

Which means statements can span multiple lines:

```rust
let x = vec![1, 2, 3, 4, 5, 6, 7, 8]
    .iter()
    .map(|x| x + 3)
    .fold(0, |x, y| x + y);
```

(We'll go over what those actually mean later).

`fn` declares a function.

Here's a void function:

```rust
fn greet() {
    println!("Hi there!");
}
```

And here's a function that returns a 32-bit signed integer. The arrow indicates its return type:

```rust
fn fair_dice_roll() -> i32 {
    4
}
```

A pair of brackets declares a block, which has its own scope:

```rust
// This prints "in", then "out"
fn main() {
    let x = "out";
    {
        // this is a different `x`
        let x = "in";
        println!(x);
    }
    println!(x);
}
```

Blocks are also expressions, which mean they evaluate to.. a value.

```rust
// this:
let x = 42;

// is equivalent to this:
let x = { 42 };
```

Inside a block, there can be multiple statements:

```rust
let x = {
    let y = 1; // first statement
    let z = 2; // second statement
    y + z // this is the *tail* - what the whole block will evaluate to
};
```

And that's why “omitting the semicolon at the end of a function” is the same as returning, ie. these are equivalent:

```rust
fn fair_dice_roll() -> i32 {
    return 4;
}

fn fair_dice_roll() -> i32 {
    4
}
```

`if` conditionals are also expressions:

```rust
fn fair_dice_roll() -> i32 {
    if feeling_lucky {
        6
    } else {
        4
    }
}
```

A `match` is also an expression:

```rust
fn fair_dice_roll() -> i32 {
    match feeling_lucky {
        true => 6,
        false => 4,
    }
}
```

Dots are typically used to access fields of a value:

```rust
let a = (10, 20);
a.0; // this is 10
```

Or call a method on a value:

```rust
let nick = "fasterthanlime";
nick.len(); // this is 14
```

The double-colon, `::`, is similar but it operates on namespaces.

In this example, `std` is a *crate* (~ a library), `cmp` is a *module* (~ a source file), and `min` is a *function*:

```rust
let least = std::cmp::min(3, 8); // this is 3
```

`use` directives can be used to “bring in scope” names from other namespace:

```rust
use std::cmp::min;

let least = min(7, 1); // this is 1
```

Within `use` directives, curly brackets have another meaning: they're “globs”. If we want to import both `min` and `max`, we can do any of these:

```rust
// this works:
use std::cmp::min;
use std::cmp::max;

// this also works:
use std::cmp::{min, max};

// this also works!
use std::{cmp::min, cmp::max};
```

A wildcard (`*`) lets you import every symbol from a namespace:

```rust
// this brings `min` and `max` in scope, and many other things
use std::cmp::*;
```

Types are namespaces too, and methods can be called as regular functions:

```rust
let x = "amos".len(); // this is 4
let x = str::len("amos"); // this is also 4
```

`str` is a primitive type, but many non-primitive types are also in scope by default.

```rust
// `Vec` is a regular struct, not a primitive type
let v = Vec::new();

// this is exactly the same code, but with the *full* path to `Vec`
let v = std::vec::Vec::new();
```

This works because Rust inserts this at the beginning of every module:

```rust
use std::prelude::v1::*;
```

(Which in turns re-exports a lot of symbols, like `Vec`, `String`, `Option` and `Result`).

Structs are declared with the `struct` keyword:

```rust
struct Vec2 {
    x: f64, // 64-bit floating point, aka "double precision"
    y: f64,
}
```

They can be initialized using *struct literals*:

```rust
let v1 = Vec2 { x: 1.0, y: 3.0 };
let v2 = Vec2 { y: 2.0, x: 4.0 };
// the order does not matter, only the names do
```

There is a shortcut for initializing the *rest of the fields* from another struct:

```rust
let v3 = Vec2 {
    x: 14.0,
    ..v2
};
```

This is called “struct update syntax”, can only happen in last position, and cannot be followed by a comma.

Note that *the rest of the fields* can mean *all the fields*:

```rust
let v4 = Vec2 { ..v3 };
```

Structs, like tuples, can be destructured.

Just like this is a valid `let` pattern:

```rust
let (left, right) = slice.split_at(middle);
```

So is this:

```rust
let v = Vec2 { x: 3.0, y: 6.0 };
let Vec2 { x, y } = v;
// `x` is now 3.0, `y` is now `6.0`
```

And this:

```rust
let Vec2 { x, .. } = v;
// this throws away `v.y`
```

`let` patterns can be used as conditions in `if`:

```rust
struct Number {
    odd: bool,
    value: i32,
}

fn main() {
    let one = Number { odd: true, value: 1 };
    let two = Number { odd: false, value: 2 };
    print_number(one);
    print_number(two);
}

fn print_number(n: Number) {
    if let Number { odd: true, value } = n {
        println!("Odd number: {}", value);
    } else if let Number { odd: false, value } = n {
        println!("Even number: {}", value);
    }
}

// this prints:
// Odd number: 1
// Even number: 2
```

`match` arms are also patterns, just like `if let`:

```rust
fn print_number(n: Number) {
    match n {
        Number { odd: true, value } => println!("Odd number: {}", value),
        Number { odd: false, value } => println!("Even number: {}", value),
    }
}

// this prints the same as before
```

A `match` has to be exhaustive: at least one arm needs to match.

```rust
fn print_number(n: Number) {
    match n {
        Number { value: 1, .. } => println!("One"),
        Number { value: 2, .. } => println!("Two"),
        Number { value, .. } => println!("{}", value),
        // if that last arm didn't exist, we would get a compile-time error
    }
}
```

If that's hard, `_` can be used as a “catch-all” pattern:

```rust
fn print_number(n: Number) {
    match n.value {
        1 => println!("One"),
        2 => println!("Two"),
        _ => println!("{}", n.value),
    }
}
```

You can declare methods on your own types:

```rust
struct Number {
    odd: bool,
    value: i32,
}

impl Number {
    fn is_strictly_positive(self) -> bool {
        self.value > 0
    }
}
```

And use them like usual:

```rust
fn main() {
    let minus_two = Number {
        odd: false,
        value: -2,
    };
    println!("positive? {}", minus_two.is_strictly_positive());
    // this prints "positive? false"
}
```

Variable bindings are immutable by default, which means their interior can't be mutated:

```rust
fn main() {
    let n = Number {
        odd: true,
        value: 17,
    };
    n.odd = false; // error: cannot assign to `n.odd`,
                   // as `n` is not declared to be mutable
}
```

And also that they cannot be assigned to:

```rust
fn main() {
    let n = Number {
        odd: true,
        value: 17,
    };
    n = Number {
        odd: false,
        value: 22,
    }; // error: cannot assign twice to immutable variable `n`
}
```

`mut` makes a variable binding mutable:

```rust
fn main() {
    let mut n = Number {
        odd: true,
        value: 17,
    }
    n.value = 19; // all good
}
```

Traits are something multiple types can have in common:

```rust
trait Signed {
    fn is_strictly_negative(self) -> bool;
}
```

You can implement:

- one of your traits on anyone's type
- anyone's trait on one of your types
- but not a foreign trait on a foreign type

These are called the “orphan rules”.

Here's an implementation of our trait on our type:

```rust
impl Signed for Number {
    fn is_strictly_negative(self) -> bool {
        self.value < 0
    }
}

fn main() {
    let n = Number { odd: false, value: -44 };
    println!("{}", n.is_strictly_negative()); // prints "true"
}
```

Our trait on a foreign type (a primitive type, even):

```rust
impl Signed for i32 {
    fn is_strictly_negative(self) -> bool {
        self < 0
    }
}

fn main() {
    let n: i32 = -44;
    println!("{}", n.is_strictly_negative()); // prints "true"
}
```

A foreign trait on our type:

```rust
// the `Neg` trait is used to overload `-`, the
// unary minus operator.
impl std::ops::Neg for Number {
    type Output = Number;

    fn neg(self) -> Number {
        Number {
            value: -self.value,
            odd: self.odd,
        }        
    }
}

fn main() {
    let n = Number { odd: true, value: 987 };
    let m = -n; // this is only possible because we implemented `Neg`
    println!("{}", m.value); // prints "-987"
}
```

An `impl` block is always *for* a type, so, inside that block, `Self` means that type:

```rust
impl std::ops::Neg for Number {
    type Output = Self;

    fn neg(self) -> Self {
        Self {
            value: -self.value,
            odd: self.odd,
        }        
    }
}
```

Some traits are *markers* - they don't say that a type implements some methods, they say that certain things can be done with a type.

For example, `i32` implements trait `Copy` (in short, `i32` *is* `Copy`), so this works:

```rust
fn main() {
    let a: i32 = 15;
    let b = a; // `a` is copied
    let c = a; // `a` is copied again
}
```

And this also works:

```rust
fn print_i32(x: i32) {
    println!("x = {}", x);
}

fn main() {
    let a: i32 = 15;
    print_i32(a); // `a` is copied
    print_i32(a); // `a` is copied again
}
```

But the `Number` struct is not `Copy`, so this doesn't work:

```rust
fn main() {
    let n = Number { odd: true, value: 51 };
    let m = n; // `n` is moved into `m`
    let o = n; // error: use of moved value: `n`
}
```

And neither does this:

```rust
fn print_number(n: Number) {
    println!("{} number {}", if n.odd { "odd" } else { "even" }, n.value);
}

fn main() {
    let n = Number { odd: true, value: 51 };
    print_number(n); // `n` is moved
    print_number(n); // error: use of moved value: `n`
}
```

But it works if `print_number` takes an immutable reference instead:

```rust
fn print_number(n: &Number) {
    println!("{} number {}", if n.odd { "odd" } else { "even" }, n.value);
}

fn main() {
    let n = Number { odd: true, value: 51 };
    print_number(&n); // `n` is borrowed for the time of the call
    print_number(&n); // `n` is borrowed again
}
```

It also works if a function takes a *mutable* reference - but only if our variable binding is also `mut`.

```rust
fn invert(n: &mut Number) {
    n.value = -n.value;
}

fn print_number(n: &Number) {
    println!("{} number {}", if n.odd { "odd" } else { "even" }, n.value);
}

fn main() {
    // this time, `n` is mutable
    let mut n = Number { odd: true, value: 51 };
    print_number(&n);
    invert(&mut n); // `n is borrowed mutably - everything is explicit
    print_number(&n);
}
```

Trait methods can also take `self` by reference or mutable reference:

```rust
impl std::clone::Clone for Number {
    fn clone(&self) -> Self {
        Self { ..*self }
    }
}
```

When invoking trait methods, the receiver is borrowed implicitly:

```rust
fn main() {
    let n = Number { odd: true, value: 51 };
    let mut m = n.clone();
    m.value += 100;
    
    print_number(&n);
    print_number(&m);
}
```

To highlight this: these are equivalent:

```rust
let m = n.clone();

let m = std::clone::Clone::clone(&n);
```

Marker traits like `Copy` have no methods:

```rust
// note: `Copy` requires that `Clone` is implemented too
impl std::clone::Clone for Number {
    fn clone(&self) -> Self {
        Self { ..*self }
    }
}

impl std::marker::Copy for Number {}
```

Now, `Clone` can still be used:

```rust
fn main() {
    let n = Number { odd: true, value: 51 };
    let m = n.clone();
    let o = n.clone();
}
```

But `Number` values will no longer be moved:

```rust
fn main() {
    let n = Number { odd: true, value: 51 };
    let m = n; // `m` is a copy of `n`
    let o = n; // same. `n` is neither moved nor borrowed.
}
```

Some traits are so common, they can be implemented automatically by using the `derive` attribute:

```rust
#[derive(Clone, Copy)]
struct Number {
    odd: bool,
    value: i32,
}

// this expands to `impl Clone for Number` and `impl Copy for Number` blocks.
```

Functions can be generic:

```rust
fn foobar<T>(arg: T) {
    // do something with `arg`
}
```

They can have multiple *type parameters*, which can then be used in the function's declaration and its body, instead of concrete types:

```rust
fn foobar<L, R>(left: L, right: R) {
    // do something with `left` and `right`
}
```

Type parameters usually have *constraints*, so you can actually do something with them.

The simplest constraints are just trait names:

```rust
fn print<T: Display>(value: T) {
    println!("value = {}", value);
}

fn print<T: Debug>(value: T) {
    println!("value = {:?}", value);
}
```

There's a longer syntax for type parameter constraints:

```rust
fn print<T>(value: T)
where
    T: Display,
{
    println!("value = {}", value);
}
```

Constraints can be more complicated: they can require a type parameter to implement multiple traits:

```rust
use std::fmt::Debug;

fn compare<T>(left: T, right: T)
where
    T: Debug + PartialEq,
{
    println!("{:?} {} {:?}", left, if left == right { "==" } else { "!=" }, right);
}

fn main() {
    compare("tea", "coffee");
    // prints: "tea" != "coffee"
}
```

Generic functions can be thought of as namespaces, containing an infinity of functions with different concrete types.

Same as with crates, and modules, and types, generic functions can be “explored” (navigated?) using `::`

```rust
fn main() {
    use std::any::type_name;
    println!("{}", type_name::<i32>()); // prints "i32"
    println!("{}", type_name::<(f64, char)>()); // prints "(f64, char)"
}
```

This is lovingly called [turbofish syntax](https://turbo.fish/), because `::<>` looks like a fish.

Structs can be generic too:

```rust
struct Pair<T> {
    a: T,
    b: T,
}

fn print_type_name<T>(_val: &T) {
    println!("{}", std::any::type_name::<T>());
}

fn main() {
    let p1 = Pair { a: 3, b: 9 };
    let p2 = Pair { a: true, b: false };
    print_type_name(&p1); // prints "Pair<i32>"
    print_type_name(&p2); // prints "Pair<bool>"
}
```

The standard library type `Vec` (~ a heap-allocated array), is generic:

```rust
fn main() {
    let mut v1 = Vec::new();
    v1.push(1);
    let mut v2 = Vec::new();
    v2.push(false);
    print_type_name(&v1); // prints "Vec<i32>"
    print_type_name(&v2); // prints "Vec<bool>"
}
```

Speaking of `Vec`, it comes with a macro that gives more or less “vec literals”:

```rust
fn main() {
    let v1 = vec![1, 2, 3];
    let v2 = vec![true, false, true];
    print_type_name(&v1); // prints "Vec<i32>"
    print_type_name(&v2); // prints "Vec<bool>"
}
```

All of `name!()`, `name![]` or `name!{}` invoke a macro. Macros just expand to regular code.

In fact, `println` is a macro:

```rust
fn main() {
    println!("{}", "Hello there!");
}
```

