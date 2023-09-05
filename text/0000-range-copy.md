- Feature Name: enhance_range_copy
- Start Date: 2023-09-01
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

RFC proposes adding new Range types that are `Copy`able , while changing the range syntax to use those types for future Rust editions. 
Implementing `IntoIterator` rather than `Iterator`, will avoid some drawbacks of the existing Range type - namely easy of use and easier embedding in types.

# Motivation
[motivation]: #motivation

Because all `Range` types in `core::ops` implement Iterator, they can't implement `Copy`, because it could lead to `Iterator` being  and causing issues.

Right now it's a small stumbling block to learning that ranges can't be reused, e.g.:

```rust
let x = 0..10;
for i in x {} // fine
for i in x {} // error x was moved and Range doesn't implement Copy
```

Additionally, types like `Result` and `Option` that pass them around are no longer `Copy`-able, so embedding becomes harder.

In addition with `Range`
```rust
fn eats_a_range(nom: std::ops::Range<usize>) {
    // Say goodbye!
}

fn main() {
    let super_cool_range = 9..27;
    let even_cooler_vec = vec!["Hello", "world!"];
    
    // In the real world, you probably actually care about what's happening inside of `map`.
    // For the sake of example, though, we're gonna pretend the value we get given isn't important.
    
    // `move` causes `super_cool_range` to be moved into the closure, but...
    let foobles = even_cooler_vec.into_iter().map(move |_| eats_a_range(super_cool_range));
    
    // (... do something with `foobles` here)
}
```
[[source]](https://kaylynn.gay/blog/post/rust_ranges_and_suffering)

Prevents duplication of effort:
Currently, a bunch of crates reinvents their own `Range` type to paper over this nitpick, for example:
- [Span in regex automata](https://docs.rs/regex-automata/latest/regex_automata/struct.Span.html)
- [Compiler team had to use SpanData to replace RangeInclusive](https://github.com/rust-lang/compiler-team/issues/534)
- [Clap's num_args](https://docs.rs/clap/latest/clap/struct.Arg.html#method.num_args)
- [toml_edit TomlError uses it for e](https://docs.rs/toml_edit/latest/toml_edit/struct.TomlError.html)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC deals with minor nit in language. It doesn't have a major impact or introduce new terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation


- Create a new type for `Copy`-able `Range` called e.g. `RangeValue`
- Create an alias `type RangeIter = Range` and make existing uses of Range `deprecated`.
  - All usage of `(a..b).map_fn` would need to be change into `(a..b).into_iter().map_fn` in interim.
- In a new Rust edition 202X change desugaring `a .. b` to build `RangeValue`
- Add blanket impl that says that if a type implements `Index<RangeIter>` (alias for 2021 `Range`) it should implement `Index<RangeValue>`. It should be defined
(idea by dlight from [irlo](https://internals.rust-lang.org/t/enhancing-rust-range/19197/12) )

# Drawbacks
[drawbacks]: #drawbacks

There could be some confusion between using `Range` and `RangeIter`; hopefully, `deprecated` notice will remove some of the type confusion. That said this new type may prove source of minor frustration for existing users.

Having double the pointer length isn't super cache-friendly, so it might have some performance problems.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Possible designs include some kind of independent crate, but without adding it to Rust `core`/`std` the range literal old behavior will remain the same.
- It fixes a minor nit in Rust Language but isn't critical, so leaving it unchanged is an alternative.


# Prior art
[prior-art]: #prior-art

Possible prior art might include stuff like:
- [Span in regex automata](https://docs.rs/regex-automata/latest/regex_automata/struct.Span.html)
- [Compiler team had to use SpanData to replace RangeInclusive](https://github.com/rust-lang/compiler-team/issues/534)
- [Clap's num_args](https://docs.rs/clap/latest/clap/struct.Arg.html#method.num_args)
- [toml_edit TomlError uses it for e](https://docs.rs/toml_edit/latest/toml_edit/struct.TomlError.html)

But they are all papering over the Range not being `Copy` and can't fix the standard library.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

N/A

# Future possibilities

The issue is pretty self-contained, and shouldn't affect anything other than changes.
