+++
title = 'Rust note'
date = 2021-10-17 15:26:57
+++
## 内存管理

Rust中没有垃圾回收机制，通过**所有权**和**借用**来保证数据的内存安全。

### 所有权(Ownership)

所有权系统在编译时会检查一组规则，当满足这些规则时才会通过编译，同时能保证程序运行的速度。

#### 作用域界定规则

Rust中作用域用大括号`{}`表示，无论是`if`、`else`还是`match`，或是直接的`{}`都表示一个作用域。

```rust
// `mascot` is not valid and cannot be used here, because it's not yet declared.
{
    let mascot = String::from("ferris");   // `mascot` is valid from this point forward.
    // do stuff with `mascot`.
}
// this scope is now over, so `mascot` is no longer valid and cannot be used.
println!("{}", mascot); // this will cause an error.
```

```
error[E0425]: cannot find value `mascot` in this scope
     --> src/main.rs:5:20
      |
    5 |     println!("{}", mascot);
      |                    ^^^^^^ not found in this scope
```

在超出作用域后，mascot会被删除，其堆空间中对应的String所拥有的内存也会被释放。

#### 移动语义

当一个变量赋值给另一个变量时，会发生**所有权转移**，原变量不再拥有它的所有权，如下例中mascot所拥有的String的所有权被转移到了ferris下，如果此时使用mascot将会报错。

```rust
{
    let mascot = String::from("ferris");
    let ferris = mascot;
    println!("{}", mascot) // This will cause an error.
}
```

当我们在某个作用域使用变量后，如果需要它再别的作用域继续使用，就可以交接所有权：

```rust
fn main() {
    let ferris; // declared outside.
    {
        let mascot = String::from("hello");
        println!("{}", mascot);
        ferris = mascot; // change ownership.
    }
    println!("{}", ferris) // use outside.
}

```

#### 函数中的所有权

将变量传递给函数时，也会进行所有权的交接，原变量将不再拥有数据，同时也不能把原变量再次传给函数：

```rust
fn process(input: String) {}

fn caller() {
    let s = String::from("Hello, world!");
    process(s); // Ownership of the string in `s` moved into `process`
    process(s); // Error! ownership already moved.
    println!("{}", s) // Error! ownership already moved.
}
```

对于实现了`Copy`特征的简单类型（如`u32`）会自动进行复制，上面的代码就可以正常进行。当然，我们也可以使用`clone()`函数将数据拷贝到函数中，但是如果数据类型比较复杂，成本就比较高昂。

### 借用(Borrow)

Rust中可以通过引用在别的地方使用变量，这点和C++类似：

```rust
let greeting = String::from("hello");
let greeting_reference = &greeting; // We borrow `greeting` but the string data is still owned by `greeting`
println!("Greeting: {}", greeting); // We can still use `greeting`
```

使用函数时，我们可以多次将引用传入函数，这样不会报错，但是它并不能被修改，因为传入普通的引用并没有那样的权限：

```rust
fn print_greeting(message: &String) {
  println!("Greeting: {}", message);
  message.push_str("!"); // this will cause an error.
  println!("Greeting: {}", message);
}

fn main() {
  let greeting = String::from("Hello");
  print_greeting(&greeting); // `print_greeting` takes a `&String` not an owned `String` so we borrow `greeting` with `&`
  print_greeting(&greeting); // Since `greeting` didn't move into `print_greeting` we can use it again
}
```

此时我们可以传入**可变引用**，这样就可以修改它的值了：

```rust
fn main() {
    let mut greeting = String::from("hello");
    change(&mut greeting);
}

fn change(text: &mut String) {
    text.push_str(", world");
}
```

#### 借用和可变引用

Rust内存管理的核心就是借用和可变引用，Rust代码必须同时实现以下其中一个引用，但是不能同时实现：

- 变量有一个或多个不可变引用(&T)
- 变量恰好有一个可变引用(&mut T)

