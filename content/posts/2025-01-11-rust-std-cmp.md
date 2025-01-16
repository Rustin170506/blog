---
title: 'Rust equality and ordering'
date: 2025-01-11
description: 'Exploring the PartialEq, Eq, PartialOrd, Ord, and Reverse traits in Rust'
tags: ["Rust"]
categories: ["Rust"]
---

# std::cmp

The `std::cmp` module offers several traits for comparing values. It allows us to check for equality and determine the ordering of values. Unlike some other languages where the compiler automatically provides these implementations, Rust requires us to explicitly implement or derive these traits.

Implementing these traits can sometimes be challenging. In this post, we will explore the `PartialEq`, `Eq`, `PartialOrd`, `Ord`, and `Reverse` traits in detail.

## PartialEq

Many might assume that `PartialEq` is termed "partial" because it only compares certain parts of a type. However, the "partial" in `PartialEq` actually indicates that the equality comparison does not need to be reflexive `(a == a)`. In other words, `PartialEq` does not require a type to always be equal to itself.

The `PartialEq` trait is defined as follows:

```rust
pub trait PartialEq<Rhs = Self>
where
    Rhs: ?Sized,
{
    // Required method
    fn eq(&self, other: &Rhs) -> bool;

    // Provided method
    fn ne(&self, other: &Rhs) -> bool { ... }
}
```

The `PartialEq` trait requires the implementation of the `eq` method, which checks if two values are equal. By default, the `ne` method is provided, which checks if two values are not equal. Essentially, `eq` corresponds to the `==` operator, while `ne` corresponds to the `!=` operator.

At first glance, it might seem that the `PartialEq` trait is sufficient for determining equality between values. However, from a mathematical standpoint, equality should be reflexive, meaning a value must always be equal to itself.

It may seem intuitive that all types should exhibit reflexive equality, there are exceptions. For instance, the floating-point types `f32` and `f64` include a special value called `NaN` (Not a Number). `NaN` is unique in that it is not equal to itself, thus violating the reflexivity property. Consequently, `PartialEq` does not enforce reflexivity by default.

According to the [IEEE 754] standard, comparisons involving `NaN` always return false, even when comparing `NaN` with itself. This means that if `x` is `NaN`, the following comparisons will all return false:

- `x == x`
- `x < x`
- `x > x`

However, the comparison `x != x` will return true.

So in some of data structures, like `HashSet` and `HashMap`, the `PartialEq` trait is not sufficient for some of their operations. For example, the `contains_key` method of `HashMap` requires the `Eq` trait, which enforces reflexivity.

```rust
pub fn contains_key<Q>(&self, k: &Q) -> bool
where
    K: Borrow<Q>,
    Q: Hash + Eq + ?Sized,
```

As a result, it is impossible to create a `HashMap` with `f32` or `f64` keys because they do not implement the `Eq` trait.

```rust
let mut map: HashMap<f64, String> = HashMap::new();

map.insert(3.14, String::from("Pi"));
map.insert(2.71, String::from("Euler"));
```

Attempting to compile this code will result in an error because `f64` does not satisfy the `Eq` trait requirements.

```rust
error[E0599]: the method `insert` exists for struct `HashMap<f64, String>`, but its trait bounds were not satisfied
 --> src/main.rs:8:9
    |
8 |     map.insert(3.14, String::from("Pi"));
    |         ^^^^^^
    |
    = note: the following trait bounds were not satisfied:
                    `f64: Eq`
                    `f64: Hash`
```

So Rust provides the `Eq` trait, which extends the `PartialEq` trait by enforcing reflexivity. We will discuss the `Eq` trait in more detail later in this post.

To implement the `PartialEq` trait, you can either do it manually or use the `derive` attribute.

For example, you can derive the `PartialEq` trait for a custom type like this:

```rust
#[derive(PartialEq)]
struct Point {
    x: i32,
    y: i32,
}
```

