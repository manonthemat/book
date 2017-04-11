## Advanced Traits

We covered traits in Chapter 10, but like lifetimes, we didn't get to all the
details. Now that we know more Rust, we can get into the nitty-gritty.

### Associated Types

*Associated types* are a way of associating a type placeholder with a trait
such that the trait method definitions can use these placeholder types in their
signatures. The implementer of a trait will specify the concrete type to be
used in this type's place for the particular implementation.

We've described most of the things in this chapter as being very rare.
Associated types are somewhere in the middle; they're more rare than the rest
of the book, but more common than many of the things in this chapter.

An example of a trait with an associated type is the `Iterator` trait provided
by the standard library. It has an associated type named `Item` that stands in
for the type of the values that we're iterating over. We mentioned in Chapter
13 that the definition of the `Iterator` trait is as shown in Listing 19-20:

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

<span class="caption">Listing 19-20: The definition of the `Iterator` trait
that has an associated type `Item`</span>

This says that the `Iterator` trait has an associated type named `Item`. `Item`
is a placeholder type, and the return value of the `next` method will return
values of type `Option<Self::Item>`. Implementers of this trait will specify
the concrete type for `Item`, and the `next` method will return an `Option`
containing a value of whatever type the implementer has specified.

#### Associated Types Versus Generics

When we implemented the `Iterator` trait on the `Counter` struct in Listing
13-6, we specified that the `Item` type was `u32`:

```rust,ignore
impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
```

This feels similar to generics. So why isn't the `Iterator` trait defined as
shown in Listing 19-21?

```rust
pub trait Iterator<T> {
    fn next(&mut self) -> Option<T>;
}
```

<span class="caption">Listing 19-21: A hypothetical definition of the
`Iterator` trait using generics</span>

The difference is that with the definition in Listing 19-21, we could also
implement `Iterator<String> for Counter`, or any other type as well, so that
we'd have multiple implementations of `Iterator` for `Counter`. In other words,
when a trait has a generic parameter, we can implement that trait for a type
multiple times, changing the generic type parameters' concrete types each time.
Then when we use the `next` method on `Counter`, we'd have to provide type
annotations to indicate which implementation of `Iterator` we wanted to use.

With associated types, we can't implement a trait on a type multiple times.
Using the actual definition of `Iterator` from Listing 19-20, we can only
choose once what the type of `Item` will be, since there can only be one `impl
Iterator for Counter`. We don't have to specify that we want an iterator of
`u32` values everywhere that we call `next` on `Counter`.

The benefit of not having to specify generic type parameters when a trait uses
associated types shows up in another way as well. Consider the two traits
defined in Listing 19-22. Both are defining a trait having to do with a graph
structure that contains nodes of some type and edges of some type. `GGraph` is
defined using generics, and `AGraph` is defined using associated types:

```rust
trait GGraph<Node, Edge> {
    // methods would go here
}

trait AGraph {
    type Node;
    type Edge;

    // methods would go here
}
```

<span class="caption">Listing 19-22: Two graph trait definitions, `GGraph`
using generics and `AGraph` using associated types for `Node` and `Edge`</span>

Let's say we wanted to implement a function that computes the distance between
two nodes in any types that implement the graph trait. With the `GGraph` trait
defined using generics, our `distance` function signature would have to look
like Listing 19-23:

```rust
# trait GGraph<Node, Edge> {}
#
fn distance<N, E, G: GGraph<N, E>>(graph: &G, start: &N, end: &N) -> u32 {
#     0
}
```

<span class="caption">Listing 19-23: The signature of a `distance` function
that uses the trait `GGraph` and has to specify all the generic
parameters</span>

Our function would need to specify the generic type parameters `N`, `E`, and
`G`, where `G` is bound by the trait `GGraph` that has type `N` as its `Node`
type and type `E` as its `Edge` type. Even though `distance` doesn't need to
know the types of the edges, we're forced to declare an `E` parameter, because
we need to to use the `GGraph` trait and that requires specifying the type for
`Edge`.

Contrast with the definition of `distance` in Listing 19-24 that uses the
`AGraph` trait from Listing 19-22 with associated types:

```rust
# trait AGraph {
#     type Node;
#     type Edge;
# }
#
fn distance<G: AGraph>(graph: &G, start: &G::Node, end: &G::Node) -> u32 {
#     0
}
```

<span class="caption">Listing 19-24: The signature of a `distance` function
that uses the trait `AGraph` and the associated type `Node`</span>

This is much cleaner. We only need to have one generic type parameter, `G`,
with the trait bound `AGraph`. Since `distance` doesn't use the `Edge` type at
all, it doesn't need to be specified anywhere. To use the `Node` type
associated with `AGraph`, we can specify `G::Node`.

#### Trait Objects with Associated Types

You may have been wondering why we didn't use a trait object in the `distance`
functions in Listing 19-23 and Listing 19-24. The signature for the `distance`
function using the generic `GGraph` trait does get a bit more concise using a
trait object:

```rust
# trait GGraph<Node, Edge> {}
#
fn distance<N, E>(graph: &GGraph<N, E>, start: &N, end: &N) -> u32 {
#     0
}
```

This might be a more fair comparison to Listing 19-24. Specifying the `Edge`
type is still required, though, which means Listing 19-24 is still preferable
since we don't have to specify something we don't use.

It's not possible to change Listing 19-24 to use a trait object for the graph,
since then there would be no way to refer to the `AGraph` trait's associated
type.

It is possible in general to use trait objects of traits that have associated
types, though; Listing 19-25 shows a function named `traverse` that doesn't
need to use the trait's associated types in other arguments. We do, however,
have to specify the concrete types for the associated types in this case. Here,
we've chosen to accept types that implement the `AGraph` trait with the
concrete type of `usize` as their `Node` type and a tuple of two `usize` values
for their `Edge` type:

```rust
# trait AGraph {
#     type Node;
#     type Edge;
# }
#
fn traverse(graph: &AGraph<Node=usize, Edge=(usize, usize)>) {}
```

While trait objects mean that we don't need to know the concrete type of the
`graph` parameter at compile time, we do need to constrain the use of the
`AGraph` trait in the `traverse` function by the concrete types of the
associated types. If we didn’t provide this constraint, Rust wouldn't be able
to figure out which `impl` to match this trait object to.

TODO: I basically copied the last sentence from the old book but i dont really understand it /Carol

### Operator Overloading and Default Type Parameters

The `<PlaceholderType=ConcreteType>` syntax is used in another way as well: to
specify the default type for a generic type. A great example of a situation
where this is useful is operator overloading.

Rust does not allow you to create your own operators or overload arbitrary
operators, but the operations listed in `std::ops` can be overloaded by
implementing the traits associated with the operator. For example, Listing
19-25 shows how to overload the `+` operator by implementing the `Add` trait on
a `Point` struct so that we can add two `Point` instances together:

<span class="filename">Filename: src/main.rs</span>

```rust
use std::ops::Add;

#[derive(Debug,PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

impl Add for Point {
    type Output = Point;

    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

fn main() {
    assert_eq!(Point { x: 1, y: 0 } + Point { x: 2, y: 3 },
               Point { x: 3, y: 3 });
}
```

<span class="caption">Listing 19-25: Implementing the `Add` trait to overload
the `+` operator for `Point` instances</span>

We've implemented the `add` method to add the `x` values of two `Point`
instances together and the `y` values of two `Point` instances together to
create a new `Point`. The `Add` trait has an `Output` associated type that's
used to determine the type returned from `add`. result of the operation.

Let's look at the `Add` trait in a bit more detail. Here's its definition:

```rust
trait Add<RHS=Self> {
    type Output;

    fn add(self, rhs: RHS) -> Self::Output;
}
```