```rust
fn main() {
    let mut value = String::from("hello");

    let ref1 = &mut value;
    let ref2 = &mut value;  // can't get a second mutable borrow

    println!("{}, {}", ref1, ref2);
}
```

```rust
fn main() {
    let mut value = String::from("hello");

    let ref1 = &value;
    let ref2 = &mut value;  // can't use borrow and mutable borrow at the same time.

    println!("{}, {}", ref1, ref2);
}
```

这样可以防止Rust出现大量的问题，包括不进行数据争用：

- 一个变量在被一个或多个只读引用时，不会被修改
- 一个变量在可能会被修改时，不会在别的地方被读写

### 引用与生命周期

当被引用的变量超出生命周期时，引用它的变量也会变得不可用（在C++中变成空指针），此时使用它的话也会带来错误：

```rust
fn main() {
    let x;
    {
        let y = 42;
        x = &y; // We store a reference to `y` in `x` but `y` is about to be dropped.
    }
    println!("x: {}", x); // `x` refers to `y` but `y has been dropped!
}
```

此时两个变量的生命周期可以这样表示：

```rust
fn main() {
    let x;                // ---------+-- 'a
    {                     //          |
        let y = 42;       // -+-- 'b  |
        x = &y;           //  |       |
    }                     // -+       |
    println!("x: {}", x); //          |
}
```

编译器通过验证变量`x`的生命周期是否包含了被引用的变量`y`的生命周期，此处`'b`并没有包含`'a`（反而反过来了），所以编译不通过。

#### 函数与生命周期

对于函数来说，并不能知道传进来引用的生命周期，如果函数要根据条件返回其中一个变量时，要确保它们在返回时还有效：

```rust
fn main() {
    let magic1 = String::from("abracadabra!");
    let result;
    {
        let magic2 = String::from("shazam!");
        result = longest_word(&magic1, &magic2);
    }
    println!("The longest magic word is {}", result);
}

fn longest_word(x: &String, y: &String) -> &String {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

Rust规定函数在返回引用时，要知道它们的生命周期，并通过一定的方式检测变量是否在返回后，依然在生命周期中（而不是返回一个已经被释放的数据）。

Rust中声明生命周期的方法为：

```rust
fn longest_word<'a>(x: &'a String, y: &'a String) -> &'a String {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

此处的`'a`可以用任何字符串代替，只是给生命周期起个名字。在确定好生命周期后，rust就能检测出上述代码的错误：`magic2`的生命周期与`magic1`并不相同（而我们在定义函数时都预期它们的生命周期为`'a'`，这样返回时可能会导致错误。

### 实验

实现copy_and_return函数，将参数插入到向量中，同时返回参数。

```rust
// 可以直接在指定好value和返回值的生命周期后，返回value
fn copy_and_return<'a>(vector: &mut Vec<String>, value: &'a str) -> &'a str {
    vector.push(String::from(value));
    value
}

fn main() {
    let name1 = "Joe";
    let name2 = "Chris";
    let name3 = "Anne";

    let mut names = Vec::new();

    assert_eq!("Joe", copy_and_return(&mut names, &name1));
    assert_eq!("Chris", copy_and_return(&mut names, &name2));
    assert_eq!("Anne", copy_and_return(&mut names, &name3));

    assert_eq!(
        names,
        vec!["Joe".to_string(), "Chris".to_string(), "Anne".to_string()]
    )
}
```

```rust
// 还可以在插入后返回向量的最后一个值，因为返回了向量相关值，所以也要指定向量的生命周期
fn copy_and_return<'a>(vector: &'a mut Vec<String>, value: &'a str) -> &'a String {
    vector.push(String::from(value));
    vector.get(vector.len()-1).unwrap()
}
```

## 使用泛型类型(generic type)和特征(traits)

使用泛型可以用一个函数来处理多种输入类型的情况，提高数据的易维护性。

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let boolean = Point { x: true, y: false };
    let integer = Point { x: 1, y: 9 };
    let float = Point { x: 1.7, y: 4.3 };
    let string_slice = Point { x: "high", y: "low" };
}
```

此时`x`和`y`都必须是同一类型，若要让两者不同类型，则需要定义两个范型：

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

fn main() {
    let integer_and_boolean = Point { x: 5, y: false };
    let float_and_string = Point { x: 1.0, y: "hey" };
    let integer_and_float = Point { x: 5, y: 4.0 };
    let both_integer = Point { x: 10, y: 30 }; // same type also accepted
    let both_boolean = Point { x: true, y: true };
}
```