Expanding the `derive` attribute generates the following implementation:

```rust
struct Point {
    x: i32,
    y: i32,
}

#[automatically_derived]
impl ::core::marker::StructuralPartialEq for Point {}

#[automatically_derived]
impl ::core::cmp::PartialEq for Point {
    #[inline]
    fn eq(&self, other: &Point) -> bool {
        self.x == other.x && self.y == other.y
    }
}
```

As shown, the `PartialEq` trait is implemented for the `Point` struct. The `eq` method checks if the `x` and `y` fields of two `Point` instances are equal.

For enums, the `derive` attribute generates an implementation that checks if the variants are equal:

```rust
#[derive(PartialEq)]
enum Direction {
    Up,
    Down,
    Left,
    Right,
}
```

The `derive` attribute generates the following implementation:

```rust
enum Direction {
    Up,
    Down,
    Left,
    Right,
}

#[automatically_derived]
impl ::core::marker::StructuralPartialEq for Direction {}

#[automatically_derived]
impl ::core::cmp::PartialEq for Direction {
    #[inline]
    fn eq(&self, other: &Direction) -> bool {
        let self_discr = ::core::intrinsics::discriminant_value(self);
        let other_discr = ::core::intrinsics::discriminant_value(other);
        self_discr == other_discr
    }
}
```

You can also implement the `PartialEq` trait manually:

```rust
impl PartialEq for Point {
    fn eq(&self, other: &Self) -> bool {
        self.x == other.x && self.y == other.y
    }
}
```

As mentioned earlier, the term "partial" in `PartialEq` does not imply that only part of a struct is compared. Moreover, implementing the `PartialEq` trait for only some fields of a type is not recommended. Let's consider the following example.

From the definition of the `PartialEq` trait, we can see that it takes a type parameter `Rhs`, which defaults to `Self`. This means you can use the `PartialEq` trait to compare two different types.

```rust
// Define clothing types, deriving PartialEq to allow comparisons
#[derive(PartialEq)]
enum ClothingType {
    Shirt,
    Pants,
    Jacket,
}

// Define a Clothing struct with an ID and a clothing type
struct Clothing {
    id: i32,
    clothing_type: ClothingType,
}

// Allow comparisons between Clothing and ClothingType
impl PartialEq<ClothingType> for Clothing {
    fn eq(&self, other: &ClothingType) -> bool {
        self.clothing_type == *other
    }
}

// Allow comparisons between ClothingType and Clothing
impl PartialEq<Clothing> for ClothingType {
    fn eq(&self, other: &Clothing) -> bool {
        *self == other.clothing_type
    }
}

fn main() {
    let my_clothing = Clothing {
        id: 101,
        clothing_type: ClothingType::Shirt,
    };

    // Check if the clothing item is a shirt
    assert!(my_clothing == ClothingType::Shirt);
    // Check if the clothing item is not a jacket
    assert!(ClothingType::Jacket != my_clothing);
}
```

In this example, we define a `Clothing` struct that includes an ID and a `ClothingType` field. We then implement the `PartialEq` trait for both `Clothing` and `ClothingType`, enabling comparisons between these two types.
However, the above example can be problematic as it may violate the transitivity property of equality. For instance, consider the following code:

```rust
// Define clothing types, deriving PartialEq to allow comparisons
#[derive(PartialEq)]
enum ClothingType {
    Shirt,
    Pants,
    Jacket,
}

// Define a Clothing struct with an ID and a clothing type, deriving PartialEq
#[derive(PartialEq)]
struct Clothing {
    id: i32,
    clothing_type: ClothingType,
}

// Allow comparisons between Clothing and ClothingType
impl PartialEq<ClothingType> for Clothing {
    fn eq(&self, other: &ClothingType) -> bool {
        self.clothing_type == *other
    }
}

// Allow comparisons between ClothingType and Clothing
impl PartialEq<Clothing> for ClothingType {
    fn eq(&self, other: &Clothing) -> bool {
        *self == other.clothing_type
    }
}

fn main() {
    let c1 = Clothing {
        id: 101,
        clothing_type: ClothingType::Shirt,
    };
    let c2 = Clothing {
        id: 102,
        clothing_type: ClothingType::Shirt,
    };

    assert!(c1 == ClothingType::Shirt);
    assert!(ClothingType::Shirt == c2);

    // Check if the clothing items are equal
    assert!(c1 == c2); // This assertion will fail
}
```

