---
layout: post
title: Rust Notes
category: notes
---

## 1. 「变量」和「类型」

* 不变：一旦一个变量绑定了一个值，就不能被绑定到其他值上面
* 可变：该变量可以被绑定到其他值上
* 不变 vs. shadowing：shadowing 相当于创建了一个新的变量
* 静态语言：编译时就要知道所有变量的类型
* 标量：整型（有符号和无符号都支持），浮点（f32 和 f64），布尔，字符（注意：不是字符串）
* 复合类型：tup, array（stack上，固定长度）
* 字符串常量（string literal）vs. String 类型：前者在栈上（运行期不可变），后者在堆上（运行期可变）

## 2. copy / move / clone

针对 stack 上的数据会发生 copy。heap 上的数据会发生 move。例如：

```rust
#![allow(unused)]

fn main() {
// copy
    let x = 5; // bind the value 5 to x
    let y = x; // make a COPY of the value in x and bind it to y

// move
    let s1 = String::from("hello");
    let s2 = s1; // s1 was moved into s2。之后 s1 不再有效

// clone
    let s1 = String::from("hello");
    let s2 = s1.clone();
}
```

## 3. 所有权（ownership）

* 所有权（ownership）3 原则
  * Rust 中每一个值都有一个称之为其「所有者」（owner）的变量
    * 使用 let 来创建值或者资源，同时该变量成为资源的 owner
  * 值有且只能有一个所有者（变量）。
    * 当值从一个变量 **move** 给另外一个值时，所有权转移
    * 不过除了 **move** 之外，可以 **borrow** 一个值（后面会详细说）
  * 当所有者（变量）离开作用域，这个值将被丢弃，同时其资源也被释放

> **NOTE**：思考一下，我们使用 C 语言的时候，如果遵循这 3 个原则来使用指针？

* **move**：通过赋值（=）实现了堆上数据的拥有者（变量）的所有权的转移（类似 C 语言中的「浅拷贝」）
  * 一旦某个变量的所有权转移（move）走了以后，就不能再使用这个变量了
* 【move】和【copy】
  * 默认的赋值语句是 move（移动）
  * 可以通过实现 Copy trait 把默认的 move 改成 copy（复制，所谓的深拷贝）
  * **栈上**的变量的赋值都是 copy
* Copy trait 继承了 Clone trait
* **函数参数**的 move：就像赋值语句一样，栈上的是深拷贝，堆上的浅拷贝
* **函数返回值** move：就像赋值语句一样，栈上的是深拷贝，堆上的浅拷贝

> NOTE：一旦离开作用域，所有者（变量）拥有的堆上数据被释放（类似 C 语言的 free）

* 在【函数】和【闭包】中，在入参是 move 的场景（而不是 borrow 的场景），进入函数后，参数和返回值的 owner 关系发生改变，原先的变量可能不再有效

一个函数入参是 move 的例子：

```rust
fn main() {
    let s1 = String::from("abc");

    // 该字符串的 ownership 被 move，之后 s1 失去了这个字符串的 ownership，s1 不再有效
    let len = calculate_length(s1);
    // 不能再使用 s1（已经发生了 move）
    println!("The length is {}.", len);
}

// move
fn calculate_length(s: String) -> usize {
    // 函数内部拿到了这个字符串的 ownership
    s.len()
    // 当函数结束的时候，这个字符串被 dropped
}
```

## 4. borrowing 和 references

### a. 借用

* **借用** (borrowing)：对函数参数来说，如果不想 ownership 发生变化可以利用引用（reference）。下面给一个简单的例子：

```rust
fn main() {
    let s1 = String::from("hello");

    // 把 s1 的引用（&s1）作为参数传给函数，这样就可以达到 borrowing 的效果：s1 对这个字符串值的 ownership 不变
    let len = calculate_length(&s1);
    // s1 依旧是这个字符串值的 owner，可以使用
    println!("The length of '{}' is {}.", s1, len);
}

// 借用：使用引用（&String）作为函数的入参
fn calculate_length(s: &String) -> usize {
    // 函数内部使用引用 s（borrowing 到一个字符串的值）
    s.len()
    // 当函数结束的时候，这个引用（s）被 dropped
}
```

* 要想在函数内部修改借用的参数的值，需要利用「可变引用」：

```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

> NOTE：为啥不需要对这个可变引用进行 deference（类似 C 语言中的 * 操作符）？
>
> (*some_string).push_str(", world");
>
> 这是因为在「method」或者「field」上的 . 操作符可以自动 deference 引用。也就是说在这种场景，不需要区分是引用还是值，直接使用 . 操作符就行了：
>
> some_string.push_str(", world");
>
> 但对于其他场景，还是需要进行 deference，例如：
>
> (*some_string) = String::from("new string it");

### b. 引用

* **定义**：The &s1 syntax lets us create a reference that refers to the value of s1 but does not own it
  * **翻译**：s1 是一个值；而 &s1 就是指向这个值的「引用」（但并不 own 这个值）
* Rust 提供了「引用」这种方式来 access 一个 value：可使用 value，但不 own 这个 value。又分为 2 种「引用」：
  * &T；「shared 引用」；多个 「shared 引用」可共享同一个 value，但不能修改这个 value
  * &mut T；「可变引用」；不能共享同一个 value，但可以修改这个 value

和普通类型一样，引用实际上也是类型：

* i32：一个整数类型（32 位）
* &i32：一个指向 i32 的不变引用类型
* &mut i32：一个指向 i32 的可变引用类型

引用的例子：

```rust
#![allow(unused)]

fn main() {

    // s 是一个值
    let mut s = String::from("hello");

    let r1 = &s; // 创建到 s 的不可变引用，OK
    let r2 = &s; // 创建多个到 s 的不可变引用，也 OK
    let r3 = &mut s; // ERROR：再创建一个到 s 可变引用：不行了，因为已经创建了针对 s 的其他引用了
}
```

* 引用两原则
  * At any given time, you can have either (but not both of) one mutable reference or any number of immutable references
    * 在给定作用域中的给定值有且只有一个「可变引用」
    * if we have an immutable reference to something, we cannot also take a mutable reference
  * References must always be valid
    * 值的生命周期必须比指向它的引用的生命周期大（outlives）
    * 如果被引用的变量失效了，这个引用也就失效了

## 5. 生命周期（Lifetime）

> **【生命周期】原则：一个引用的生命周期不能超过其引用的变量的有效期**

* Rust 中，【生命周期】的概念只和变量的**引用**有关；Rust 中所有的引用都会关联一个「生命周期」
* 【生命周期】是程序可以安全使用这个**引用**的一个范围
* 【生命周期注解】告诉 Rust 多个引用的生命周期如何相互联系

### a. 函数定义使用生命周期注解

先给一个在函数中使用**生命周期注解**的例子：

* 告诉 Rust 这 3 个引用的【生命周期】的关系；返回值的生命周期和「两个入参」的生命周期相同：

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

编译时，会检查其是否符合生命周期原则：

```rust
fn main() {
    let string1 = String::from("long string is long");

    {
        let string2 = String::from("xyz");
        let result = longest(string1.as_str(), string2.as_str());
        println!("The longest string is {}", result);
    } // 内层代码块的结束
}
```

检查过程如下：

* 这里 2 个入参的生命周期 'a，是 string1 和 string2 这 2 个参数作用域重叠的部分（内层代码块的结束之前）
* 返回值 result 的生命周期也是 'a（string1 和 string2 这 2 个参数作用域重叠的部分）
* 那么 println! 使用 result 的时候，'a 是有效的（内层代码块的结束之前），所以**符合生命周期原则**

再来一个**不符合生命周期原则**的例子：

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {}", result);
}
```