### 使用特征(traits)定义共享行为

特征指的是一组类型可实现的通用接口，比如`ToString`可以将各种未知类型的数据转化为字符串，`traits`和C++里的`interface`有些类似。

我们定义一个`Area`的特征，来表示对象的面积，定义`area`函数来制定接口形式：

```rust
trait Area {
    fn area(&self) -> f64
}
```

我们创建一些新的类型，并实现`Area`特征：

```rust
struct Circle {
    radius: f64
}

struct Rectangle {
    width: f64
    height: f64
}

impl Area for Circle {
    fn area(&self) -> f64 {
        use std::f64::consts::PI; // use PI
        PI * self.radius.powf(2.0)
    }
}

impl Area for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
}

fn main() {
    let circle = Circle { radius: 5.0 };
    let rectangle = Rectangle {
        width: 10.0,
        height: 20.0,
    };

    println!("Circle area: {}", circle.area());
    println!("Rectangle area: {}", rectangle.area());
}
```

### 派生(derive)特征

我们在实现自定义类型时，可能需要对类型进行以下操作：

- 比较
- 输出
- Debug输出

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 4, y: -3 };

    if p1 == p2 { // can't compare two Point values!
        println!("equal!");
    } else {
        println!("not equal!");
    }

    println!("{}", p1); // can't print using the '{}' format specifier!
    println!("{:?}", p1); //  can't print using the '{:?}' format specifier!

}
```

此时代码编译失败，因为我们没有实现`Debug`、`Display`和`PartialEq`特征，使用派生特征可以自动实现`Debug`和`PartialEq`，但Rust并不提供`Display`的标准实现：

```rust
#[derive(Debug, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}
```

若需要实现`Display`特征，我们需要手动实现：

```rust
use std::fmt;