This should look familiar; it's a trait with one method and an associated type.
The new part is the `RHS=Self` in the angle brackets: this syntax is called
*default type parameters*. `RHS` is a generic type parameter (short for "right
hand side") that's used for the type of the `rhs` parameter in the `add`
method. If we don't specify a concrete type for `RHS` when we implement the
`Add` trait, the type of `RHS` will default to the type of `Self` (the type
that we're implementing `Add` on).

Let's look at another example of implementing the `Add` trait. Imagine we have
two structs holding values in different units, `Millimeters` and `Meters`. We
can implement `Add` for `Millimeters` in different ways as shown in Listing
19-26:

```rust
use std::ops::Add;

struct Millimeters(u32);
struct Meters(u32);

impl Add for Millimeters {
    type Output = Millimeters;

    fn add(self, other: Millimeters) -> Millimeters {
        Millimeters(self.0 + other.0)
    }
}

impl Add<Meters> for Millimeters {
    type Output = Millimeters;

    fn add(self, other: Meters) -> Millimeters {
        Millimeters(self.0 + (other.0 * 1000))
    }
}
```

<span class="caption">Listing 19-26: Implementing the `Add` trait on
`Millimeters` to be able to add `Millimeters` to `Millimeters` and
`Millimeters` to `Meters`</span>

If we're adding `Millimeters` to other `Millimeters`, we don't need to
parameterize the `RHS` type for `Add` since the default `Self` type is what we
want. If we want to implement adding `Millimeters` and `Meters`, then we need
to say `impl Add<Meters>` to set the value of the `RHS` type parameter.

Default type parameters are used in two main ways:

1. To extend a type without breaking existing code.
2. To allow customization in a way most users don't want.

The `Add` trait is an example of the second purpose: most of the time, you're
adding two like types together. Using a default type parameter in the `Add`
trait definition makes it easier to implement the trait since you don't have to
specify the extra parameter most of the time. In other words, we've removed a
little bit of implementation boilerplate.

The first purpose is similar, but in reverse: since existing implementations of
a trait won't have specified a type parameter, if we want to add a type
parameter to an existing trait, giving it a default will let us extend the
functionality of the trait without breaking the existing implementation code.

### Fully Qualified Syntax for Disambiguation

Rust cannot prevent a trait from having a method with the same name that
another trait's method has, nor can it prevent us from implementing both of
these traits on one type. We can also have a method implemented directly on the
type with the same name as well! In order to be able to call each of the
methods with the same name, then, we need to tell Rust which one we want to
use. Consider the code in Listing 19-27 where traits `Foo` and `Bar` both have
method `f` and we implement both traits on struct `Baz`, which also has a
method named `f`:

<span class="filename">Filename: src/main.rs</span>

```rust
trait Foo {
    fn f(&self);
}

trait Bar {
    fn f(&self);
}

struct Baz;

impl Foo for Baz {
    fn f(&self) { println!("Baz’s impl of Foo"); }
}

impl Bar for Baz {
    fn f(&self) { println!("Baz’s impl of Bar"); }
}

impl Baz {
    fn f(&self) { println!("Baz's impl"); }
}

fn main() {
    let b = Baz;
    b.f();
}
```

<span class="caption">Listing 19-27: Implementing two traits that both have a
method with the same name as a method defined on the struct directly</span>

For the implemetation of the `f` method for the `Foo` trait on `Baz`, we're
printing out `Baz's impl of Foo`. For the implementation of the `f` method for
the `Bar` trait on `Baz`, we're printing out `Baz's impl of Bar`. The
implementation of `f` directly on `Baz` prints out `Baz's impl`. What should
happen when we call `b.f()`? In this case, Rust will always use the
implementation on `Baz` directly and will print out `Baz's impl`.

In order to be able to call the `f` method from `Foo` and the `f` method from
`Baz` rather than the implementation of `f` directly on `Baz`, we need to use
the *fully qualified syntax* for calling methods. It works like this: for any
method call like:

```rust,ignore
receiver.method(args);
```

We can fully qualify the method call like this:

```rust,ignore
<Type as Trait>::method(receiver, args);
```

So in order to disambiguate and be able to call all the `f` methods defined in
Listing 19-27, we specify that we want to treat the type `Baz` as each trait
within angle brackets, then use two colons, then call the `f` method and pass
the instance of `Baz` as the first argument. Listing 19-28 shows how to call
`f` from `Foo` and then `f` from `Bar` on `b`:

<span class="filename">Filename: src/main.rs</span>

```rust
# trait Foo {
#     fn f(&self);
# }
# trait Bar {
#     fn f(&self);
# }
# struct Baz;
# impl Foo for Baz {
#     fn f(&self) { println!("Baz’s impl of Foo"); }
# }
# impl Bar for Baz {
#     fn f(&self) { println!("Baz’s impl of Bar"); }
# }
fn main() {
    let b = Baz;
    b.f();
    <Baz as Foo>::f(&b);
    <Baz as Bar>::f(&b);
}
```

<span class="caption">Listing 19-28: Using fully qualified syntax to call the
`f` methods defined as part of the `Foo` and `Bar` traits</span>

This will print:

```text
Baz's impl
Baz’s impl of Foo
Baz’s impl of Bar
```

We only need the `Type as` part if it's ambiguous, and we only need the `<>`
part if we need the `Type as` part. So if we only had the `f` method directly
on `Baz` and the `Foo` trait implemented on `Baz` in scope, we could call the
`f` method in `Foo` by using `Foo::f(&b)` since we wouldn't have to
disambiguate from the `Bar` trait.

### Super traits

Sometimes, you may want a trait to be able to rely on another trait existing.
For example, let's say that you have a `Foo` trait and a `Bar` trait, but you
want `Bar`'s methods to be able to call `Foo`'s methods. Let's try it. (It
won't work just yet...)

```rust,ignore
trait Foo {
    fn foo(&self) {
        println!("Foo");
    }
}

trait Bar {
    fn bar(&self) {
        self.foo();
    }
}
```

We get this error:

```text
error: no method named `foo` found for type `&Self` in the current scope
  --> <anon>:10:14
   |
10 |         self.foo();
   |              ^^^
   |
   = help: items from traits can only be used if the trait is implemented and in scope; the following trait defines an item `foo`, perhaps you need to implement it:
   = help: candidate #1: `main::Foo`
```

In other words, we haven't said that anything that implements `Bar` also
implements `Foo`. We can do that with a `:`, like this:

```rust
trait Foo {
    fn foo(&self) {
        println!("Foo");
    }
}

trait Bar: Foo {
    fn bar(&self) {
        self.foo();
    }
}
```

This works fine.

### Coherence

Finally, traits have a concept called 'coherence'. This governs exactly who is
allowed to implement a trait. In short:

> To implement a type for a trait, you must have defined either the type, the
> trait, or both.

Put another way:

> You cannot implement a trait you didn't define for a type you didn't define.

For example, defining the `Display` trait, which is defined in the standard
library, on a tuple of string slices, which is defined in the standard library,
won't work:

```rust,ignore
use std::fmt;

impl fmt::Display for (&'static str, &'static str) {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.0, self.1)
    }
}
```

gives

```text
error[E0117]: only traits defined in the current crate can be implemented for arbitrary types
 --> <anon>:4:1
  |
4 |   impl fmt::Display for (&'static str, &'static str) {
  |  _^ starting here...
5 | |     fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
6 | |         write!(f, "({}, {})", self.0, self.1)
7 | |     }
8 | | }
  | |_^ ...ending here: impl doesn't use types inside crate
  |
  = note: the impl does not reference any types defined in this crate
```

Why do we have this rule? Allowing this would lead to ambiguity, confusion, and
broken code.  Imagine that we have a crate `foo` that has a type `A` and a
trait `B`. If we could implement `B` for `A` in our code, it would work, but
what if someone else _also_ implemented `B` for `A` in their code? Furthermore,
what if a new release of `foo` comes out and implements `B` for `A` themselves?
These problems are not insurmountable, of course; we could determine some kind
of complex precedent rules to determine which `impl` 'wins' and works.

### The newtype pattern

There is a way to get around this, though. We call it the 'newtype pattern'.
You create a new type that's a thin wrapper around the type you want to
implement the trait for, and then implement the trait for the wrapper. This
*will* work:

```rust
use std::fmt;

struct Wrapper((&'static str, &'static str));

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", (self.0).0, (self.0).1)
    }
}
```

The downside is that since `Wrapper` is a new type, it has no methods; we'll
have to implement them all. If you want it to have every single method that the
inner type has, implementing `Deref` can help you there. Otherwise, you'll have
to implement the methods yourself.
