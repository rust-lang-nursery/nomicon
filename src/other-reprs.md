# Alternative representations

Rust allows you to specify alternative data layout strategies from the default.
There's also the [unsafe code guidelines] (note that it's **NOT** normative).




# repr(C)

This is the most important `repr`. It has fairly simple intent: do what C does.
The order, size, and alignment of fields is exactly what you would expect from C
or C++. Any type you expect to pass through an FFI boundary should have
`repr(C)`, as C is the lingua-franca of the programming world. This is also
necessary to soundly do more elaborate tricks with data layout such as
reinterpreting values as a different type.

We strongly recommend using [rust-bindgen][] and/or [cbindgen][] to manage your FFI
boundaries for you. The Rust team works closely with those projects to ensure
that they work robustly and are compatible with current and future guarantees
about type layouts and reprs.

The interaction of `repr(C)` with Rust's more exotic data layout features must be
kept in mind. Due to its dual purpose as "for FFI" and "for layout control",
`repr(C)` can be applied to types that will be nonsensical or problematic if
passed through the FFI boundary.

* ZSTs are still zero-sized, even though this is not a standard behavior in
C, and is explicitly contrary to the behavior of an empty type in C++, which
says they should still consume a byte of space.

* DST pointers (wide pointers) and tuples are not a concept
  in C, and as such are never FFI-safe.

* Enums with fields also aren't a concept in C or C++, but a valid bridging
  of the types [is defined][really-tagged].