In this scenario, equality comparisons can lead to unexpected results, breaking the transitivity rule. Transitivity requires that if `a == b` and `b == c`, then `a == c` must also hold true. Violating this property can cause logical inconsistencies and bugs in your code.

In most cases, it is advisable to derive the `PartialEq` trait for a struct or enum. If you choose to implement the trait manually, be careful to maintain the transitivity property to avoid logical inconsistencies.

## Eq

The `Eq` trait extends the `PartialEq` trait by enforcing reflexivity. It is defined as follows:

```rust
pub trait Eq: PartialEq { }
```
The `Eq` trait does not introduce any new methods beyond those in `PartialEq`. Instead, it acts as a marker to signify that a type adheres to the reflexivity property of equality. This means that any type implementing `Eq` must ensure that every instance of the type is equal to itself.

It's important to note that the Rust compiler cannot automatically verify the reflexivity property. Therefore, it is the programmer's responsibility to ensure that the `Eq` trait is implemented correctly and that the reflexivity property is upheld.

Similar to the `PartialEq` trait, you can derive the `Eq` trait for a custom type:

```rust
#[derive(Eq, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}
```

Note that you must also derive the `PartialEq` trait because `Eq` depends on it.

## PartialOrd

The `PartialOrd` trait is used to compare values and determine their ordering. It is defined as follows:

```rust
pub trait PartialOrd<Rhs = Self>: PartialEq<Rhs>
where
    Rhs: ?Sized,
{
    // Required method
    fn partial_cmp(&self, other: &Rhs) -> Option<Ordering>;

    // Provided methods
    fn lt(&self, other: &Rhs) -> bool { ... }
    fn le(&self, other: &Rhs) -> bool { ... }
    fn gt(&self, other: &Rhs) -> bool { ... }
    fn ge(&self, other: &Rhs) -> bool { ... }
}
```
The `lt`, `le`, `gt`, and `ge` methods of this trait allow comparisons using the `<`, `<=`, `>`, and `>=` operators, respectively.

The `partial_cmp` method returns an `Option<Ordering>` enum, which can be `Some(Ordering::Less)`, `Some(Ordering::Equal)`, `Some(Ordering::Greater)`, or `None`.

You might wonder why the `PartialOrd` trait returns an `Option<Ordering>` instead of an `Ordering` directly. The reason is similar to the `PartialEq` trait: it accounts for cases where values cannot be compared, such as `NaN` in floating-point numbers. Thus, `partial_cmp` returns `None` when a comparison is not possible.

```rust
fn main() {
    let x = f32::NAN;
    let y = 5.0;

    assert_eq!(x.partial_cmp(&y), None);
    assert_eq!(y.partial_cmp(&x), None);
}
```

To implement the `PartialOrd` trait, you can either do it manually or derive it using the `derive` attribute.

For example, you can derive the `PartialOrd` trait for a struct like this:

```rust
#[derive(PartialEq, PartialOrd)]
struct Point {
    x: i32,
    y: i32,
}
```

Expanding the `derive` attribute generates the following implementation:

```rust
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::rust_2021::*;
#[macro_use]
extern crate std;
struct Point {
    x: i32,
    y: i32,
}

... // Implementation of PartialEq

#[automatically_derived]
impl ::core::cmp::PartialOrd for Point {
    #[inline]
    fn partial_cmp(&self, other: &Point)
        -> ::core::option::Option<::core::cmp::Ordering> {
        match ::core::cmp::PartialOrd::partial_cmp(&self.x, &other.x) {
            ::core::option::Option::Some(::core::cmp::Ordering::Equal) =>
                ::core::cmp::PartialOrd::partial_cmp(&self.y, &other.y),
            cmp => cmp,
        }
    }
}
```

