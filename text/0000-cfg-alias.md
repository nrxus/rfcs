- Feature Name: `cfg-alias`
- Start Date: 2018-04-13
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allows adding an alias for an existing `#[cfg(...)]` attribute or combination of attributes.

The proposed syntax is:
```rust
#[cfg(my_alias)] = #[cfg(all(complicated, stuff, not(easy, to, remember)))]
```

# Motivation
[motivation]: #motivation

It is not uncommon for `#[cfg(...)]` attributes to be composed of other properties, such as:

`#[cfg(any(target_arch = "x86", target_arch = "x86_64"))]`

The above conditional compilation attribute will prevent this code to attempt to compile on
target architectures that are not `x86` or `x86_64`.

This way of combining conditional compilation attributes is used often,
and there is currently no way to encapsulate it into a single conditional compilation attribute
that can be re-used.

This RFC proposes introducing aliases to `#[cfg(...)]` attributes, to help the user encapsulate
existing conditional compilation attributes.

With this RFC, you would be able to write the following:

```rust
#[cfg(any_x86)] = #[cfg(any(target_arch = "x86", target_arch = "x86_64"))]

#[cfg(any_x86)]
fn function_compiles_only_on_x86_target_archs() {
  //...
}

#[cfg(not(any_x86))]
fn function_compiles_only_on_not_x86_target_archs() {
  //...
}
```

The new `#[cfg(any_x86)]` attribute could be used anywhere that
`#[cfg(any(target_arch = "x86", target_arch = "x86_64"))]` and have the same effect.

In the above example, there are only two conditional compilation attributes wrapped in an `any`.
Once could easily use this to simplify many conditional compilation attributes, such as:

```rust
#[cfg(integration_test)] = #[cfg(all(test, feature = integration))]
#[cfg(contract_test)] = #[cfg(all(test, feature = contract))]
#[cfg(unit_test)] = #[cfg(not(any(integration_test, contract_test)))]

#[cfg(unit_test)]
mod test() {
  //... run your fast tests
}

#[cfg(integration_test)]
mod test() {
  //...do some slower tests with the real services mocked
}

#[cfg(contract_test)]
mod test() {
  //...run some contract tests to verify responses of real network calls
}

```

The above aliases could be used throughout a codebase,
allowing the developer to easily control which tests can be run through a feature flag.

Currently the above scenario would require duplicating on the meaning of `unit_test`
for every block of unit tests the developer would write and not allow the developer
to name the concept of `#[cfg(test, not(any(integration, contract)))]`

The proposed syntax for cfg aliases is meant to reflect the current way in which
Rust allows for aliasing of other types/traits.

```rust
// alias types:
struct MyComplicatedStruct;
type MyStruct = MyComplicatedStruct;

// alias enums per RFC 2338:
enum MyComplicatedEnum {};
type MyEnum = MyComplicatedEnum;

// alias traits per RFC 1733:
trait MyComplicatedTrait;
trait MyTrait = MyComplicatedTrait;

// alias conditional compilation attributes per this RFC:
#[cfg(my_attr)] = #[cfg(my_complicated_attribute)]
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

`#[cfg(...)]` is used to control the compilation (and therefore execution) of certain code routes.
Oftentime, a single attribute is not enough and Rust provides the ability of combining attributes using:

`#[cfg(any(x, y, ...))]`
`#[cfg(all(x, y, ...))]`
`#[cfg(not(x))]`

Repeating the combinations of the attributes can be frustrating and a cause of errors, thus
Rust provides the ability to alias `#[cfg(...)]` attributes, allowing you to name repeated concepts.

Rust allows you to do this by introducing the the aliases at the top of your crate.

If you are writing a binary, you can it to your `main.rs`.
If you are writing a library, you can add it to your `lib.rs`

```rust
//main.rs or lib.rs
#[cfg(my_alias)] = #[cfg(all(some, long, complicated, not(easy, to, remember)))]
```

Aliases introduced at the top of your create will be usable anywhere in your crate.

```rust
//START OF LIB.RS

mod inner;

#[cfg(my_alias)] = #[cfg(all(some, long, complicated, not(easy, to, remember)))]

#[cfg(my_alias)]
fn some_fn() {
  //...
}

//END OF LIB.RS

//START OF INNER.RS

#[cfg(my_alias)] //this will work because the alias was introduced in lib.rs
fn conditional_fn() {
  //...
}
```

`#[cfg(...)]` aliases do NOT get exposed between crates.
If in your binary, you are using the common pattern of having a `lib` module that gets exported
to your `main.rs`, then any cfg alias declared in your `main.rs`
will NOT be exposed to the files in your `lib` module.
In a similar vein, any cfg alias declared in your `lib` module will not be exposed to your `main.rs`
binary.

Declaring a `#[cfg(...)]` alias in other module will be an error at compile time.