* 这个例子中返回值 result 的生命周期也同样是：string1 和 string2 这 2 个参数作用域重叠的部分
* 但 println! 使用 result 的时候，已经超出了 'a 的有效范围（内层代码块的结束之后），所以**不符合生命周期原则**

### b. 结构体的生命周期注解

* 结构体的生命周期注解：结构内有引用的话，结构体本身的生命周期必须和其中引用的对象的生命周期保持一致

```rust
#![allow(unused)]

struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.')
        .next()
        .expect("Could not find a '.'");
    let i = ImportantExcerpt { part: first_sentence };
}
```

最后详细分析一个生命周期的例子。

先定义一个结构体 S：

```rust
struct S<'a> {
    x: &'a i32,
    y: &'a i32
}
```

然后在下面这段代码中使用这个结构体（具有 2 层代码块）：

```rust
fn main() {
    let x = 10;
    let r;
    {
        let y = 20;
        let s = S { x: &x, y: &y };
        r = s.x;
    } // 内层代码块的结束
} // 外层代码块的结束
```

检查过程如下：

* 结构体 S 的 2 个引用字段有相同的生命周期 'a
* 这个例子中，生命周期 'a 为变量 x 和 y 的重叠部分（也就是**内层代码块的结束**）
* 变量 s 的生命周期也要和这 2 个字段的生命周期 'a 相同，所以变量 s 的生命周期也必须是 x 和 y 这 2 个变量的重叠部分（也是**内层代码块的结束**）
* 但 r = s.x 这个赋值语句要求 s 生命周期 'a 也必须能涵盖 r 的生命周期（也就是说 'a 不能比 r 先结束而失效）
* 但实际上 r 的有效期需要到**外层代码块结束**
* 最终检查结果是：不符合「生命周期原则」

解决方法是修改结构体 S 的定义：

```rust
struct S<'a, 'b> {
    x: &'a i32,
    y: &'b i32
}
```

## 6. 指针

* Rust 中的指针有 3 种：
  * 【引用】：Rust 中安全的指针。也就是【非所有权指针】，分 &T 和 &mut T 两种。其中 &T 本身是一种 Copy 类型；而 &mut T 并没有实现 Copy Trait（Copy **trait** 的细节请参考其他小节）
  * 【原始指针】：也就是 Raw Points，用于 unsafe 代码；这里不详细介绍了
  * 【智能指针】：智能指针的 2 个关键 **trait**：Drop（离开作用域后，自动释放资源） 和 Deref。标准库提供了几种智能指针：Box\<T>，Rc\<T>，Arc\<T>，Cell\<T>，RefCell\<T>

这里只先给一个使用 Box 的例子，不深入更多具体细节了。

使用 Box 的例子：

```rust
#![allow(unused)]

fn main() {
    let x = 42;
    let y = Box::new(84);

    {
        let z = (x, y);
    } // 该 scope 结束后，不但 z 被释放，而且 y 也被释放（因为 y 这个变量是用 Box::new 来初始化的，所以用 y 来构建 z 的时候是 move，而不是 copy）

    let x2 = x; // 能通过生命周期检查
    let y2 = y; // 不能通过生命周期检查，因为 y 已经被释放
}
```

## 7. collections

* collections 存放的是指向 heap 上的数据的指针的集合（和 array/tuple 不一样）：
  * vector：存放同种类型的元素
  * string：Rust 标准库提供了 String 类型
  * hash map

### a. vector

* 两种方法获取 vector 中的某个元素
  * let does_not_exist = &v[100]; // 直接 panic，如果数组的 size 小于 100
  * let does_not_exist = v.get(100); // 返回 None，如果数组的 size 小于 100

* 下面代码编译时直接报错：first 是不变引用；但 **push 的时候发生的 borrow 行为**：编译失败。

```rust
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];

    let first = &v[0];

    v.push(6);
}
```

----

遍历 vector，并修改其中的值：

```rust
fn main() {
    let mut v = vec![100, 32, 57];
    for i in &mut v {
        *i += 50;
    }
}
```

----

如果一个 vector 中需要存不同类型的值，可以利用 Enum 类型来实现：

```rust
fn main() {
    enum SpreadsheetCell {
        Int(i32),
        Float(f64),
        Text(String),
    }

    let row = vec![
        SpreadsheetCell::Int(3),
        SpreadsheetCell::Text(String::from("blue")),
        SpreadsheetCell::Float(10.12),
    ];
}
```

总之，Rust 必须在编译期确定 vector 中的数据的类型。

### b. hash map

* Rust 中使用 std::collections::HashMap 来表示 1 对 1 的关系。
* Rust 的 Map，所有的 key 必须为相同的类型。所有的 value 必须为相同的类型。

----

创建一个 map 来记录 2 个队伍的分数：

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();

    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);
}
```

利用 vector 的 zip 方法在初始化时创建 HashMap：

```Rust
use std::collections::HashMap;

fn main() {
    let teams = vec![String::from("Blue"), String::from("Yellow")];
    let initial_scores = vec![10, 50];
    let scores: HashMap<_, _> = teams.iter().zip(initial_scores.iter()).collect();
}
```

----

Hash Map 会对其中的值有 ownership。下面的例子中，在执行了 map.insert 以后，field_name 和 field_value 失去对原先值的 ownership：

```rust
use std::collections::HashMap;

fn main() {
    let field_name = String::from("Favorite color");
    let field_value = String::from("Blue");

    let mut map = HashMap::new();
    map.insert(field_name, field_value); // 之后 field_name 和 field_value 失去对原先值的 ownership
}
```

----

拿到 HaspMap 中的值：

```rust
#![allow(unused)]

fn main() {
    let team_name = String::from("Blue");
    let score = scores.get(&team_name);
}
```

这里的 score 是一个 Some(&10)，也就是说，是一个 Option<&V>；如果值不存在，返回 None。

----

遍历 HashMap：

```rust
fn main() {
    for (key, value) in &scores {
        println!("{}: {}", key, value);
    }
}
```

----

利用 entry 方法和 or_insert 来实现只有当 key 不存在时才 insert 新值：

```rust
fn main() {
    scores.entry(String::from("Blue")).or_insert(50);
}
```

### c. String

String 特性

* UTF-8 编码
* growable
* mutable

----

新建 String

```rust
#![allow(unused)]

fn main() {
    let mut s = String::new(); // 新建
    let s = "initial contents".to_string(); // 带初始值的新建
    let s = String::from("initial contents"); // 和上面等价
}
```

* push_str 和 push

```rust
#![allow(unused)]

fn main() {
    let mut s1 = String::from("foo");
    let s2 = "bar";
    s1.push_str(s2); // push_str 把一个 string slice 加在后面。push_str 不需要 s2 的 ownership

    s.push('l'); // push 用来添加单个字符在后面
}
```

----

* 用 + 号连结

```rust
#![allow(unused)]

fn main() {
    let s1 = String::from("Hello, ");
    let s2 = String::from("world!");
    let s3 = s1 + &s2; // Note s1 has been moved here and can no longer be used
}
```

* 复杂的字符串拼接用 format!

```rust
#![allow(unused)]

