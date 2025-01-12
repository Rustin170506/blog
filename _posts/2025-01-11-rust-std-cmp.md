---
title: 'Rust equality and ordering'
layout: post

categories: post
tags:
- Rust
- Ed
- PartialEq
- Ord
- PartialOrd
---

# std::cmp

The `std::cmp` module offers several traits for comparing values. It allows us to check for equality and determine the ordering of values. Unlike some other languages where the compiler automatically provides these implementations, Rust requires us to explicitly implement or derive these traits.

Implementing these traits can sometimes be challenging. In this post, we will explore the `PartialEq`, `Eq`, `PartialOrd`, `Ord`, and `Reverse` traits in detail.

## PartialEq

Many might assume that `PartialEq` is termed "partial" because it only compares certain parts of a type. However, the "partial" in `PartialEq` actually indicates that the equality comparison does not need to be reflexive. In other words, `PartialEq` does not require a type to always be equal to itself.

It trait is defined as follows:

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

The `PartialEq` trait mandates the implementation of the `eq` method, which checks if two values are equal. By default, the `ne` method is also provided, which checks if two values are not equal. Essentially, `eq` corresponds to the `==` operator, while `ne` corresponds to the `!=` operator.

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

So Rust provides the `Eq` trait, which extends the `PartialEq` trait by enforcing reflexivity.

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

Of course, you can also implement the `PartialEq` trait manually:

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

[IEEE 754]: https://en.wikipedia.org/wiki/IEEE_754