```rust
//lib.rs

#[cfg(my_alias)] = #[cfg(any(foo, bar))] // this will compile

mod inner {
  #[cfg(my_other_alias)] = #[cfg(any(foo, bar))] // this will NOT compile
}
```

A repeated alias will be an error at compile time

```rust
#[cfg(foobar)] = #[cfg(any(foo, bar))]
#[cfg(foobar)] = #[cfg(any(x))] // error. foobar already exists

```

In a similar vein,
an alias that shares the same name as an existing cfg will be an error at compile time

//compiled using `--cfg=foobar` flag in rustc

```rust
$[cfg(foobar)] = #[cfg(any(foo, bar))] // error. foobar already exists
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Any `#[cfg(...)]` alias should behave exactly as its expanded non-aliased form
and be accepted in the same spots as any other `#[cfg(...)]` attribute.

Aliases will only  be allowed at the top of a crate. This means either `main.rs` for a binary,
or `lib.rs` for a library.

Aliases that are defined at the root of the crate can be used anywhere WITHIN the crate.

Aliases are not exportable between crates, and a binary that is split into one/many `lib`s, will NOT
be able to use or share the `#[cfg(...)]` aliases from it's libraries.

Aliases will fail at compile-time if the same attribute already exists, either due to an existing alias,
or an existing configuration passed through the `cfg=...` flag in rustc.

# Drawbacks
[drawbacks]: #drawbacks

This introduces another way in which conditional compilation attributes may be introduced,
thus potentially making it harder to find why a path of code was compiled or not.

With this design, there is no immediate way to differentiate an aliased cfg versus one introduced
through a compiler flag/feature.

That being said, it is the belief of the author that by blocking the export of cfg alias, it becomes
the responsability of the developer to be aware of what cfg have been aliased within their crate.

It also the belief of the author, that as an user of this feature I do not want to care where the
cfg attribute came from (i.e., alias, intrinsic, or compiler flag).
If I truly care then a simple search for the feature within my root module will immediately return
whether the cfg is an alias, and an alias of what.

# Rationale and alternatives
[alternatives]: #alternatives

## Syntax Alternatives

An [existing issue](https://github.com/rust-lang/rfcs/issues/831) proposed the following syntax:

```rust
#![cfg_shortcut(foo, any(super, long, clause))]
```

for introducing "shortcuts" of existing cfg attributes.

This alternative syntax provides the same functionality as the syntax proposed in this RFC,
but it introduces the concept a "shortcut" which would be a new term in the language.

I believe referring to this as an alias rather than a shortcut matches the current terms
used in the rust language better. Also the syntax proposed in this RFC more closely matches
how we alias types or traits in rust.

Another alternative would be to identify aliased cfg's through a special prefix.

For example, if we forced aliases to be prefixed with `$` we would get the following:

```rust
#[cfg($foobar)] = #[cfg(any(foo,bar))]

#[cfg($foobar)]
fn conditional_function() {
  //...
}
```

The above syntax achieves the same as the proposed syntax, with the gained advantage that now it is
easier to identify whether a cfg attribute comes from an alias or an existing cfg attribute.

The disadvantage is that now we are exposing, what the author believes to be, an implementation detail.
The fact that the cfg attribute comes from an alias should NOT be important at the moment of use.

This syntax also means that no cfgs will be allowed through a compiler flag that start with `$`.

The author does not know (nor could he find) if we currently block any prefixes on cfg attribute names.

## Visibility alternatives

The proposed RFC blocks the introduction of cfg aliases anywhere but in the root of the crate
(`main.rs` for binaries or `lib.rs` for libraries). An alternative would be to allow it anywhere,
and expose those aliases within any child of that module. This will allow for modularizing the
declaration of cfg aliases only at the top of where it is needed.

The reasoning behind not going for this approach is that it further escalates the problem of not
knowing where the cfg alias might have come from,
since now it can be in any parent module instea of only at the root.
Implementing the RFC as-is would allow to add this further enhancement later without any breaking
changes. The author recommends revisiting this point after the current implementation is stable,
and we get enough feedback on whether declaring cfg aliases would be useful in non-root modules.

Another alternative would be to allow the cfg alias in any module,
but do NOT expose it to any other module, child or parent.
The author believes this would severely limit the use case of aliasing cfg attributes since the logic
would have to be copied across modules within the same crate.

# Prior art
[prior-art]: #prior-art

There is an [existing issue](https://github.com/rust-lang/rfcs/issues/831) filed to the RFC repository
that lies down a proposal very similar to this one but with different syntax.

Currently in Rust, we allow the aliasing of structs and enums, with an approved RFC for aliasing
traits. This will extend the concept of aliasing to `#[cfg(...)]` attributes.

# Unresolved questions
[unresolved]: #unresolved-questions

- Does rust currently have limitations on the names of cfg attributes?
