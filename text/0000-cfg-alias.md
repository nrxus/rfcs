- Feature Name: `cfg-alias`
- Start Date: 2018-04-13
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

ANDRES TODO
- choose either "the author" or "I" type language and stick with it
- remove use of "we", use "Ruse" or "In Rust"

# Summary
[summary]: #summary

Allows adding an alias for an existing `#[cfg(...)]` attribute or combination of attributes.

The proposed syntax is:
```rust
#[cfg(my_alias)] = #[cfg(all(complicated, stuff, not(easy, to, remember)))]
```

# Motivation
[motivation]: #motivation

It is not uncommon for `#[cfg(...)]` conditional compilation attributes, referred to as cfg attributes going forward, to be composed of other properties. For example, the following cfg attribute prevents code from being compiled on target architectures that are not `x86` or `x86_64`:

`#[cfg(any(target_arch = "x86", target_arch = "x86_64"))]`

Because there is currently no concise way to refer to this cfg attribute, it is repeated verbatim 48<sup>*</sup> times in multiple files across the rustc code.

This RFC proposes introducing aliases for cfg attributes, allowing developers to encapsulate a long cfg attribute into single reusable name.

If the above changes are accepted, developers could write the following:

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

The new `#[cfg(any_x86)]` attribute could be used anywhere that `#[cfg(any(target_arch = "x86", target_arch = "x86_64"))]` is currently used and have the same effect.

Another concrete example comes from the popular crate serde (v1.0.37), in which the author found 77<sup>*</sup> uses of `#[cfg(any(feature = "std", feature = "alloc"))]`. This RFC would allow serde developers to write an alias that gives meaning to the combination of these features and use that single name throughout their codebase.

The two examples contain only two cfg attributes wrapped in an `any`. One could also use cfg aliases to simplify other types of cfg attributes. For example:

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

These aliases could be used throughout a codebase, allowing the developer to easily control which tests can be run using a feature flag.

