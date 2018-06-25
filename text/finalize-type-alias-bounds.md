- Feature Name: `finalize_type_alias_bounds`
- Start Date: 2018-06-24
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

[RFC 2071]: https://github.com/rust-lang/rfcs/blob/master/text/2071-impl-trait-type-alias.md

Finalizes the syntax of named existential types in [RFC 2071] as:

+ `type FirstType: BoundA;`
+ `type SecondType = (impl BoundB, impl BoundC);`

and recommends, with a warn-by-default lint, that the two above forms
should be used instead of:

```rust
type FirstType = impl BoundA;

type Temp0: BoundA;
type Temp1: BoundB;
type SecondType = (BoundA, BoundC);
```

# Motivation
[motivation]: #motivation

[2071_alt]: https://github.com/rust-lang/rfcs/blob/master/text/2071-impl-trait-type-alias.md#alternatives

[RFC 2071] introduced to Rust the notion of *named* existential types (hence
*named type* for short). However, the RFC did not resolve the question of syntax,
and chose to provide a temporary syntax instead:
> As discussed in the [alternatives][2071_alt] section above, we will
> need to reconsider the optimal syntax before stabilizing this feature.

This RFC is motivated by that unresolved question and has as its sole purpose
to resolve it.

The rationale for the proposed grammar is discussed in
[that section][alternatives].

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Type alias bounds

[RFC 2071] specified the named type syntax as:

```rust
existential type Adder: Fn(usize) -> usize;
```

we change that syntax by removing `existential` and so the final syntax becomes:

```rust
type Adder: Fn(usize) -> usize;
```

This also applies to specifying associated types in implementations:

```rust
struct Smarty<'a, T>(&'a T);

pub trait Deref {
    type Target: ?Sized; // We could already do this!

    fn deref(&self) -> &Self::Target;
}

impl<T> Deref for Smarty<'_, T> {
    type Target: ?Sized; // And now we can do this as well!

    fn deref(&self) -> &Self::Target {
        self.0.deref()
    }
}
```

## Equals `impl Trait`

In addition to the type alias bounds syntax above, you may also write:

```rust
type Adder = impl Fn(usize) -> usize;

impl<T> Deref for Smarty<'_, T> {
    type Target = impl ?Sized;

    ...
}
```

This is equivalent to what we wrote above.

## Linting towards a uniform style

To retain a uniform style across the ecosystem, we propose that the compiler
provide a *warn*-by-default lint which suggests that the developer use the
syntax that excels most in the given context. Concretely, we propose that:

1. ```rust
   type Foo = impl Bound;
   ```

   should be written as:

   ```rust
   type Foo: Bound;
   ```

2. If a type:

   ```rust
   type Foo: Bound;
   ```

   is *not* exported, and only referred to *once*,
   then the usage should be inlined. This entails that the following snippet:

   ```rust
   type Temp0: BoundA;
   type Temp1: BoundB;
   type MyType = (BoundA, BoundC);
   ```

   should be written as:

   ```rust
   type MyType = (impl BoundA, impl BoundC);
   ```

If this was a `clippy` lint, it would most likely fall under `complexity`.
This category of lints deal with encouraging more readable code.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Grammar

We first define the production:

```
named_type_alias
     : TYPE ident generic_params ":" ty_param_bounds ";"
     ;
```

And to the grammar of `item`, we add:

```
item : visibility? named_type_alias
     | .. // The rest of the `item` production.
     ;
```

And to the grammar of `impl_item`, we add:

```
impl_item : attrs_and_vis maybe_default named_type_alias
          | .. // Rest of the production.
          ;
```

The production `"existential" named_type_alias` in [RFC 2071]
is removed from the language.

## `impl Trait` inside `"type" $ident = "ty_sum" ";"`

The production `ty_sum`, when entering `item_type` and `impl_type` is amended
to accept `impl Trait` (roughly: `"impl" ty_param_bounds`). Note that this is
already accepted in the parser, but rejected later. Those checks should be
removed in the contexts here specified.

In these contexts, an occurrence of `"impl" ty_param_bounds` (hence `I`)
is given semantics by first generating the equivalent tree for:

```rust
"type" $fresh maybe_generics ":" ty_param_bounds ";"
```

where `$fresh` is a fresh and temporary name and where `maybe_generics`
includes each free variable in `I` in order. The occurrence of `I` is then
substituted for the corresponding generated `$fresh` with any free variables
in `I` applied in order.

An example of this desugaring is:

```rust
type Foo<T, U> = (
    Box<u8>,
    impl Display,
    impl Bar<Vec<U>>,
    impl Baz<T>,
    impl Quux<U, T>
);
```

which is desugared to:

```rust
type _F0: Display;
type _F1<U>: Bar<Vec<U>>;
type _F2<T>: Baz<T>;
type _F3<U, T>: Quux<U, T>;

type Foo<T, U> = (
    Box<u8>,
    _F0,
    _F1<U>,
    _F2<T>,
    _F3<U, T>,
);
```

## Semantics and type checking

Everything in [RFC 2071] that applies to:

```
"existential" "type" ident ":" ty_param_bounds ";"
```

now instead applies to:

```
"type" ident ":" ty_param_bounds ";"
```

## Lint

1. Irrespective of whether it occurs as a type alias or inside an `impl`:

   ```rust
   visibility "type" ident maybe_generics "=" "impl" ty_param_bounds;
   ```

   is linted (warn-by-default), suggesting that the user instead write:

   ```rust
   visibility "type" ident maybe_generics ":" ty_param_bounds;
   ```

2. Given a type alias:

   ```rust
   "type" ident maybe_generics ":" ty_param_bounds;
   ```

   If `ident` is only referred to *once* in the same module,
   it is inlined as `"impl" ty_param_bounds` (respecting type variables)
   where it was referred to.

# Drawbacks
[drawbacks]: #drawbacks

There are no drawbacks in finalizing a syntax other than doing so possibly
prematurely.

For a discussion of drawbacks in the particular proposed syntaxes,
read the next section.

# Rationale and alternatives
[alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