fn main() {
    let s1 = String::from("tic");
    let s2 = String::from("tac");
    let s3 = String::from("toe");

    let s = format!("{}-{}-{}", s1, s2, s3);
}
```

* 支持 == 和 !=
  * 字符串内容（存储在内存中的字符串内容）是否相等的比较
    * 例如："th\u{e9}" 和 "the\u{301}" 是不相等的，尽管它们显示出来都是法文的 thé，但它们内存中的字符串内容不同
* Rust 保证 String 中的字符必须是有效的 UTF-8 编码的字符
  * 当真的要处理无效的 UTF-8 编码的时候，需要使用其他的类型（&Path，OsString 等等），而不能使用 String 类型

### d. slice / string slice

* slice 允许你引用 collection 中一段连续的元素序列，而不用引用整个 collection
  * 除了「引用」之外，slice 也不拥有数据的 ownership

例如，引用一个整数数组中的一部分：

```rust
#![allow(unused)]

fn main() {
    let a = [1, 2, 3, 4, 5];

    let slice = &a[1..3];
}
```

* Rust 专门有一个 string slice（&str）类型来表达 string literals：a reference to part of a String。

举个例子：

需要写一个函数获取字符串的第一个单词。函数入参可以是一个对 String 类型的引用（&String）。但返回值是什么呢？

这里返回值可以使用 string slice（&str）：

```rust
fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes(); // 把 String 转换为 byte 数组

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```

* 但在处理字符串时，尽量不要牵涉到 **byte** 操作，而是要尽量用**字符**相关的操作
  * 因为 Rust 中的字符是 UTF-8 编码，一个 UTF-8 编码的字符有几个 bytes 是不确定的

利用 chars 来遍历字段串的例子：

```rust
#![allow(unused)]

fn main() {
    for c in "您好".chars() {
        println!("{}", c);
    }
}
```

* String 上的 slice range 语法是 byte 级别的操作。如果尝试从一个多字节字符的中间位置创建字符串 slice，则程序将会因错误而退出。

```rust
#![allow(unused)]

fn main() {
    let s = String::from("hello world");

    // 该例子是 ASCII 字符集上的 String，所以不会出错
    let hello = &s[0..5];
    let world = &s[6..11];
}
```

----

其他一些关于 &str 的要点：

* "您好".len() 返回的是 **byte** 数，而不是字符数。"您好".chars().count() 才返回的是**字符数**
* &str 不能被修改；所以如果要在运行时修改一个字符串，需要使用 String
  * 不过 make_ascii_uppercase 和 make_ascii_lowercase 这 2 个方法是例外，它们会修改 &mut str

## 8. struct

定义一个 struct ，然后定义一个返回 struct 的方法

```rust
#![allow(unused)]

fn main() {
    struct User {
        username: String,
        email: String,
        sign_in_count: u64,
        active: bool,
    }

    fn build_user(email: String, username: String) -> User {
        User {
            // 变量名和结构体字段名相同时，可以使用初始化简写
            email,
            // 变量名和结构体字段名相同时，可以使用初始化简写
            username,
            active: true,
            sign_in_count: 1,
        }
    }
}
```

结构体的上下文中可以定义方法（method）:

* method 的第一个参数总是 self，它代表调用该方法的结构体实例

下面的例子给 Rectangle 这个结构体定义并实现了 area 和 can_hold（注意 **impl** 关键字的使用）：

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }

    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}

fn main() {
    let rect1 = Rectangle { width: 30, height: 50 };
    let rect2 = Rectangle { width: 10, height: 40 };

    println!("The area of the rectangle is {} square pixels.", rect1.area());
    println!("Can rect1 hold rect2? {}", rect1.can_hold(&rect2));
}
```

> 一般来说 method 的第一个参数可以是 &self 或者 &mut self

* **tuple structs**：tuple structs 的用法和普通 struct 类似，但 tuple structs  没有具体的字段名，只有字段的类型。简单点说，就是定义一个有类型名字的 tuple 类型。给个例子：

```rust
/// 定义一个颜色类型，其包含 3 个三元色的值
struct Color(i32, i32, i32);
```

## 9. enum

Rust 的枚举（enum）可以包含多种不同类型。例如：

```rust
#![allow(unused)]

fn main() {
    enum Message {
        Quit,
        Move { x: i32, y: i32 },
        Write(String),
        ChangeColor(i32, i32, i32),
    }
}
```

Rust 还可以在枚举上定义方法（类似 struct）。例如针对上面定义的 Message 枚举类型定义了一个 call 方法：

```rust
#![allow(unused)]

fn main() {
    enum Message {
        Quit,
        Move { x: i32, y: i32 },
        Write(String),
        ChangeColor(i32, i32, i32),
    }

    impl Message {
        fn call(&self) {
            // 在这里定义方法体
        }
    }

    let m = Message::Write(String::from("hello"));
    m.call();
}
```

标准库中的 Option 就是一个枚举类型，其定义如下：

```rust
enum Option<T> {
    Some(T),
    None,
}
```

## 10.match

* match 关键字后跟一个表达式
* 执行时，根据这个表达式计算出来的值，按顺序进行匹配，进入匹配成功的分支
* match 的每个分支由 2 部分组成
  * 模式：用来匹配是否进入该分支
  * 代码：进入该分支后需要执行的代码；
    * 而且每个分支的执行代码有一个结果值，被匹配到的分支的结果值就是整个 match 表达式的值

给一个复杂的例子：

```rust
#![allow(unused)]

fn main() {
    #[derive(Debug)]

    // 所有州的名字
    enum UsState {
        Alabama,
        Alaska,
    }

    // 各种面值的硬币，但 Quarter 面值的硬币有点特殊，不同州的 Quarter 硬币虽然面额相同，但硬币造型可能不一样
    enum Coin {
        Penny,
        Nickel,
        Dime,
        Quarter(UsState),
    }

    fn value_in_cents(coin: Coin) -> u8 {
        match coin {
            Coin::Penny => 1,
            Coin::Nickel => 5,
            Coin::Dime => 10,
            Coin::Quarter(state) => {
                // 不同州的 Quarter 硬币需要额外执行以下打印州名的动作（虽然返回值都同样是 25）
                println!("State quarter from {:?}!", state);
                25
            }
        }
    }
}

```

## 11. module 和 代码组织

几个关键词的作用：

* mod
* pub
* super
* use

另外就是下面 3 个 cargo 命令的用法：

* cargo new comm --lib
* cargo fmt
* cargo test

### a. Attribute

* 在 Rust 中，Attributes 类似 Java 的注解（annotations），或者 C/C++ 中的 #ifndef 这种给编译器使用的属性
  * Attributes 都是给编译器使用的

* 注解在 Rust 的各种 items 上
  * 例如：注解在 module 上，并对整个 module 生效（标注在 module 上：\#[cfg]，#[allow]）
  * Rust 中 item 的类型具体有：
    * module
    * extern crate 声明
    * use 声明
    * 函数定义
    * 类型定义
    * 结构体定义
    * 枚举定义
    * 联合体定义
    * 常量项
    * 静态项
    * trait 定义
    * 实现
    * 外部块
* crate 上：标注在整个 crate 的最前面，需要使用 \#![cfg] （多一个 ! 表示作用在整个 crate 上）
* Rust 中常用的 Attributes 有 4 大类：
  * Built-in attributes（内建属性）：下面会有些例子
  * Macro attributes（宏属性）：略
  * Derive macro helper attributes（派生宏辅助属性）
  * Tool attributes（工具属性）：略

