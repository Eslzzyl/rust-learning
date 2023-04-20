 **trait**是一个定义泛型行为的方法。trait 可以与泛型结合来将泛型限制为拥有特定行为的类型，而不是任意类型。

**生命周期**（*lifetimes*）是一类允许我们向编译器提供引用如何相互关联的泛型。Rust 的生命周期功能允许在很多场景下借用值的同时仍然使编译器能够检查这些引用的有效性。

下面的函数从一个`i32`类型的vector中找出最大项。

```rust
fn largest(list: &[i32]) -> i32 {
    let mut largest = list[0];

    for &item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

其参数`list`的类型是`i32`值的slice。

## 10.1 泛型数据类型

### 在函数定义中使用泛型

考虑将上面的`largest`函数改为对一个`char`序列进行作用。那么需要定义一个新的函数：

```rust
fn largest_char(list: &[char]) -> char {
    let mut largest = list[0];

    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

注意Rust**不支持**函数重载，但是可以使用trait实现类似的效果。

上面两个函数除了签名（参数+返回值）不同外，其他部分基本相同。我们可以引进泛型参数来消除这种重复。

```rust
fn largest<T>(list: &[T]) -> T {
```

`T`是类型参数，可以是符合命名标准的任何名字，但`T`是最常用的。

Rust中类型的命名应使用驼峰命名法。

### 结构体定义中的泛型

下面的结构体可以存放任何类型的坐标值。（类似C++中的类模板）

```rust
struct Point<T> {
    x: T,
    y: T,
}
fn main() {
    let integer = Point { x: 5, y: 10};
    let float = Point {x: 1.0, y: 4.0};
}
```

这里`x`和`y`的类型虽然可以变化，但它们必须是同一种类型。如果想使用不同类型，可以：

```rust
struct Point<T, U> {
    x: T,
    y: U,
}
```

定义中可以使用任意多的泛型类型参数，但不宜太多，以免影响可读性。

### 枚举定义中的泛型

标准库中的`Option`和`Result`都是这类例子。

```rust
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

### 方法定义中的泛型

下面的代码在上面的`Point<T>`的基础上实现了名为`x`的方法。

```rust
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

fn main() {
    let p = Point {x: 5, y: 10};
    println!("p.x = {}", p.x());
}
```

注意必须在 `impl` 后面声明 `T`，这样就可以在 `Point<T>` 上实现的方法中使用它了。在 `impl` 后面声明 `T`告诉编译器`Point`的尖括号中的类型是泛型而不是一个具体类型。

上面的限制暗示我们可以为泛型结构体的某一个特化版本定义其专有方法，实际上确实如此。下面的代码为`Point<f32>`类型定义了专有方法。

```rust
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

结构体定义中的泛型类型参数并不总是与结构体方法签名中使用的泛型是同一类型。

```rust
impl<T, U> Point<T, U> {
    fn mixup<V, W>(self, other: Point<V, W>) -> Point<T, W> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}
```

### 泛型代码的性能

Rust通过在编译时进行泛型代码的所谓**单态化**（monomorphization）来保证效率。具体类型将在编译时填充，故不会产生任何额外的运行时消耗。

编译器在编译时会查找所有使用了泛型的地方，自动地实例化出对应的具体类型。比如

```rust
let integer = Some(5);
let float = Some(5.0);
```

这段代码在编译期会产生下面的代码：

```rust
enum Option_i32 {
    Some(i32),
    None,
}

enum Option_f64 {
    Some(f64),
    None,
}

fn main() {
    let integer = Option_i32::Some(5);
    let float = Option_f64::Some(5.0);
}
```

## 10.2 trait：定义共享的行为

trait告诉编译器某个特定类型拥有可能与其他类型共享的功能。

- 可以通过trait以一种抽象的方式定义共享的行为。
- 可以使用trait bounds指定泛型是任何拥有特定行为的类型。

trait类似其他语言中的接口（interface），但不完全相同。

### 定义trait

如果可以对不同的类型调用相同的方法，这些类型就可以共享相同的行为了。

假设有一些媒体内容，`NewsArticle`结构体用于存放新闻，`Tweet`结构体存放推特内容。现在想要实现一个功能，总结上面两类媒体内容。下面是一个表现这个概念的`Summary` trait的定义：

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

使用`trait`关键字声明一个trait，后面跟着trait的名字。

大括号中声明描述实现这个trait的类型所需要的行为的 方法签名。方法签名后跟分号（类似C/C++的函数声明），并不实现方法。接着每一个实现这个trait的类型都必须实现自己的`summarize`方法，签名应当保持完全一致。

trait体中可有多个方法。

### 为类型实现trait

下面的代码分别为`NewsArticle`和`Tweet`实现了`Summary` trait。

```rust
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

在类型上实现trait类似实现普通的方法，不同的是`impl`关键字之后，我们需要提供需要实现的trait的名称，接着是`for`和完成实现的类型名称。

完成实现之后，就可以像调用普通方法那样调用trait方法。

```rust
println!("1 new tweet: {}", tweet.summarize());
```

如果想要实现定义在其他作用域的trait，需要先将其引入当前作用域。假设上面的`Summary` trait是定义在`aggregator` crate中的，那么需要使用`use aggregator::Summary;`。`Summary`还必须是`pub`的。

#### 孤儿规则

实现trait的一个限制是，只有当trait，或者 要实现trait的类型位于crate的本地作用域时，才能实现。比如

- 可以为`aggregator` crate的自定义类型`Tweet`实现标准库中的`Display` trait，因为`Tweet`类型位于本地作用域。
- 也可以在`aggregator` crate中为`Vec<T>`实现`Summary` trait，这是因为`Summary` trait位于`aggregator` crate的本地作用域中。

但是不能为外部类型实现外部trait。比如不能在`aggregator` crate中为`Vec<T>`实现`Display` trait。

上面这个限制是被称为 **相干性**（*coherence*） 的程序属性的一部分，或者更具体的说是 **孤儿规则**（*orphan rule*），其得名于不存在父类型。

这条规则确保了其他人编写的代码不会破坏你的代码，反之亦然。没有这条规则的话，两个 crate 可以分别对相同类型实现相同的 trait，而 Rust 将无从得知应该使用哪一个实现。

### 默认实现

有时需要为trait中的某些或全部方法提供默认的行为。这样当实现trait时，可以选择保留默认行为或另给出新的行为。

下面的代码为`summarize`方法指定了一个默认的字符串值。

```rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
```

为某个类型实现`Summary`时，指定一个空的`impl`块表示使用trait中的默认行为。

```rust
impl Summary for NewsArticle { }
```

这就算是实现了`Summary`。

如果想要更改默认行为，按照原先的实现方式进行实现即可。

### trait作为参数

假设我们已经为`NewsArticle`和`Tweet`实现了`Summary` trait。现在可以定义一个函数`notify`来调用`summarize`方法：

```rust
pub fn notify(item: impl Summary) {
    println!("Breaking new! {}", item.summarize());
}
```

上面的写法表示`notify`支持 任何实现了`Summary` trait的类型（如`NewsArticle`或`Tweet`的实例）作为实参。

### Trait Bound 语法

上面在参数中使用的的`impl [Trait]`语法实际上是一种较长形式的语法糖。这种形式名为trait bound。

```rust
pub fn notify<T: Summary>(item: T) {
    println!("Breaking news! {}", item.summarize());
}
```

trait bound的优势在于它可以支持一些比较复杂的场景。如，希望接受两个实现了`Summary`的类型作为参数，`impl [Trait]`写法如下：

```rust
pub fn notify(item1: impl Summary, item2: impl Summary) {...}
```

这种写法适用于希望两个`item`是不同类型的情况（只要它们都实现了`Summary`），但如果想要强制它们都是相同类型，则只能用trait bound：

```rust
pub fn notify<T: Summary>(item1: T, item2: T) {...}
```

#### 通过 + 指定多个 trait bound

如果希望`item`参数同时实现多个trait，则可以

```rust
pub fn notify(item: impl Summary + Display) {...}
```

或

```rust
pub fn notify<T: Summary + Display>(item: T) {...}
```

#### 通过 where 简化 trait bound

使用过多的trait bound也有缺点。如果trait bound太长，则函数在名称和参数列表中间会有很长的信息，可读性差。

对此，还有一种把trait bound放在签名之后的写法。

例如，对于下面的函数：

```rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: T, u: U) -> i32 {...}
```

可以改成：

```rust
fn some_function<T, U>(t:T, u: U) -> i32
	where T: Display + Clone,
		  U: Clone + Debug
{...}
```

### 返回实现了trait的类型

`impl [Trait]`语法同样适用于返回值。

```rust
fn returns_summarizable() -> impl Summary {
    Tweet {
        username: String::from("horse_ebooks"),
        content: String::from("of course, as you probably already know, people"),
        reply: false,
        retweet: false,
    }
}
```

这种情况常用于闭包和迭代器的实现。

不过这种情况只能用于返回单一类型的函数。比如下面这个函数将无法通过编译：

```rust
fn returns_summarizable(switch: bool) -> impl Summary {
    if switch {
        NewsArticle {
            headline: String::from("Penguins win the Stanley Cup Championship!"),
            location: String::from("Pittsburgh, PA, USA"),
            author: String::from("Iceburgh"),
            content: String::from("The Pittsburgh Penguins once again are the best
            hockey team in the NHL."),
        }
    } else {
        Tweet {
            username: String::from("horse_ebooks"),
            content: String::from("of course, as you probably already know, people"),
            reply: false,
            retweet: false,
        }
    }
}
```

### 使用 trait bounds 修复 largest 函数

回到10.1节中的`largest`函数。

```rust
fn largest<T>(list: &[T]) -> T {
    let mut largest = list[0];
    for &i in list.iter() {
        if largest < i {
            largest = i;
        }
    }
    largest
}
```

这个函数不能正常通过编译，因为我们使用了`<`来比较两个`T`。`<`运算符是`std::cmp::PartialOrd`的一个默认方法，所以需要在`T`的 trait bound中指定`PartialOrd`。

`PartialOrd`在 prelude 中，因而不需要显式引入作用域。

将`largest`的签名修改如下：

```rust
fn largest<T: PartialOrd>(list: &[T]) -> T {...}
```

然而还是不能通过编译。因为`let mut largest = lists[0];`这行涉及到一个移动操作，但泛型可能存储在栈上，因此无法移动。RLS推荐的方法是在`list[0]`前加`&`，以构成一个引用，这样`largest`会变成一个引用，同名函数的返回值也就成了一个引用。这是一种方案。

如果我们不希望修改返回的类型，就得将数据从`list[0]`中拷贝出来，从而避免移动。于是我们还需要对`T`加上`Copy` trait的限制。

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {...}
```

这样就解决了所有的问题。

不过存储在堆上的类型没有实现`Copy` trait，因而也就不能用于`largest`。这不是我们希望的。我们可以把`Copy`改成`Clone`。

### 使用 trait bound 有条件地实现方法

通过使用带有 trait bound 的泛型参数的 `impl` 块，可以有条件地只为那些实现了特定 trait 的类型实现方法。

```rust
struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self {
            x,
            y,
        }
    }
}

//只有为T类型实现了PartialOrd和Display的Pair<T>才有下面的方法
impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("The largest member is x = {}", self.x);
        } else {
            println!("The largest member is y = {}", self.y);
        }
    }
}
```

也可以为任何实现了特定 trait 的类型实现特定的 trait。比如，标准库为所有实现了`Display` trait 的类型实现了`ToString` trait：

```rust
impl<T: Display> ToString for T {
    ...
}
```

这种操作称为 blanket implementations。

可以对整型调用`to_string`方法，因为整型都实现了`Display` trait，从而也实现了`ToString` trait。

```rust
let s = 3.to_string();
```

blanket implementation 会出现在 trait 文档的 "Implementers" 部分。

## 10.3 生命周期与引用有效性

如何理解rust中的生命周期标注？：https://www.zhihu.com/question/435470652

每一个引用都有其生命周期，也就是引用保持有效的作用域。大部分时候，生命周期可以自动推断，就像泛型那样；有些时候则不能，需要显式注明。

### 生命周期避免了悬垂引用

考虑下面的程序

```rust
{
    let r;
    {
        let x = 5;
        r = &x;
    }
    println!("r: {}", r);
}
```

（注：Rust不允许空值，如果一个变量没有给出初值，则在其首次被使用之前必须赋值。）

上面的代码不能通过编译，原因是`r`所引用的值(即`x`)在被使用之前就离开了作用域。编译器将提示”`x` does not live long enough“。

#### 借用检查器

Rust编译器有一个借用检查器(borrow checker)，它通过比较作用域来确保所有的借用都是有效的。

对上面的程序加上生命周期注释：

```rust
{
    let r;                // ---------+-- 'a
                          //          |
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
                          //          |
    println!("r: {}", r); //          |
}                         // ---------+
```

可以看到，`r`的生命周期是`'a`，`x`的生命周期是`'b`，`'b`显然比`'a`要短，Rust 不允许被引用值（此处为`x`）的生命周期比引用它的值（此处为`r`）的生命周期短，于是报错。

下面的程序将正常通过编译：

```rust
{
    let x = 5;            // ----------+-- 'b
                          //           |
    let r = &x;           // --+-- 'a  |
                          //   |       |
    println!("r: {}", r); //   |       |
                          // --+       |
}                         // ----------+
```

### 函数中的泛型生命周期

考虑我们需要写一个比较两个字符串长度并返回较长者的函数。

```rust
fn longest(x: &str, y:&str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

上面的函数无法通过编译。

编译器不知道返回的引用是指向`x`还是指向`y`（事实上我们也不知道），此时需要增加泛型生命周期参数来定义引用间的关系，以便借用检查器开展分析工作。

### 函数签名中的生命周期注解

就像泛型类型参数，泛型生命周期参数需要声明在一个尖括号中，这个尖括号需要位于函数名和参数列表之间。

在上面的例子中，我们想要表达的限制是两个参数和返回的引用的生命周期是相关的，也就是这两个参数和返回的引用存活的一样久。

```rust
fn longest<'a>(x: &'a str, y: 'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

这段代码就能正常通过编译并完成我们需要的比较操作了。

现在的函数签名表示：对于某些生命周期 `'a`，函数会获取两个参数，他们都是与生命周期 `'a` 存在的一样长的字符串 slice。函数会返回一个同样也与生命周期 `'a` 存在的一样长的字符串 slice。

当具体的引用被传递给 `longest` 时，被 `'a` 所替代的具体生命周期是 `x` 的作用域与 `y` 的作用域相重叠的那一部分。换一种说法就是泛型生命周期 `'a` 的具体生命周期等同于 `x` 和 `y` 的生命周期中较小的那一个。（有点像集合相交）这样，返回的引用值就能保证在 `x` 和 `y` 中较短的那个生命周期结束之前保持有效。

```rust
let string1 = String::from("long string is long");
{
    let string2 = String::from("xyz");
    let result = longest(string1.as_str(), string2.as_str());
    println!("The longest string is {}", result);
}
```

可以看到，`result`引用的值在内部作用域结束之前都是有效的。而`result`本身也属于内部作用域。于是借用检查器认可这段代码，允许其通过编译。

```rust
let string1 = String::from("long string is long");
let result;
{
    let string2 = String::from("xyz");
    result = longest(string1.as_str(), string2.as_str());
}
println!("The longest string is {}", result);
```

上面的代码中，`string2`的生命周期比`string1`短，而`result`的生命周期因此和`string2`一样长。为了保证 `println!` 中的 `result` 是有效的，`string2` 需要直到外部作用域结束都是有效的。因此借用检查器不允许这段代码通过编译。

在函数中使用生命周期注解时，这些注解出现在函数签名中，而不存在于函数体的任何代码中。

添加生命周期注解实际上是精确了编译器的工作，这样编译器在出错时能够更精确地定位到错误。

### 深入理解生命周期

生命周期语法是用于将函数的多个参数与其返回值的生命周期进行关联的。

当从函数返回一个引用时，返回值的生命周期参数需要与某个（些）参数的生命周期参数相匹配。为什么？因为：如果返回的引用没有指向任何一个参数，那么唯一的可能就是这个引用指向了函数内部的某个值。这无疑是悬垂引用。

### 结构体定义中的生命周期注释

结构体中可以包含引用，但每个引用都必须添加生命周期注解。

```rust
struct ImportantExcerpt<'a> {	//注解表示该结构体的实例不能比part字段中的引用存在得更久
    part: &'a str,	//存放字符串slice，这时一个引用
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("Could not find a '.'");
    let i = ImportantExcerpt {
        part: first_sentence,
    };
}
```

### 生命周期省略（Lifetime Elision）

考虑下面的代码

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

函数`first_word`使用了引用，却没有生命周期注解，但它仍然能够编译通过。

在早期版本的 Rust 中，每一个引用都必须有明确的生命周期。那时`first_word`的签名就要写成这样：

```rust
fn first_word<'a>(s: &'a str) -> &'a str {...}
```

后来 Rust 团队发现在某些特定场合下程序员们总是重复地编写一模一样的生命周期注解。这些特定场合是可预测的，并且遵循几个明确的模式。于是这些模式就被编码进了编译器，允许代码编写者在这些情况下省略生命周期注解。

这里提到 Rust 的历史是因为未来可能有更多的模式被添加到编译器中。未来只会需要更少的生命周期注解。

被编码进 Rust 引用分析的模式被称为 **生命周期省略规则**（*lifetime elision rules*）。这里的*规则*是对编译器而言的，而不是对程序员而言。程序员不需要遵守什么特定的规则，规则的匹配是编译器的工作。

函数或方法的参数的生命周期被称为 **输入生命周期**（*input lifetimes*），而返回值的生命周期被称为 **输出生命周期**（*output lifetimes*）。

编译器采用三条规则来判断引用何时不需要明确的注解。

1. 每一个作为引用的参数默认都有独立的生命周期参数。
2. 如果只有一个输入生命周期参数，那么它将被赋予输出生命周期参数。
3. 如果方法有多个输入生命周期参数，并且其中一个参数是 `&self` 或 `&mut self`，说明这是一个对象的方法（见17章）, 那么所有输出生命周期参数被赋予 `self` 的生命周期。

如果编译器检查完这三条规则后仍然存在没有计算出生命周期的引用，编译器将会报错。

### 静态生命周期

静态生命周期 `'static` 是一种特殊的生命周期，其存活于整个程序运行期间。

所有的字符串字面值都拥有 `'static` 生命周期。

### 结合泛型类型参数、trait bounds 和生命周期

```rust
use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(
    x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
where
    T: Display,
{
    println!("Announcement! {}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

这是上面的`longest`函数外加了一个`ann`字段。它完成比较工作并额外输出`ann`字段。

注意，生命周期本身也是一种泛型，因此也写在尖括号里。