* If `T` is an [FFI-safe non-nullable pointer
  type](ffi.html#the-nullable-pointer-optimization),
  `Option<T>` is guaranteed to have the same layout and ABI as `T` and is
  therefore also FFI-safe. As of this writing, this covers `&`, `&mut`,
  and function pointers, all of which can never be null.

* Tuple structs are like structs with regards to `repr(C)`, as the only
  difference from a struct is that the fields aren’t named.

* `repr(C)` is equivalent to one of `repr(u*)` (see the next section) for
fieldless enums. The chosen size is the default enum size for the target platform's C
application binary interface (ABI). Note that enum representation in C is implementation
defined, so this is really a "best guess". In particular, this may be incorrect
when the C code of interest is compiled with certain flags.

* Fieldless enums with `repr(C)` or `repr(u*)` still may not be set to an
integer value without a corresponding variant, even though this is
permitted behavior in C or C++. It is undefined behavior to (unsafely)
construct an instance of an enum that does not match one of its
variants. (This allows exhaustive matches to continue to be written and
compiled as normal.)



# repr(transparent)

This can only be used on structs with a single non-zero-sized field (there may
be additional zero-sized fields). The effect is that the layout and ABI of the
whole struct is guaranteed to be the same as that one field.

The goal is to make it possible to transmute between the single field and the
struct. An example of that is [`UnsafeCell`], which can be transmuted into
the type it wraps.

Also, passing the struct through FFI where the inner field type is expected on
the other side is guaranteed to work. In particular, this is necessary for `struct
Foo(f32)` to always have the same ABI as `f32`.

More details are in the [RFC][rfc-transparent].

As an extended example, it is instructive to think about transparent types
in terms of wrappers.

In Rust we can define an integer type three ways

```
struct Wrapper1(u32);
struct Wrapper2 {
    value: u32
}

fn main() {
    let x:u32 = 4;
    let y:Wrapper1 = Wrapper1(4);
    let z:Wrapper2 = Wrapper2{ value: 4 };
}
```

This defines an integer, a tuple with a single member, and a struct
with a single field.  They are three different types, but they are all
represented by four bytes in memory representing the value 4.  They
all represent the value 4, but they are not 4.  In particular, we can
write `x+1` but we cannot write `y+1` or `z+1`.

There are two reasons you might want to use a wrapper:

* New types: You can use wrappers `Vec1Index` and `Vec2Index` around
  integers and create types `Vec1` and `Vec2` that can be indexed by
  `Vec1Index(4)` and `Vec2Index(4)` respectively but not vice versa.
  The wrappers `Vec1Index(4)` and `Vec2Index(4)` around the integer `4`
  define new, distinct types even though the underlying representation
  is just four bytes representing the integer `4`.

* New assertions: You can use a wrapper like `Positive` around an
  integer to indicate to the user that the integer is guaranteed to be
  positive.  Just ensure that `Positive::new(4)` succeeds and
  `Postive::new(0)` panics, so the value wrapped by `Positive` is
  guaranteed to be positive.

Again, the integer wrappers above are all respresented by four bytes.
The method of accessing the four bytes, however, differ.  We can write
`x+1` and not `z+1`, but we can write `z.value+1`.

Rust lets us annotate wrappers with the "transparent" attribute. (Rust
allows the wrappers to have additional zero-length fields like phantom
data, but let's ignore that for now).
See the
[Transparent RFC](https://rust-lang.github.io/rfcs/1758-repr-transparent.html) for
an excellent discussion.

The purpose of the transparent attribute on an integer wrapper is to
allow us to silently transmute between the integer and the integer
wrapper.  It is a mistake to read the documentation as saying that the
wrapper just disappears when it is declared transparent.  Even if we
make `Wrapper1` and `Wrapper2` above transparent, that doesn't mean that
in Rust code we can treat them like ordinary integers.  We still have to
use tuple or field selection to access the underlying integer, even
though they have the same underlying representation.

The place where transparent becomes interesting is in a foreign
function interface (FFI).

There are two reasons you might want to declare a wrapper to be
transparent in the conext of an FFI:

* You might want to give an assurance or a warning about the return
  value from a function.  Imagine C function that returns a `u32` as a
  return value `rv`.  You might want to wrap `rv` in a wrapper
  `Postive(rv)` or `Warning(rv)` because you know that `rv` is positive
  or you know there might be something dangerous about `rv` that a user
  should be forced to check.  If we declare the wrapper to be
  transparent, then the FFI is allowed to silently transmute the `u32`
  return value `rv` into `Positive(rv)` or `Warning(rv)`.

* You might be compiling to an architecture that does not allow
  functions to return structs, so a function fundamentally cannot return
  a tuple or struct wrapper.  The original motivation for transparent
  appears to be the ARM architecture where a struct value is returned by
  passing the function a pointer to the destination struct, and the
  function fills in the struct via the pointer.  This is true even for a
  wrapper struct like ours that contains only a single integer field,
  and even if the function is fully capable of returning the integer
  value intended to be contained in that field.

It is this second case and discussion in the literature of "handling"
the integer wrapper as if it were just an integer that can lead to
confusion.

What transparent really means is that the underlying representation of
the wrapper is the same as the underlying representation of the wrapped
value.  But this is often naturally true.  An u32 and a u32 struct
wrapper are both going to consume four bytes.  A struct containing a u32
struct wrapper as a substruct is going to allocate four bytes for the
u32 struct wrapper just as if would allocate four bytes for the u32 if
the u32 struct wrapper were replaced by the u32 itself.
But the struct containing the u32 wrapper and the
struct containing the u32 are different structs with different types,
even though the underlying representations are the same.
The method of accessing the u32 value is different: access the wrapped u32
with struct.member.submember versus access the u32 with struct.member.

# repr(u*), repr(i*)

These specify the size to make a fieldless enum. If the discriminant overflows
the integer it has to fit in, it will produce a compile-time error. You can
manually ask Rust to allow this by setting the overflowing element to explicitly
be 0. However Rust will not allow you to create an enum where two variants have
the same discriminant.

The term "fieldless enum" only means that the enum doesn't have data in any
of its variants. A fieldless enum without a `repr(u*)` or `repr(C)` is
still a Rust native type, and does not have a stable ABI representation.
Adding a `repr` causes it to be treated exactly like the specified
integer size for ABI purposes.

If the enum has fields, the effect is similar to the effect of `repr(C)`
in that there is a defined layout of the type. This makes it possible to
pass the enum to C code, or access the type's raw representation and directly
manipulate its tag and fields. See [the RFC][really-tagged] for details.

Adding an explicit `repr` to an enum suppresses the null-pointer
optimization.

These reprs have no effect on a struct.




# repr(packed)

`repr(packed)` forces Rust to strip any padding, and only align the type to a
byte. This may improve the memory footprint, but will likely have other negative
side-effects.

In particular, most architectures *strongly* prefer values to be aligned. This
may mean the unaligned loads are penalized (x86), or even fault (some ARM
chips). For simple cases like directly loading or storing a packed field, the
compiler might be able to paper over alignment issues with shifts and masks.
However if you take a reference to a packed field, it's unlikely that the
compiler will be able to emit code to avoid an unaligned load.

**[As of Rust 2018, this still can cause undefined behavior.][ub loads]**

`repr(packed)` is not to be used lightly. Unless you have extreme requirements,
this should not be used.

This repr is a modifier on `repr(C)` and `repr(Rust)`.




# repr(align(n))

`repr(align(n))` (where `n` is a power of two) forces the type to have an
alignment of *at least* n.

This enables several tricks, like making sure neighboring elements of an array
never share the same cache line with each other (which may speed up certain
kinds of concurrent code).

This is a modifier on `repr(C)` and `repr(Rust)`. It is incompatible with
`repr(packed)`.





[unsafe code guidelines]: https://rust-lang.github.io/unsafe-code-guidelines/layout.html
[drop flags]: drop-flags.html
[ub loads]: https://github.com/rust-lang/rust/issues/27060
[`UnsafeCell`]: ../std/cell/struct.UnsafeCell.html
[rfc-transparent]: https://github.com/rust-lang/rfcs/blob/master/text/1758-repr-transparent.md
[really-tagged]: https://github.com/rust-lang/rfcs/blob/master/text/2195-really-tagged-unions.md
[rust-bindgen]: https://rust-lang.github.io/rust-bindgen/
[cbindgen]: https://github.com/eqrion/cbindgen