一些常用的 Attributes：

* #[allow]：显示的关闭编译 warning。例如：#[allow(non_camel_case_types)]
* \#[cfg]：用于设置编译选项，具体配置项可参考说明。例如：#[cfg(test)]（用于标注只在测试时生效的代码）
* \#[test]：测试相关
* \#[doc]：自动生成代码文档
* \#[derive]：derive 属性会在使用 derive 语法标记的类型上生成对应 trait 的默认实现的代码。给 2 个例子：
  * \#[derive(Debug)]：只要给 struct 和 enum 加上这个 derive 属性，就可以在 println! 里面直接使用 {:?} 或者 {:#?}
  * \#[derive(Copy, Clone)]：Copy 和 Clone

## 12. 异常处理

* 开发人员作为不同身份时，可以利用不同的方式来应对异常
  * 作为库的开发人员
    * 定义这个库的专用异常类型，通过这个专用异常类型告诉库的使用者，当遇到不同的异常时，需要进行相应的处理
      * 举个例子，一个用于查询 MySQL 的库；当执行 SQL 语句失败时，需要告诉库的使用者失败原因；而库的使用者根据失败原因做不同的动作：可重试的进行重试，不可重试的直接通知最终用户本次请求失败等等
    * 而且大多数场景，库本身可以不直接打印日志（而是由库的使用者来决定库相关日志的配置，例如日志级别，日志文件位置）
  * 作为应用（在线服务）的开发人员
    * 不可恢复异常；对一个在线服务，开发人员可以考虑把「不可恢复异常」透传到最外层，在最外层完成下列工作：
      * 记录上下文（统计，排查，报警等目的）
      * 结束当前请求
      * 同时返回给最终用户一个友好的提示
    * 可恢复异常：这里的「可恢复异常」可以理解为程序正常执行的多种可能的分支中的一个逻辑分支
      * 开发人员的工作：把代码流程转入对应的逻辑分支（如有必要可打印日志，例如用于统计）

要达到以上目标，一种程序设计语言中，需要提供以下功能：

* 函数调用除了返回正常结果，也可以抛出异常
* 异常需要带必要的信息：异常码，异常信息，异常时的上下文信息（调用堆栈，代码行号等）
  * 尽可能提供模块级别（Module Level）的异常类，而不是全局（Crate Level）的异常类
  * 相同的底层异常（例如一个 IO 异常）抛出时，最好能区分该异常抛出时的上下文信息

* 便捷的把异常透传到最外层
* 捕捉到异常后，便捷的根据异常具体的信息，执行不同的代码逻辑
* 通过日志门面库（类似 Java 的 slf4j）和日志实现库（类似 Java 的 log4j，logback）配合，来灵活的记录日志

而在 Rust 中，异常处理相关的语法有：

* panic!
* RESULT

### a. panic

其实对在线服务来说，panic 其实不怎么用。使用 panic 的例子：

```rust
fn main() {
    panic!("crash and burn");
}
```

* 发生 panic 后，Rust 会 unwind stack 后继续执行。但有 2 个例外的情况 Rust 不会做 unwind：
  * 当遇到一个 panic 后，Rust 在做 drop 时，又触发了一个新的 panic；此时会干掉整个进程！
  * 编译时带上 -C panic=abort 选项，当遇到一个 panic 后，直接干掉整个进程！
* panic 是线程级别的。单个线程里面 panic 了，并不会影响其他线程的正常执行
* 也可以 catch 住 panic，这样线程可以继续执行（参考标准库的：std::panic::catch_unwind）

### b. Result

Rust 使用一个标准库定义的 Enum 类型 Result 来表达函数的正常返回和异常：

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

> 用 enum 的潜台词就是函数返回时，要么返回一个正常的结果，要么遇到一个可恢复异常。

----

分别对正常结果，和异常处理的例子：

```rust
fn main() {
    match get_weather(hometown) {
        Ok(report) => {
            display_weather(hometown, &report);
        }
        Err(err) => {
            println!("error querying the weather: {}", err);
            schedule_weather_retry();
        }
    }
}
```

----

异常类型忽略的写法：

```rust
#![allow(unused)]

// 返回一个 () 或遇到一个异常
fn remove_file(path: &Path) -> Result<()> {}
```

这种写法用来表示这个 Module 中定义了 Result 类型的 **type alias**，省得到处都要写异常的具体类型（一般来说，某个 Module 中的异常都是相同类型的异常）：

```rust
pub type Result<T> = result::Result<T, io::Error>;
```

----

Result 配合 unwrap 的使用：

```rust
#![allow(unused)]

use std::fs::File;

fn main() {
    let f = File::open("hello.txt").unwrap();
}
```

Result 配合 expect 的使用：

```rust
#![allow(unused)]

use std::fs::File;

fn main() {
    let f = File::open("hello.txt").expect("Failed to open hello.txt");
}
```

----

一个把异常传递到调用者的例子：

```rust
use std::io;
use std::io::Read;
use std::fs::File;

fn read_username_from_file() -> Result<String, io::Error> {
    let f = File::open("hello.txt");

    let mut f = match f {
        Ok(file) => file,
        Err(e) => return Err(e), // 第一个可能传递异常的位置
    };

    let mut s = String::new();

    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => Err(e), // 第二个可能传递异常的位置
    }
}
```

上面的例子，可以利用 ? 进行简化（功能完全相同，但代码更整洁）

```rust
use std::io;
use std::io::Read;
use std::fs::File;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

还可以利用链式写法进一步简化：

```rust
use std::io;
use std::io::Read;
use std::fs::File;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut s = String::new();

    File::open("hello.txt")?.read_to_string(&mut s)?;

    Ok(s)
}
```

**注意：** 一个方法如果不返回 Result 类型，就不能使用 ? 来进行简化。

### c. 实现 Error Trait

在很多场景里面，使用 Rust 时，目前最好的实践还是自己实现一个 Error 类型（枚举或结构），也就是实现标准库里面的 Error Trait：

```rust
pub trait Error: Debug + Display {
    /// The lower-level source of this error, if any.
    /// dyn Error 是一个 trait object
    /// trait object 作为 Rust 中的多态，这样既可以无需关注异常的具体类型，也可以通过 source 获取异常时具体的调用链
    /// 如果该错误类型中包含了底层的错误 Err，那么 source 方法应该返回 Some(err), 如果没有返回 None。不重写则默认为 None
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
}
```

一个实现了 source，并利用 source 打印异常堆栈的例子：

```rust
// A new error type, wrapping a sqlx::Error
pub struct StoreTokenError(sqlx::Error);

impl std::error::Error for StoreTokenError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        // The compiler transparently casts `&sqlx::Error` into a `&dyn Error`
        Some(&self.0)
    }
}

fn error_chain_fmt(
    e: &impl std::error::Error,
    f: &mut std::fmt::Formatter<'_>,
) -> std::fmt::Result {
    writeln!(f, "{}\n", e)?;
    let mut current = e.source();
    while let Some(cause) = current {
        writeln!(f, "Caused by:\n\t{}", cause)?;
        current = cause.source();
    }
    Ok(())
}