impl fmt::Display for Point {
    fn fmt(&self, f:&mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
```

### 特征边界和范型函数

当一个类型实现了某个特征后，我们可以把它看作**拥有某特征的类型**，这点可以体现在函数传递中。我们在定义函数的参数列表时，可以将参数指定为**实现了某特征的类型**，然后在函数中调用这个类型所实现的方法（虽然函数可能并不了解它的具体实现）。

假设我们的Web应用中定义了一个将值序列化为JSON的特征：

```rust
trait AsJson {
    fn as_json(&self) -> String;
}
```

我们的函数接受实现了该特征的类型，并调用`as_json`方法：

```rust
fn send_data_as_json(value: &impl AsJson) {
    println!("Sending JSON data to server...");
    println!("-> {}", value.as_json());
    println!("Done!\n");
}
```

### 迭代器

Rust中所有的迭代器都会实现标准库中的`Iterator`特征，其核心部分为：

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

创建迭代器需要两个步骤：

1. 创建一个结构来保留迭代器的状态。
2. 实现该结构的迭代器。

定义迭代器`Counter`用于对1到任意数字进行计数，这里实现`new`方法来初始化一个计数器

```rust
struct Counter {
    length: usize,
    count: usize
}

impl Counter {
    fn new(length: usize) -> Counter {
        Counter {
            count: 0,
            length
        }
    }
}
```

接下来实现`Iterator`特征，这里将`Item`定义为`usize`：

```rust
impl Iterator for Counter {
    type Item = usize;
    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;
        if self.count <= self.length {
            Some(self.count)
        } else {
            None
        }
    }
}
```

这样就可以用`for in`来遍历该对象：

```rust
fn main() {
    for number in Counter::new(10) {
        println!("{}", number);
    }
}
```

### 实现范型类型

实现一个`Container`结构，让他的value值可以为任意给定的类型。

使用`impl`来实现范型函数时，需要在`impl`和`Container`后都定义类型`<T>`：

```rust
struct Container<T> {
    value: T
}

impl<T> Container<T> {
    pub fn new(value: T) -> Container<T> {
        Container{ value }
    }
}
```

可以直接用`Self`返回类型本身

```rust
impl<T> Container<T> {
    pub fn new(value: T) -> Self {
        Container{ value }
    }
}
```

验证:

```rust
fn main() {
    assert_eq!(Container::new(42).value, 42);
    assert_eq!(Container::new(3.14).value, 3.14);
    assert_eq!(Container::new("Foo").value, "Foo");
    assert_eq!(Container::new(String::from("Bar")).value, String::from("Bar"));
    assert_eq!(Container::new(true).value, true);
    assert_eq!(Container::new(-12).value, -12);
    assert_eq!(Container::new(Some("text")).value, Some("text"));
}
```

### 实现连续相同数分组的迭代器

- 输入：[ 1, 1, 2, 1, 3, 3 ]
- 输出：[ [1, 1], [2], [1], [3, 3] ]

```rust
struct Groups<T> {
    inner: Vec<T>,
}

impl<T> Groups<T> {
    fn new(inner: Vec<T>) -> Self {
        Groups { inner }
    }
}

impl<T: PartialEq> Iterator for Groups<T> {
    type Item = Vec<T>;

    fn next(&mut self) -> Option<Self::Item> {
        if self.inner.is_empty() {
            return None;
        }
        
        let mut index=1;
        for item in &self.inner[1..] {
            if item == &self.inner[0] {
                index += 1;
            } else {
                break;
            }
        }
        
        let items = self.inner.drain(0..index).collect();
        
        Some(items)
    }
}

fn main() {
    let data = vec![4, 1, 1, 2, 1, 3, 3, -2, -2, -2, 5, 5];
    // groups:     |->|---->|->|->|--->|----------->|--->|
    assert_eq!(
        Groups::new(data).into_iter().collect::<Vec<Vec<_>>>(),
        vec![
            vec![4],
            vec![1, 1],
            vec![2],
            vec![1],
            vec![3, 3],
            vec![-2, -2, -2],
            vec![5, 5],
        ]
    );

    let data2 = vec![1, 2, 2, 1, 1, 2, 2, 3, 4, 4, 3];
    // groups:      |->|---->|---->|----|->|----->|->|
    assert_eq!(
        Groups::new(data2).into_iter().collect::<Vec<Vec<_>>>(),
        vec![
            vec![1],
            vec![2, 2],
            vec![1, 1],
            vec![2, 2],
            vec![3],
            vec![4, 4],
            vec![3],
        ]
    )
}

```

值得注意的是，这里使用了`&self.inner[1..]`。直接使用`self.inner[1..]`会有如下错误：

```
error[E0277]: the size for values of type `[T]` cannot be known at compilation time
   --> src/main.rs:21:21
    |
21  |         for item in self.inner[1..] {
    |                     ^^^^^^^^^^^^^^^ doesn't have a size known at compile-time
    |
    = help: the trait `Sized` is not implemented for `[T]`
    = note: required because of the requirements on the impl of `IntoIterator` for `[T]`
note: required by `into_iter`
```

个人的理解是，如果这里直接使用`self.inner`，编译器不能知道这个`Group`的`inner`向量的长度（这里的`inner`可能会是任意`Group`变量的向量），如果改成这个`inner`的引用，就可以在编译的时候知道各`Group`变量的向量的长度。

