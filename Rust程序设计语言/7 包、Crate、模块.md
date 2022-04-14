一个包可以包含多个二进制crate项和一个可选的crate库。

Rust 有许多功能可以让你管理代码的组织，包括哪些内容可以被公开，哪些内容作为私有部分，以及程序每个作用域中的名字。这些功能。这有时被称为 “模块系统（the module system）”，包括：

- **包**（*Packages*）： Cargo 的一个功能，它允许你构建、测试和分享 crate。
- **Crates** ：一个模块的树形结构，它形成了库或二进制项目。
- **模块**（*Modules*）和 **use**： 允许你控制作用域和路径的私有性。
- **路径**（*path*）：一个命名例如结构体、函数或模块等项的方式。

## 7.1 包和crate

crate 是一个二进制项或者库。*包*是是提供一系列功能的一个或多个crate。一个包会包含一个Cargo.toml文件，阐述如何构建这些crate。

一个包中至多包含一个*库crate*。

一个包中可以包含任意多个*二进制crate*。

一个包中至少包含一个crate，无论是库的还是二进制的。

当我们使用`cargo new`创建一个新项目时，cargo会自动创建一个包，并创建`src/main.rs`文件，这个文件是与包同名的*二进制crate*的**crate根**。类似地，如果项目包含`src/lib.rs`文件，cargo则认为包带有同名的库crate，且`src/lib.rs`是crate根。crate根文件将由Cargo传递给`rustc`来构建库或者二进制项目。

## 7.2 模块 作用域 私有性

*模块* 让我们可以将一个 crate 中的代码进行分组，以提高可读性与重用性。模块还可以控制项的 *私有性*，即项是可以被外部代码使用的（*public*），还是作为一个内部实现的内容，不能被外部代码使用（*private*）。

本章使用一个餐馆的例子。餐馆中会有一些地方被称之为 *前台*（*front of house*），还有另外一些地方被称之为 *后台*（*back of house*）。前台是招待顾客的地方，在这里，店主可以为顾客安排座位，服务员接受顾客下单和付款，调酒师会制作饮品。后台则是由厨师工作的厨房，洗碗工的工作地点，以及经理做行政工作的地方组成。

使用`cargo new --lib restaurant`创建一个新的名为`restaurant`的库。在`src/lib.rs`中编写如下代码：

```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}
        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}
        fn server_order() {}
        fn take_payment() {}
    }
}
```

一个模块以`mod`关键字开始，然后指定模块的名字。

模块可以嵌套。

一个模块可以包含结构体、枚举、常量、特性或者函数等。

这些模块以`src/lib.rs`为根形成了一棵模块树（module tree），其结构如下：

```
crate
 └── front_of_house
     ├── hosting
     │   ├── add_to_waitlist
     │   └── seat_at_table
     └── serving
         ├── take_order
         ├── serve_order
         └── take_payment
```

在上图中，`hosting`和`serving`互为*兄弟（siblings）*。如果一个模块 A 被包含在模块 B 中，我们将模块 A 称为模块 B 的 *子*（*child*），模块 B 则是模块 A 的 *父*（*parent*）。

## 7.3 路径

在Rust中，我们使用路径的方式来在模块树中找到一个项的位置。

路径有两种形式：

- **绝对路径**（*absolute path*）从 crate 根开始，以 crate 名或者字面值 `crate` 开头。
- **相对路径**（*relative path*）从当前模块开始，以 `self`、`super` 或当前模块的标识符开头。

绝对路径和相对路径都后跟一个或多个由双冒号`::`分割的标识符。

```rust
mod front_of_house {
    //略，完整代码见上
}

pub fn eat_at_restaurant() {
    //绝对路径
    crate::front_of_house::hosting::add_to_waitlist();

    //相对路径
    front_of_house::hosting::add_to_waitlist();
}
```

函数`eat_at_restaurant`和模块`front_of_house`在同一个层级，因此函数可以直接调用模块。考虑将二者同时移动到一个新的模块，则绝对路径的调用会失效，而相对路径不会失效。

不过这段代码不能通过编译，因为模块`front_of_house`是（默认）私有的。Rust中函数、方法、结构体、枚举、模块和常量默认都是私有的。

### 使用`pub`关键字暴露路径

父模块中的项不能使用子模块中的私有项，但是子模块中的项可以使用他们父模块中的项。以餐馆为例，餐馆内的事务对餐厅顾客来说是不可知的，但办公室经理可以洞悉其经营的餐厅并在其中做任何事情。

可以使用`pub`关键字来创建公共项，使子模块的内部暴露给上级模块。但`pub`模块的内容仍然是私有的，除非将该内容也用`pub`进行标记。

兄弟模块可以互相访问而无需`pub`标记。

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}
```

如此修改后，上面的`eat_at_restaurant`函数就可以正常工作。

### 使用`super`起始的相对路径

我们还可以使用 `super` 开头来构建从父模块开始的相对路径。这么做类似于文件系统中以 `..` 开头的语法。考虑后厨的厨师修改了一个订单并亲自将其提供给顾客：

```rust
fn serve_order() {}

mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        super::serve_order();
    }

    fn cook_order() {}
}
```

### 创建公有的结构体和枚举

在结构体和枚举的声明前加上`pub`可以使其成为公有的，但公有结构体的字段默认还是私有的，而公有枚举的字段都会自动变成公有的。这是因为部分私有而部分公有的枚举意义不大，而手动在所有字段前加`pub`显得麻烦。

在餐馆中，顾客可以选择随餐附赠的面包类型，但是厨师会根据季节和库存情况来决定随餐搭配的水果。餐馆可用的水果变化是很快的，所以顾客不能选择水果，甚至无法看到他们将会得到什么水果。

```rust
mod back_of_house {
    pub struct Breakfast {
        pub toast: String,
        seasonal_fruit: String,
    }

    impl Breakfast {
        pub fn summer(toast: &str) -> Breakfast {
            Breakfast {
                toast: String::from(toast),
                seasonal_fruit: String::from("peaches"),
            }
        }
    }
}

pub fn eat_at_restaurant() {
    let mut meal = back_of_house::Breakfast::summer("Rye");
    //改变面包的类型
    meal.toast = String::from("Wheat");
    println!("I'd like {} toast please", meal.toast);

    //下面这行不能通过编译，因为meal.seasonal_fruit是私有的
    // meal.seasonal_fruit = String::from("blueberries");
}
```

因为 `Breakfast` 具有私有字段，所以它需要提供一个公共的关联函数来构造实例(这里我们命名为 `summer`)。如果 `Breakfast` 没有这样的函数，我们将无创建 `Breakfast` 实例，因为我们不能设置私有字段 `seasonal_fruit` 的值。

## 7.4 使用`use`将名称引入作用域

每次我们想要调用`add_to_waitlist`时，都要写`front_of_house::hosting::add_to_waitlist`，这样显得有些啰嗦。

我们可以使用`use`将路径一次性引入作用域，然后像调用本地项一样调用其中的项。

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```

`use`同样支持相对路径。对于上面的代码，`crate::`是可省略的，省略后使用的就是相对路径。

### 创建惯用的`use`路径

我们可能奇怪为什么不直接引入`front_of_house::hosting::add_to_waitlist`，然后在调用时直接`add_to_waitlist`（这种方法可以正常通过编译）。因为上面的做法是Rust中的惯用法，这样可以提醒用户`add_to_waitlist`函数不是在本地定义的，而是在某个模块中。

另一方面，使用 `use` 引入结构体、枚举和其他项时，习惯是指定它们的完整路径。下面是将 `HashMap` 结构体引入二进制 crate 作用域的习惯用法。

```rust
use std::collections::HashMap;

fn main() {
    let mut map = HashMap::new();
    map.insert(1, 2);
}
```

上面的例外是，如果两个模块中有同名的项，则应引入父模块。

```rust
use std::fmt;
use std::io;

fn function1() -> fmt::Result {}

fn function2() -> io::Result<()> {}
```

### 使用`as`提供新的名称

```rust
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {}

fn function2() -> IoResult<()> {}
```

上面两种方法都是惯用的。

### 使用`pub use`重导出名称

### 使用外部包

标准库（`std`）对于你的包来说也是外部 crate。因为标准库随 Rust 语言一同分发，无需修改 *Cargo.toml* 来引入 `std`，不过需要通过 `use` 将标准库中定义的项引入项目包的作用域中来引用它们。

### 嵌套路径来消除大量的`use`行

当需要引入很多定义于相同包或相同模块的项时，为每一项单独列出一行会占用源码很大的空间：

```rust
use std::cmp::Ordering;
use std::io;
//...
```

可以使用嵌套路径将相同的项在一行中引入作用域。这么做需要指定路径的相同部分，接着是两个冒号，接着是大括号中的各自不同的路径部分。

```rust
use std::{cmp::Ordering, io};
```

另一个例子：

```rust
use std::io;
use std::io::Write;
```

可以简写成

```rust
use std::io::{self, Write};
```

### 通过`glob`运算符将所有的公有定义引入作用域

使用`glob`(`*`)运算符可以一次性将一个路径下**所有**的公有项引入作用域。

```rust
use std::collections::*;
```

`glob`运算符经常用于测试模块 `tests` 中，这时会将所有内容引入作用域；`glob`运算符有时也用于 prelude 模式。

## 7.5 将模块分割进不同文件

下面将`front_of_house`模块移动到一个单独的文件`src/front_of_house.rs`中。

文件`src/front_of_house.rs`：

```rust
pub mod hosting {
    pub fn add_to_waitlist() {}
}
```

文件`src/lib.rs`：

```rust
mod front_of_house;		//告诉Rust在另一个与模块同名的文件中加载模块的内容

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
    hosting::add_to_waitlist();
}
```

类似地，可以将`src/front_of_house.rs`修改为

```rust
pub mod hosting;
```

同时创建文件`src/hosting.rs`，写入

```rust
pub fn add_to_waitlist() {}
```

效果将完全一致。