The comparison follows a [lexicographical] order, first comparing the `x` fields and then the `y` fields top to bottom.

For enums, variants are ordered by their discriminant values:

```rust
#[derive(PartialEq, PartialOrd)]
enum Direction {
    Up,
    Down,
    Left,
    Right,
}

assert!(Direction::Up < Direction::Down);
```

The values of the variants are assigned discriminant values by default, starting from `0` for the first variant and incrementing by `1` for each subsequent variant.

You can verify the discriminant values of the variants using the `std::mem::discriminant` function:

```rust
use std::mem::discriminant;

let up = Direction::Up;
let down = Direction::Down;

println!("{:?}", discriminant(&up)); // Output: Discriminant(0)
println!("{:?}", discriminant(&down)); // Output: Discriminant(1)
```

The `derive` attribute generates the following implementation:

```rust
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::rust_2021::*;
#[macro_use]
extern crate std;
enum Direction { Up, Down, Left, Right, }

... // Implementation of PartialEq

#[automatically_derived]
impl ::core::cmp::PartialOrd for Direction {
    #[inline]
    fn partial_cmp(&self, other: &Direction)
        -> ::core::option::Option<::core::cmp::Ordering> {
        let __self_discr = ::core::intrinsics::discriminant_value(self);
        let __arg1_discr = ::core::intrinsics::discriminant_value(other);
        ::core::cmp::PartialOrd::partial_cmp(&__self_discr, &__arg1_discr)
    }
}
```
You can manually set the discriminant values for the variants to change the ordering:

```rust
#[derive(PartialEq, PartialOrd)]
enum Direction {
    Up = 4,
    Down = 3,
    Left = 2,
    Right = 1,
}

assert!(Direction::Up > Direction::Down);
```

It is also possible to implement the `PartialOrd` trait manually:

```rust
struct Point {
    x: i32,
    y: i32,
}

impl PartialEq for Point {
    fn eq(&self, other: &Self) -> bool {
        self.x == other.x && self.y == other.y
    }
}

impl PartialOrd for Point {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        match self.x.partial_cmp(&other.x) {
            Some(Ordering::Equal) => self.y.partial_cmp(&other.y),
            other => other,
        }
    }
}

assert!(Point { x: 1, y: 2 } < Point { x: 2, y: 1 });
```

But be careful when implementing the `PartialOrd` trait manually. If you derive the `PartialEq` trait for a type, you should also derive the `PartialOrd` trait to ensure consistency between equality and ordering comparisons.

```rust
#[derive(PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

impl PartialOrd for Point {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.x.cmp(&other.x))
    }
}

let p1 = Point { x: 1, y: 2 };
let p2 = Point { x: 1, y: 1 };

// `PartialEq` and `PartialOrd` are inconsistent
assert!(p1.partial_cmp(&p2) == Some(Ordering::Equal));
assert!(p1 == p2); // This assertion will fail
```

When you choose to implement the `PartialOrd` trait manually, it is advisable to implement all related traits (`PartialEq`, `Eq`, `PartialOrd`, and `Ord`) consistently and manually to ensure logical coherence.

## Ord

The `Ord` trait builds upon the `PartialOrd` and `Eq` traits, ensuring that a type adheres to a [total ordering]. It is defined as follows:

```rust
pub trait Ord: Eq + PartialOrd {
    // Required method
    fn cmp(&self, other: &Self) -> Ordering;

    // Provided methods
    fn max(self, other: Self) -> Self
       where Self: Sized { ... }
    fn min(self, other: Self) -> Self
       where Self: Sized { ... }
    fn clamp(self, min: Self, max: Self) -> Self
       where Self: Sized { ... }
}
```