impl std::fmt::Debug for StoreTokenError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        error_chain_fmt(self, f)
    }
}
```

另外的惯用法是，在一个 Module 中定义一个枚举错误类型来封装多种异常类型：

```rust
#[derive(Debug)]
pub enum SubscribeError {
    ValidationError(String),
    DatabaseError(sqlx::Error),
    StoreTokenError(StoreTokenError),
    SendEmailError(reqwest::Error),
}
```

然后用 From 来把其他错误类型转换为这个 Module 专用的错误类型。例如：

```rust
impl From<sqlx::Error> for SubscribeError {
    fn from(e: sqlx::Error) -> Self {
        Self::DatabaseError(e)
    }
}
```

尽管通过以上代码可以实现一个 Error Trait，但还是太繁琐了。现在流行的做法是用库来自动生成 Error 类型，例如：

* **thiserror**
* **snafu**

具体用法可以参考这些库的文档。

### d. 日志

* Rust 比较流行的「日志门面库」是：log 库（类似 Java 的 slf4j）

```rust
use log::*;
```

## 13. Traits

* 在 Rust 中，Trait 就类似 Java 的 Interface（接口）的作用
* 使用 trait 时必须先 use 它。但 Clone 和 Iterator 是例外，可以不导入也能用，这是因为 Clone 和 Iterator 在 standard prelude 中

> **standard prelude**（标准前置模块）：Rust 会把「标准前置模块」自动导入到所有 module 中；所有可以直接用，无需导入

* trait 作为函数参数的例子（注意 **impl** 关键字的使用）：

```rust
// item 可以是实现了 Summary trait 的类型
pub fn notify(item: impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

* 给一个和上面 **impl** 等价的使用了**泛型参数声明**的写法：

```rust
pub fn notify<T: Summary>(item: T) {
    println!("Breaking news! {}", item.summarize());
}
```

* 指定多个 trait bound 的例子：

```rust
#![allow(unused)]

pub fn notify<T: Summary + Display>(item: T) {}
```

等价：

```rust
#![allow(unused)]

pub fn notify(item: impl Summary + Display) {}
```

* 通过 where 简化的例子：

```rust
#![allow(unused)]

fn some_function<T, U>(t: T, u: U)
    where T: Display + Clone,
          U: Clone + Debug
{}
```

* Rust 中可以使用 trait 来实现多态（polymorphism）的效果。利用 trait 来达到多态的效果有 2 种方法：
  * **Trait Objects**：A reference to a trait type  is called a trait object
    * 翻译：trait object 就是对一个 trait 类型的引用
    * trait object 是 fat pointer，由 2 部分组成：
      * 指向 value 的指针
      * 执行 value 的类型的指针
  * **Generic Type Parameter**：使用泛型作为函数类型参数

给一个 trait object 的例子：

```rust
#![allow(unused)]

fn main() {
    let mut local_file = File::create("hello.txt")?;

    // 函数入参 &mut local_file 是一个 &mut File 类型的引用
    // 而 say_hello 的入参定义是一个 trait object：&mut dyn Write
    // 这里隐含了一个 Rust 的自动转换：&mut File => &mut dyn Write
    say_hello(&mut local_file)?;
}
```

给一个泛型作为函数类型参数的例子：

```rust
#![allow(unused)]

use std::hash::Hash;
use std::fmt::Debug;

// 这里的类型 T 必须同时是 3 个 traits 的实现：Debug，Hash，Eq
fn top_ten<T: Debug + Hash + Eq>(values: &Vec<T>) {}
```

* 利用 trait 来达到多态的效果的 2 种方法的对比：
  * Trait Object：
    * 如果一个集合中需要包含多种不同类型的元素，那么要用 trait object
    * 另外，使用 trait object 可以节约编译时间（因为 trait object 是在运行时才确认指向的具体对象）；就是所谓的「dynamic dispatch」（dyn 关键字）
  * Generic Type Parameter：
    * 泛型类型参数的运行效率更高（泛型是在编译时展开，所以运行时无额外的代价）

----

其他一些 trait 相关的注意事项：

* 可以给  trait 提供一个 default 实现
* Rust 允许在任何类型实现任何 trait；也就是说，我们甚至可以在基础类型（举例： char 类型）上实现新的 trait
* 也可以在某个泛型类型上实现 trait。给个例子：

```rust
#![allow(unused)]

// 在 Write 这个泛型类型上实现一个 WriteHtml trait
impl<W: Write> WriteHtml for W {
    fn write_html(&mut self, html: &HtmlDocument) -> io::Result<()>
    {}
}
```

* **orphan rule**：can’t implement external traits on external types
  * 不能给对外部类型实现外部 traits；这样 Rust 才能确保当前的这个 trait 的实现是「唯一」的（在一个 crate 内，一个 trait 不能同时有 2 个实现）
  * 举个例子：在我们自己的 crate 里面不能对 Vec\<T\> 实现 Display trait（标准库里面已经在 Vec\<T\> 上实现了 Display trait）

下面的小节会介绍一些常用 Traits

### a. Drop

* 当一个 value 的 owner 失效后，Rust 会 **Drop** 这个 value。具体成来说可能有几种场景：
  * 变量的 scope 结束
  * vector 中的一个元素被删除
* 通常来说，Rust 已经提供了 Drop Trait 的实现（例如 vector/标准库中的文件相关）
* 但我们也可以自己重新实现 Drop Trait。但一旦实现了 Drop Trait，就不能再实现 Copy Trait
* 另外一个值得注意的是，Drop 只会被调用一次（move 的时候其实不会调用 Drop，只有 value 真正失效时，才会调用 Drop）

### b. Sized

* A sized type is one whose values all have the same size in memory
  * 翻译：该类型中所有的 values 的 size 必须是相同
* 实现了 std::marker::Sized 这个 trait。Rust 已经都实现了，开发者不能再提供自己的实现
  * 所以唯一的用途就是作为类型的 bound，比如使用泛型时的 bound
* enum 也是 sized 类型：尽管 enum 中各个变量的类型可能不同，但 Rust 会按最大所需的 size 给每个变量分配空间

尽管 Rust 中绝大多数类型是 Sized 的，但有几种类型不是 Sized：

* slice type str (note, **without an &**) is unsized
* Array slice types like [T] (again, **without an &**)
* **dyn type**：the referent of a **trait object**

> unsized 类型不能作为函数参数，也不能直接用一个变量来 store 这个类型的值。只能通过引用或者指针来操作 unsized 类型的值

### c. Clone

Clone trait 定义如下：

```rust
trait Clone: Sized {
    fn clone(&self) -> Self;
    fn clone_from(&mut self, source: &Self) {
        *self = source.clone()
    }
}
```

* Clone 必然是工作在 Sized 类型上（该类型的大小可确定）
* 大多数时候，clone_from （clone 出来源的值，并替换掉 self）使用默认实现就行了：直接用 clone 来实现

### d. Copy

Rust 中，赋值时，默认的行为时 move。但可以通过实现 **Copy trait** （std::marker::Copy）来替换默认行为。Copy trait 定义如下：

```rust
trait Copy: Clone {}
```

* **限制**：实现了 Drop trait 的类型不能被 Copy
* \#[derive(Copy, Clone)] 的使用

### e. Deref and DerefMut

先回顾一下 Rust 中的「**解引用**」操作符号 * 的用法：

```rust
fn main() {
    let x = 5;
    // & 引用就是 Rust中的安全指针
    let y = &x;

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

通过实现 std::ops::Deref 和 std::ops::DerefMut 这 2 个 trait 来定义 * 这个「**解引用**」操作符号。例如：

* Box\<T> 实现了 Deref 和 DerefMut。那么对于 Box\<T> 的一个值 b，那么 *b 就是 refer to T 的值

给个例子：

```rust
fn main() {
    let x = 5;
    // Box<T> 就是 Rust 中的智能指针
    let y = Box::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y);
}
```

Deref 和 DerefMut 这 2 个 trait 大概定义如下：

```rust
trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}

trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

* 其中，Target 是可以通过 Self 拿到的返回，同时 2 者生命周期保持一致
* 而且，DerefMut 需要实现 Deref
* **deref coercions**: one type is being “coerced” into behaving as another。也就是自动类型转换，例如：
  * Rc\<T> 实现了 Deref<Target=T>。所以 Rc\<String> 的值 r：r.find('?') 等价于 (*r).find('?')
  * String 实现了 Deref<Target=str>。所以可以 coerce 一个 &String 的值到 &str
  * Vec\<T> 实现了 Deref<Target=[T]>。所以可以把 bytes vector 传给一个入参是 slice &[u8] 的函数

### f. Default

实现了 Default（std::default::Default）这个 trait，可以通过实现这个 trait 来给类型设置默认值。trait 定义如下：

```rust
trait Default {
    fn default() -> Self;
}
```

给个例子：

```rust
impl Default for String {
    fn default() -> String {
        String::new()
    }
}
```

### g. AsRef and AsMut

这 2 个 traits 定义如下：

```rust
trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}

trait AsMut<T: ?Sized> {
    fn as_mut(&mut self) -> &mut T;
}
```

当需要借用（borrow）一个 &T 的时候，可以直接传入实现了这 2 个 traits 的类型的值。给个例子：

```rust
#![allow(unused)]

// 该函数需要的入参是：&Path（借用 Path 的值）
fn open(path: &Path) -> Result<File> {}
```

事实上，该函数的声明如下：

```rust
#![allow(unused)]

// 入参是一个 AsRef<Path>，这样所有实现了 AsRef<Path> 的类型的值都可以传入这个函数（例如：String 和 str）
fn open<P: AsRef<Path>>(path: P) -> Result<File> {}
```

这种机制提供了引用间的类型转换，有点类似 C++ 中的函数参数的重载（overload）：

* 如果类型 U 实现了 AsRef\<T>，则 as_ref 可以实现 &U 到 &T 的转换
* 如果类型 U 实现了 AsMut\<T>，则 as_ref 可以实现从 &U 到 &mut T 的转换

给个例子：

```rust
fn print_ref(v: impl AsRef<str>) {
    println!("{}", v.as_ref());
}

#[allow(dead_code)]
enum Msg {
    Hello,
    World,
}

impl AsRef<str> for Msg {
    fn as_ref(&self) -> &str {
        match self {
            Msg::Hello => "hello",
            Msg::World => "world",
        }
    }
}

fn main() {
    // s1 是 &str 类型, str 类型实现了 AsRef<str>，&str 也实现了 AsRef<str>
    let s1 = "hello";
    // s2 是 String 类型, 实现了 AsRef<str>, &String 也实现了 AsRef<str>
    let s2 = String::from("word");
    // msg 是 Msg 类型, 实现了 AsRef<str>, &Msg 也实现了 AsRef<str>
    let msg = Msg::Hello;

    // str 类型实现了 AsRef<str>
    print_ref(s1); // hello
    // &str 类型实现了 AsRef<str>
    print_ref(&s1); // hello

    // &String 实现了 AsRef<str>
    print_ref(&s2); // world
    // String 实现了 AsRef<str>
    print_ref(s2); // world

    // &Msg 实现了 AsRef<str>
    print_ref(&msg); // hello
    // Msg 实现了 AsRef<str>
    print_ref(msg); // hello
}
```

### h. Borrow and BorrowMut

Borrow 的定义：

```rust
trait Borrow<Borrowed: ?Sized> {
    fn borrow(&self) -> &Borrowed;
}
```

从其定义上看，Borrow 和 AsRef 非常类似。用一个例子说明 2 者的区别：

* String 实现了AsRef\<str>，AsRef\<[u8]>，和 AsRef\<Path>，但这 3 个类型（str，u8，Path） 的 hash 值各不相同，不能保持一致。其中只有 &str slice 的 hash 值能和原类型 String 的 hash 值保持一致，所以 String 类型只会实现 Borrow\<str>。

Borrow trait 主要用于 hash 表，或其他的集合类型里的 key 值的 hash 值计算。给个例子：

```rust
#![allow(unused)]

impl<K, V> HashMap<K, V> where K: Eq + Hash {
    fn get(&self, key: &K) -> Option<&V> {
        // ...
    }
}
```

上面这个例子里面的实现是需要用一个 k 来获取 Map 的 v，但这里的 k 是 String 类型，传入了一个 &K。调用它的一个例子： hashtable.get( & "twenty-two".to_string())

该调用的开销如下：

* 分配空间（"twenty-two"）
* 拷贝字符串（"twenty-two".to_string()）
* 执行 drop，销毁 String（调用结束）

为了更高效，可以改进为：入参是类型 Q，只要类型 Q 满足 Eq + Hash。同时，Map 中的 K 需要满足约束：实现了 Borrow\<Q>：

```rust
#![allow(unused)]

impl HashMap<K, V> where K: Eq + Hash
{
    fn get<Q: ?Sized>(&self, key: &Q) -> Option<&V>
        where K: Borrow<Q>,
              Q: Eq + Hash
    {
        // ... 
    }
}
```

简单点说，就是利用 Borrow 这个 trait 来对类型 K 进行约束。

### i. From and Into

* 类型转换：把一个类型的值转换为另外一个类型的值
  * 和 AsRef 不同，AsRef 进行的引用间的转换；而 From 和 Into 是值之间的转换

定义如下：

```rust
trait Into<T>: Sized {
    fn into(self) -> T;
}

trait From<T>: Sized {
    fn from(other: T) -> Self;
}
```

* 和 AsRef 差不多，Into 也有点像 overload（重载）
* From 常用来进行值的初始化

## 14. IO 标准库

Rust 的 IO 标准库重点就是 3 个 traits：

* Read：统称 Reader，具体的实现包括：Stdin，File，TcpStream
* BufRead：就是 buffered Reader，继承了 Read trait；具体的实现包括：BufferReader，Cursor，StdinLock
* Write：统称 Writer，具体的实现包括：Stdout，Stderr，File。。。

### a. Read

这里只重点介绍一个重点方法：

* reader.read(&mut buffer)：把 byte 数据读到指定的 buffer 中（buffer 空间事先已经分配好了），然后返回实际读到的数据的长度。如果出错的话，就返回 io::Error

### b. BufRead

BufRead 的重点方法：

* reader.read_line(&mut line)：按 line 把数据读入到一个 String 类型中（结果包括行分割符：\n，\r\n）。返回本次读到的 byte 数
* reader.lines()：返回一个 iterator，然后通过迭代拿到一个 io::Result\<String>；同时 \n 不会被放到读到的 String 中

### c. Write

* 利用 write!() 和 writeln!() 这 2 个宏来使用 Write
* Write 的重点方法：
  * write(&buf)：把 byte 数据写出。成功的话，返回写入的大小；失败的话，返回 Error
  * flush()：把数据 flush 到写入目标
* Write 本身可以创建对应的 BufWriter。例如：BufWriter::new(writer)；并且 BufWriter 也继承了 Write trait
  * BufWriter 被 drop 的时候，会自动把被 buffer 的数据写出。但如果出现 Error，不保证能成功，所以有可能需要显示调用 flush