Currently, the above scenario would require duplicating on the meaning of `unit_test`
for every block of unit tests the developer would write and not allow the developer
to name the concept of `#[cfg(test, not(any(feature = integration, feature = contract)))]` (I don't understand. give example of what current state of things looks like or reword)

The proposed syntax for cfg aliases is meant to reflect the current way in which
Rust allows for aliasing of other types or traits.

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

<sup>\* Data found by running this command on the different repositories (what are these different repositories? "all the repositories"? then why did you say rustc above specifically?) of the rust 2018-04-01 nightly build:
`ag "^\s*#\[cfg\(((.|\n)*?)\]" -o | sed 's/.*://' | sed 's/^[ ]*//' | sed '/^$/d' | sed ':a;N;$!ba;s/,\n/,\t/g' | tr -d " \t" | sort | uniq -c | sort -nr`</sup>

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A `#[cfg(...)]` attribute, or cfg attribute, is used to control the compilation, and therefore execution, of certain code routes. Since a single attribute is often insufficient to fully express the range of control needed, Rust allows you to combine cfg attributes through the following syntax:

`#[cfg(any(x, y, ...))]`

`#[cfg(all(x, y, ...))]`

`#[cfg(not(x))]`

However, repeating combinations of attributes, especially when they become lengthy, can be frustrating and a cause of errors. This is why Rust allows you to provide aliases for cfg attributes.

```rust
//main.rs or lib.rs
#[cfg(my_alias)] = #[cfg(all(some, long, complicated, not(easy, to, remember)))]
```

Aliases must be introduced at the top (top level?) of a crate and can be used anywhere in the crate.

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

//END OF INNER.RS
```

cfg aliases are *not* exposed between crates. (between crates or between modules?)
If in your binary you are using the common pattern of having a `lib` module that gets exported to your `main.rs`, then any cfg alias declared in your `main.rs` will NOT be exposed to the files in your `lib` module. Similarly, any cfg alias declared in your `lib` module will not be exposed to your `main.rs`
binary.

Trying to use a cfg alias declared in another module will result in a compile time error:

```rust
//lib.rs

#[cfg(my_alias)] = #[cfg(any(foo, bar))] // this will compile

mod inner {
  #[cfg(my_other_alias)] = #[cfg(any(foo, bar))] // this will *not* compile
}
```

Reusing an alias name will result in a compile time error:

```rust
#[cfg(foobar)] = #[cfg(any(foo, bar))]
#[cfg(foobar)] = #[cfg(any(x))] // error. foobar already exists

```

As will giving a cfg alias the same name as a preexisting cfg (attribute? flag?):

```rust
//compiled using `--cfg=foobar` flag in rustc

$[cfg(foobar)] = #[cfg(any(foo, bar))] // error. foobar already exists
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Any cfg alias should behave exactly as its expanded, non-aliased form
and be valid wherever any cfg attribute is valid.

Aliases will only be allowed at the top of a crate, `main.rs` for a binary, or `lib.rs` for a library. They will be useable anywhere *within* the crate.

Aliases are not exportable between crates, and a binary that is split into multiple `lib`s (can you just say libraries?) will *not* be able to use or share the cfg aliases from its libraries. (I don't think this paragraph makes sense, try to make it clearer)

Aliases will fail (will cause compile-time errors?) at compile-time if an attribute with the same name already exists, either due to an existing alias, or an existing configuration (existing attribute? or are those not attributes) passed through the `cfg=...` flag in rustc.

# Drawbacks
[drawbacks]: #drawbacks

This RFC introduces another way to construct cfg attributes, potentially making it harder to find why a path of code was compiled or not.

With this design, there is no immediate way to differentiate between an aliased cfg versus one introduced through a compiler or feature flag.

That being said, it is the belief of the author that by blocking the export of cfg aliases, it becomes the responsibility of the developer to be aware of what cfg attributes have been aliased within their crate.

It also the belief of the author that as a user of this feature, I should not need to know the source of a cfg attribute (i.e., alias, intrinsic, or compiler flag). If necessary, a text search within my root module would show me whether the cfg is an alias, and if so, what it aliases.

# Rationale and alternatives
[alternatives]: #alternatives

## Syntax Alternatives

An [existing issue](https://github.com/rust-lang/rfcs/issues/831) proposed the following syntax:

```rust
#![cfg_shortcut(foo, any(super, long, clause))]
```

for introducing "shortcuts" of existing cfg attributes.

This alternative syntax provides the same functionality as the syntax proposed in this RFC, but it introduces the concept of a "shortcut", which would be a new term in the language.

I believe that referring to this concept as an alias rather than a shortcut is a better match for the current terms used in Rust. The syntax I propose also more closely matches the syntax currently used for aliasing types and traits.

Alternatively, aliased cfg attributes could be identified by a special prefix. For example, if we forced aliases to be prefixed with `$` we would get the following:

```rust
#[cfg($foobar)] = #[cfg(any(foo,bar))]

#[cfg($foobar)]
fn conditional_function() {
  //...
}
```

This achieves the same goal as the originally proposed syntax, with the advantage of easier identification of whether a cfg attribute is an alias.

The disadvantage is that now we are exposing what the author believes to be an implementation detail. The fact that the cfg attribute comes from an alias should *not* be important at the moment of use.

This syntax also means that no cfgs will be allowed through a compiler flag that start with `$`.

The author could not find existing examples of Rust blocking prefixes on cfg attribute names.

## Visibility alternatives

The proposed RFC blocks the introduction of cfg aliases anywhere but in the root of the crate (`main.rs` for binaries or `lib.rs` for libraries). An alternative would be to allow cfg aliases to be introduced anywhere and expose those aliases to any child of that module. This will allow for declaration of cfg aliases only where needed.

However, allowing cfg alias declarations anywhere increases the problem of not
knowing where a cfg alias originated, since it could have come from any parent module instead of only at the root.

Implementing the RFC as-is does not prevent this extension from being added later and would not cause breaking change.
The author recommends revisiting this point after the current implementation is stable and we get enough feedback on whether declaring cfg aliases would be useful in non-root modules.

Another alternative would be to allow the cfg alias in any module, but *not* expose it to any other module, child or parent. The author believes this would severely limit the use case of aliasing cfg attributes since the logic
would have to be copied across modules within the same crate.

# Prior art
[prior-art]: #prior-art

There is an [existing issue](https://github.com/rust-lang/rfcs/issues/831) filed in the RFC repository that proposes a change very similar to this one, but with different syntax.

Currently in Rust, we allow the aliasing of structs and enums, with an approved RFC for aliasing traits. This will extend the concept of aliasing to cfg attributes.

# Unresolved questions
[unresolved]: #unresolved-questions

- Does Rust currently have limitations on the names of cfg attributes?