The `Ord` trait enforces reflexivity through the `Eq` trait and provides the `cmp` method for comparing two values, returning an `Ordering` enum. This ensures that every pair of values can be compared, establishing a total order.

The implementation of the `Ord` trait must be consistent with the `PartialOrd` trait. `Ord` requires that the type also be `PartialOrd`, `PartialEq`, and `Eq`.

You can choose to derive it, or implement it manually. If you derive it, you should derive all four traits. If you implement it manually, you should manually implement all four traits, based on the implementation of `Ord`.

```rust
struct Point {
    x: i32,
    y: i32,
}

impl Ord for Point {
    fn cmp(&self, other: &Self) -> Ordering {
        match self.x.cmp(&other.x) {
            Ordering::Equal => self.y.cmp(&other.y),
            other => other,
        }
    }
}

impl PartialOrd for Point {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl PartialEq for Point {
    fn eq(&self, other: &Self) -> bool {
        self.x == other.x && self.y == other.y
    }
}

impl Eq for Point {}
```

When implementing `PartialOrd`, you can directly call the `cmp` method within the `partial_cmp` implementation. This approach ensures consistency between the `PartialOrd` and `Ord` implementations.

## Reverse

In certain situations, you might need to reverse the ordering of a type. For example, when using the `BinaryHeap` data structure, you may want to retrieve the smallest element instead of the largest. Rust provides the `Reverse` wrapper type to achieve this.

```rust
use std::cmp::Reverse;
use std::collections::BinaryHeap;

fn main() {
    let mut heap = BinaryHeap::new();

    heap.push(Reverse(1));
    heap.push(Reverse(3));
    heap.push(Reverse(2));

    assert_eq!(heap.pop(), Some(Reverse(1)));
    assert_eq!(heap.pop(), Some(Reverse(2)));
    assert_eq!(heap.pop(), Some(Reverse(3)));
}
```

When you want to access the value inside the `Reverse` wrapper, you can use .0 to access the inner value.

```rust
let rev = Reverse(5);
assert_eq!(rev.0, 5);
```

The `Reverse` type is a simple wrapper around a value that implements the `Ord`, `PartialOrd`, `Eq`, and `PartialEq` traits. It reverses the ordering of the inner value, allowing you to change the sorting behavior of a type.

```rust
pub struct Reverse<T>(pub T);
```

Since this is all about ordering, there is no difference for `PartialEq` and `Eq`. However, for `PartialOrd` and `Ord`, the traits are implemented as follows:

```rust
impl<T: PartialOrd> PartialOrd for Reverse<T> {
    #[inline]
    fn partial_cmp(&self, other: &Reverse<T>) -> Option<Ordering> {
        other.0.partial_cmp(&self.0)
    }

    #[inline]
    fn lt(&self, other: &Self) -> bool {
        other.0 < self.0
    }
    #[inline]
    fn le(&self, other: &Self) -> bool {
        other.0 <= self.0
    }
    #[inline]
    fn gt(&self, other: &Self) -> bool {
        other.0 > self.0
    }
    #[inline]
    fn ge(&self, other: &Self) -> bool {
        other.0 >= self.0
    }
}

impl<T: Ord> Ord for Reverse<T> {
    #[inline]
    fn cmp(&self, other: &Reverse<T>) -> Ordering {
        other.0.cmp(&self.0)
    }
}
```

It simply reverses the order of the comparison.  That's all there is to it!

## Conclusion

This was a long post, but I hope it helped you understand the `PartialEq`, `Eq`, `PartialOrd`, `Ord`, and `Reverse` traits in Rust. Remember to use the `derive` attribute whenever possible to avoid inconsistencies.


[IEEE 754]: https://en.wikipedia.org/wiki/IEEE_754
[lexicographical]: https://en.wikipedia.org/wiki/Lexicographic_order
[total ordering]: https://en.wikipedia.org/wiki/Total_order