## 15. 线程

（待）

## 16. 异步

### a. Future

* Future。Rust 中，Future 是一个 trait，其定义大致（简化版）如下：

```rust
trait Future {
    type Output;
    fn poll(&mut self) -> Poll<Self::Output>;
}
```

* 其中 Poll 的定义大致如下：

```rust
enum Poll<T> {
    // future 完成时返回 Ready
    Ready(T),
    // future 还没有完成的话，返回 Pending
    Pending
}
```

* When a future eventually returns Poll::Ready(T), we say that the future resolves into a T
  * 翻译：当一个 future 对象完成后，会返回 Poll::Ready(T)，这时我们可以说这个 future 对象被 resolve 成一个类型 T 的返回值

### b. async/.await

* async/.await 语句。其中 async 是在 2 个主要场景中被使用，在这 2 个场景中，都会返回一个 Future trait：
  * async fn
  * async blocks（返回这个代码块最后一个表达式的值的 future）

例子如下：

```rust

// `foo()` returns a type that implements `Future<Output = u8>`.
// `foo().await` will result in a value of type `u8`.
async fn foo() -> u8 { 5 }

fn bar() -> impl Future<Output=u8> {
    // This `async` block results in a type that implements
    // `Future<Output = u8>`.
    async {
        let x: u8 = foo().await;
        x + 5
    }
}

```

* 说明：
  * 调用 async 的函数体或者代码块，并不会被执行，而是立即返回一个 future 给调用者
  * 当调用者在 future 上用 .await，这段代码块才会被执行
  * 如果执行 Future 的时候被阻塞（blocked），会 yield （让出）当前线程的控制
  * 直到阻塞（blocked）结束，线程调度器（executor）会恢复并继续执行这个 future，如果最后完成的话，.await 完成 future，并返回 future 结果
    * **NOTE**：如果 executor 有多个线程，那么 future 恢复执行后有可能会到另外一个线程里去执行，需要注意线程安全（互斥和死锁）

  ----

* **asynchronous function** 的惯用法：

  * 用 async fn 来定义一个函数
  * 然后在这个函数内部，需要使用「异步」版本的 io 操作（由「异步库」提供，例如：async_std 或 tokio）
    * 「异步」版本的 io 操作会使用 await 表达式
    * 「异步」版本的 io 操作，不会直接返回 io 操作结果，而是会返回一个 **future**
  * 当别人调用这个「**asynchronous function**」时，也会立即返回一个这个函数返回值的 future 给调用者
    * **NOTE**：调用时，这个「**asynchronous function**」的函数体并**没有**开始被执行
  * 只有当调用者**第一次** poll 这个 future 时，「**asynchronous function**」的函数体才会开始执行
  * 调用后，会返回 2 种结果给调用者（「异步」版本的 io 操作之后的 await 表达式）：
    * Poll::Pending
    * Poll::Ready

  ----

* 生命周期
  * async fn 的入参如果是  non-'static 参数（例如：引用参数），那么这个 async fn 返回的 Future 的生命周期需要被限制在函数入参的生命周期之内
    * 换句话说，当在这个 future 上做 .await 的时候，当初调用这个 async fn 时的入参的生命周期必须仍然有效

  ----

* async move 代码块：

```rust
/// `async move` block:
///
/// 如果某个 `async move` block 使用了外部的 my_string 这个变量后, 其他的 `async move` block 就不能使用这个变量了

fn move_block() -> impl Future<Output=()> {
    let my_string = "foo".to_string();
    async move {
        // ...
        println!("{}", my_string);
    }
}
```

### c. 异步运行时

运行异步代码目前由库来实现，例如：

* async-std
* tokio

这里我们先用 async-std 来举例怎么运行一段异步代码：

```rust
extern crate async_std;

use async_std::{fs::File, io, prelude::*, task};

async fn read_file(path: &str) -> io::Result<String> {
    let mut file = File::open(path).await?;
    let mut contents = String::new();
    file.read_to_string(&mut contents).await?;
    Ok(contents)
}

fn main() {
    let reader_task = task::spawn(
        async {
            let result = read_file("data.csv").await;
            match result {
                Ok(s) => println!("{}", s),
                Err(e) => println!("Error reading file: {:?}", e)
            }
        });
    println!("Started task!");
    task::block_on(reader_task);
    println!("Stopped task!");
}
```

2 个步骤来运行异步代码：

* 先用 task::spawn 启动一段异步代码的运行，返回一个新的 future（async_std::task::JoinHandle 类型）
* 然后 task::block_on 来等待这个 JoinHandle  类型的 future 运行完成，最后从这个 future 拿到最后的运行结果

## 17. Closure

* closure 就是可以把函数作为参数来使用。可以把 closure 作为参数的一些库方法：
  * Iterator
  * thread::spawn
* closure 会自动 borrow 入参的值（默认创建一个指向入参值的引用）
* 只要 closure 自己会在该引用指向的值的前面被 dropped，那么就是符合生命周期规则，没有问题

一个简单的例子：

  ```rust
fn main() {
    let s = "hello";

// f 是一个 closure，它自动 borrow 入参 s
    let f = || {
        println!("{}", s);
    };

// f 这个 closure 在 s 之前被 dropped，所以这段代码是 OK 的
    f();
}
  ```

* 但如果不能保证 closure 自己会在该引用指向的值的前面被 dropped，不能通过编译
* 此时可以考虑使用 move 关键字，使用 move 关键字后，closure 会把入参的值 move 到 closure 内部，并拿到这个值的 ownership
  * 然后当这个 closure 被 dropped 的时候，这个参数的值也会被 dropped

一个简单的例子：

```rust
fn main() {
    let s = "hello";

// f 是一个 closure，它 move 值 s，并拿到 s 的 ownership
    let f = move || {
        println!("{}", s);
    };

    f();

// 之后，当 f 这个 closure 被 dropped，s 也会被 dropped
// 如果之后还需要使用这个 s，只能在 move 之前 clone 出一份
}
```

一个进行 clone 之后再使用 closure 的简单例子：

```rust
fn main() {
    let s = "hello";
    let s2 = s.clone();

// move 的时候，用 s2
    let f = move || {
        println!("{}", s2);
    };

    f();

// 之后，当 f 这个 closure 被 dropped，s2 会被 dropped
// 而 s 还可以继续被使用
}
```

----

用一些更复杂的例子说明更多的细节：

```rust
struct City {
    name: String,
    population: i64,
    country: String,
}

fn sort_cities(cities: &mut Vec<City>) {
    cities.sort_by_key(|city| -city.population);
}
```

* 按人口降序排序
* 有点像其他语言（Java）中的匿名函数
* 但 Rust 的 closure 和其他语言的匿名函数又有区别

----

下面的例子，说明了 Closure 相关的生命周期：

```rust
/// Sort by any of several different statistics.
fn sort_by_statistic(cities: &mut Vec<City>, stat: Statistic) {
    // Rust 自动 borrow（自动有一个到 stat 的引用）
    cities.sort_by_key(|city| -city.get_statistic(stat));
}
```

----

线程相关的复杂例子：

```rust
use std::thread;

fn start_sorting_thread(mut cities: Vec<City>, stat: Statistic)
                        -> thread::JoinHandle<Vec<City>>
{
    // 定义一个 Closure
    let key_fn = |city: &City| -> i64 { -city.get_statistic(stat) };

    // spawn 新的线程
    thread::spawn(|| {
        cities.sort_by_key(key_fn);
        cities
    })
}
```

