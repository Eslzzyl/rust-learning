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

`T`是类型参数，可以是符合命名标准的任何名字，但`T`是最常用的。类型的命名应使用驼峰命名法。

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