Rust 会提示这段代码编译错误。错误的根源如下：

* 这个新的线程的工作完成之前，stat 和 cities 这 2 个变量都必须存活；但上面的代码按 borrow 的话，没有这个保证

解决方法是利用 **move** 关键字告诉 Rust 把 stat 和 cities 这 2 个变量 move 到 closure（而不是像前个例子那样走 borrow 逻辑）。代码如下：

```rust
fn start_sorting_thread(mut cities: Vec<City>, stat: Statistic)
                        -> thread::JoinHandle<Vec<City>>
{
    // 利用 move 关键字，key_fn closure 获取到 stat 的 ownership
    let key_fn = move |city: &City| -> i64 { -city.get_statistic(stat) };

    // 利用 move 关键字，该 closure 获取到 key_fn 和 cities 的 ownership
    thread::spawn(move || {
        cities.sort_by_key(key_fn);
        cities
    })
}
```

* 【回顾一下】：borrow 和 move 的本质区别在于 ownership 是否改变

## 18. Iterator

### a. 概念

先看一个 for 循环的例子：

```rust
#![allow(unused)]

fn main() {
    let v = vec![6, 7, 8, 9];

    // 实际上 for 并不是作用在 v 上的，实际上是作用在一个 iterator 上的
    for num in v {
        println!("{}", num);
    }
}
```

实际上 for 并不是作用在 v 上的，实际上是作用在一个 iterator 上的，for 语句会自动调用 vector 的 into_iter 方法：

```rust
#![allow(unused)]

fn main() {
    let v = vec![6, 7, 8, 9];

    // 显示的调用 vector 的 into_iter 方法返回一个 iterator，之后会 returns owned item
    for num in v.into_iter() {
        println!("{}", num);
    }
    // for 语句后 vector 不再有效（其中 item 的 ownership 已经转移）
}
```

上面这个例子中，使用了 into_iter 方法后，vector 中 item 的 ownership 已经转移，所以 for 语句后变量 v 不再有效。

再看一个等价于上面 for 循环的例子：

```rust
#![allow(unused)]

fn main() {
    let v = vec![6, 7, 8, 9];

    // vector 的 into_iter 方法返回一个 iterator，之后会 returns owned item
    // 然后使用 for_each 顺序消费这个 iterator 产出的序列中的值
    v.into_iter().for_each(|num| println!("{}", num));

    // vector 不再有效（其中 item 的 ownership 已经转移）
}
```

----

**迭代器**（iterator）：对某个序列中的项进行某些处理时，可以使用 iterator：

* Rust 中，iterator 也是一个值，利用一个 iterator 能产出一个序列的值。然后再对这个序列中的的 item 进行处理（类似循环）

* Rust 中，iterator 就是实现了 Iterator 这个 trait 的类型：

```rust
#![allow(unused)]

fn main() {
    pub trait Iterator {
        // 这里的 Item 类型是 Iterator 返回的序列中元素的类型
        type Item;

        // Next 一次返回迭代器中的一项，封装在 Some 中，当迭代器结束时，它返回 None
        fn next(&mut self) -> Option<Self::Item>;

        // 此处省略了方法的默认实现
    }
}

```

* 注意：next 需要一个可变引用（&mut）；也就是说，每次调用 next 会消费（consume）该序列中的一项，实际上 iterator 的状态被改变（mut）了
* Rust 中的很多集合类型（比如前面例子中的 vector），会默认实现另外一个 trait：IntoIterator，然后就可以调用这个 trait 的方法 into_iter 得到这个集合上的 iterator。不过很多集合类型默认实现的 into_iter 会转移 item 的 ownership，具体看文档

最后再总结几个相关概念：

* 一个 iterator 是实现了 Iterator trait 的类型
* 某个类型实现了 IntoIterator trait，那么就可以调用 into_iter 获取到作用在这个类型上的 iterator
  * for 语句会自动调用 into_iter 获取 iterator，然后在 iterator 上进行循环
* iterator 能产出一个序列（然后可以处理这个序列中的各个值）
* iterator 本身也是一个值
* 不断调用 next() 方法，对 iterator 产出的序列中的值进行处理叫做消费（consume）

### b. 创建 iterator

按上一节说的，可以使用 into_iter() 方法 来获取到一个类型上的 iterator。但很多时候，into_iter 会转移 ownership，所以又提供了另外 2 个方法来获取 iterator：

* iter() 方法：通过这个 iterator 获取到序列中 item 的**不变引用**，对 item 只读不修改（不转移 item 的 ownership）
* iter_mut() 方法：通过这个 iterator 获取到序列中 item 的**可变引用**，可以修改 item（不转移 item 的 ownership）

一个调用 iter() 方法的例子：

```rust
#![allow(unused)]

fn main() {
    let v = vec![4, 20, 12, 8, 6];
    // iterator 本身可变（mut），但处理其中的 item 时，是一个不变引用：&i32
    let mut iterator = v.iter();
    assert_eq!(iterator.next(), Some(&4));
    assert_eq!(iterator.next(), Some(&20));
    assert_eq!(iterator.next(), Some(&12));
    assert_eq!(iterator.next(), Some(&8));
    assert_eq!(iterator.next(), Some(&6));
    assert_eq!(iterator.next(), None);
}
```

### c. 使用 iterator

使用 iterator 需要 Iterator Consumer 和 Iterator Adaptor。

* Iterator Consumer：可以通过 Iterator Consumer 不断调用 next() 方法，对一个 iterator 进行消费。例如：
  * sum，max，min
  * fold
  * for_each
  * collect

参考具体文档熟悉更多的 consumer。

* Iterator Adaptor：把当前 iterator 变为另外类型的 iterator。例如：
  * map
  * filter
  * filter_map 和 flat_map

参考具体文档熟悉更多的 Iterator Adaptor。

## 19. 宏

Rust 中的宏分以下几种：

* declarative macros：使用 macro_rules! 来定义宏（据说未来会有更好的机制来替换 macro_rules!）
* procedural macros。翻译成「过程宏」，用代码生成代码。又分成 3 种：
  * derive。derive 只能用于结构体和枚举
  * attribute（Attribute-like）。可以用于任何地方。
  * function（function-like）

### a. macro_rules

给个例子（利用宏 vec! 来初始化 vector）：

```rust
#![allow(unused)]

fn main() {
    let v: Vec<u32> = vec![1, 2, 3];
}
```

vec! 定义如下：

```rust
#![allow(unused)]

fn main() {
    #[macro_export]
    // vec 是宏的名字
    macro_rules! vec {
      ( $( $x:expr ),* ) => {
          {
              let mut temp_vec = Vec::new();
              $(
                  temp_vec.push($x);
              )*
              temp_vec
          }
      };
    }
}
```

* 需要用 \#[macro_export] 属性来标识该宏可用
* ( $( $x:expr ),* ) 用于模式匹配。例如，可以匹配上面例子中的：[1, 2, 3] 会被匹配出 3 个值：1、2、3，对应到宏定义中的 $x
* => 类似 match 语句，当模式匹配成功以后，就执行 => 后面的代码

最终，会生成类似以下的代码：

```rust
fn main() {
    let mut temp_vec = Vec::new();
    temp_vec.push(1);
    temp_vec.push(2);
    temp_vec.push(3);
    temp_vec
}
```