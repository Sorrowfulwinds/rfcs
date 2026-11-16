- Feature Name: `domain_restricted_primitive_integers`
- Start Date: 2026-09-24
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

> **DRAFT STATUS:** Items marked **[R#]** refer to the author review notes in
> Appendix A. Appendix A is not part of the RFC and must be deleted before
> submission. Syntax is illustrative throughout; see Unresolved questions.

# Summary
[summary]: #summary

Add a core "primordial" integer type, `DomainInt<S, [D]>`, whose valid values are an explicit finite set of integer intervals (its *domain*), together with two compiler features that make it practical: compile-time checking of integer literals against the domain, and a built-in easy conversion between domains where one is a subset of the other. 
The domain is a validity invariant, so the compiler can use values outside the domain as layout niches.
Of `DomainInt<S, [D]>`:
`S` "Storage" represents the underlying storage type, that being one of the common power-of-two sizes used by supported architectures 8/16/32/64/128, with a signed `i` or unsigned `u` prefix.
`[D]` "Domain" represents an array of inclusive range expressions to define the legal intervals of this domain.

The code example `Let Foo: DomainInt<u64, [1..=10]>` declares the variable `Foo` which is stored on a unsigned 64 bit integer that can only ever be a value within the inclusive range of 1 to 10 during runtime in safe code.

The code example `Let Baz: DomainInt<i32, [-100..=-1, 1..=100]>` declares the variable `Baz` which is stored on a signed 32 bit integer that can only ever be a value within the inclusive range of -100 to 100 excluding 0 during runtime in safe code.

# Motivation
[motivation]: #motivation

Integer usage in real programs often require an invariant domain narrower than their storage type: a month is `1..=12`, a TCP port is `1..=65535`, a degree angle is `0..=359`, a game skill is `0..=100`, much FFI with C uses positive values, zero, and negative values for different results.
Rust has a handful of limited built-in mechanisms to represent domains, unsigned integers allow you to block out zero to a few power-of-two positive numbers, signed integers lets you block out a few negative to positive power-of-two sets, `NonZero<T>`allows you to exclude zero from those previous sets at the cost of mathimatical operators and loss of support for literal integer assignment.

Current options for replicating some of this proposed type would be:
1. Integers and documentation. You declare your variable and add a comment //0-10
2. Custom Structs. The practical approach you, of course, still have to write out the entire process and constructors for something the compiler *already does* for standard integer types.
3. Crates. A number of crates perform similar functions to this RFC:
    -`deranged`: Proof of concept for ranged integers `RangedU64<0, 10>`
    -`bounded-integer`: A very nice no-std crate allowing you to define integer or even some *enum* domains `BoundedU8<0, 10>`
    -`ranged_integers`: A nightly crate which picks the storage type and calculates domain changes through math operations `Ranged<0, 10>`

However these crates cannot offer features that require the compiler:
- **Literals.** `NonZero<T>` cannot be written as `let n: NonZeroU8 = 5;`; it requires `NonZeroU8::new(5).unwrap()` (or a `const` block). The same limitation applies to every programmer created domain type. A literal's validity can be determined at compile time, or even during writing with rust-analyzer, and writing out `::new()` boilerplate makes code longer and distracts from what is actually occuring.
- **Niches.** The compiler can use out-of-domain bit patterns to make `Option<DomainInt<u8, [0..=10]>>` one byte. Niches are a large reason for the addition of the `NonZero<T>` type and this would do a strong job of cleaning up that types intents and expanding it to more than just single all zero niches.
- **Subset conversions.** `DomainInt<u8, [1..=20]>` always converts losslessly into `DomainInt<u8, [1..=100]>`. The compiler can add traits for easy conversion of any subset domains with just `.into()` and can automatically derive this for any arbitrary set of domains. Knowing you can glaze over `.into()` conversions and that any `try_into()` actually matters is outstanding for code review and clarity.
- **Exaustive Matching.** Allowing the programmer to express the exact domains of an integer, allows the compiler to shorten matching against those domains to just the needed arms. Skipping `_` or `unreachable!()` usage removes further boilerplate.
- **Integer Unification.** `u8`, `i32`, `NonZeroU16`, etc, and any user domain structs are today unrelated types with parallel, seperate APIs. They are all the same thing functionally: a domain over a storage type.

## Motivating examples

*Game state.* A player attribute ranges over `1..=20` and a skill over `1..=100`.
Adding an attribute bonus to a skill should saturate at the skill's maximum, and
nothing should be able to hold a value outside either range.

*Robot control.* A joint controller accepts a 32-bit integer, but each joint has
a physical range. A command outside that range damages hardware. The range
belongs in the type of the joint field so that an invalid command cannot be
constructed in safe code.

## Never-fail APIs

If a function only accepts valid months, its signature can say so:

```rust
fn set_month(m: DomainInt<u8, [1..=12]>)            // cannot fail
// versus
fn set_month(m: u8) -> Result<(), InvalidMonth>     // every caller handles an error
```

Validation moves to the boundary where untrusted data enters the program
(`try_from`, `FromStr`), instead of being repeated at every call. Without
flow-sensitive refinement (a non-goal here) callers with runtime values still
perform one check; the benefit is that the check happens once, at the boundary,
and the type then carries the proof.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Domains

A **domain** is the set of values a type may hold. `u8` has the domain `0..=255`.
`NonZeroU8` has the domain `1..=255`. `DomainInt` lets you name any domain you
like, as one or more inclusive intervals:

```rust
type Attribute = DomainInt<u8,  [1..=20]>;
type Skill     = DomainInt<u8,  [1..=100]>;
type Angle     = DomainInt<i32, [-135..=135]>;
type Scalar    = DomainInt<u32, [0..=0xD7FF, 0xE000..=0x10FFFF]>; // two intervals
```

The first parameter is the **storage**: the machine integer that actually holds
the value, with the layout and ABI of that integer. The second is the domain. A
domain must fit inside its storage; `DomainInt<u8, [0..=300]>` and
`DomainInt<u8, [-1..=5]>` are compile errors.

A rule of thumb: *if you would write the valid range in a comment, write it in
the type.*

## Literals are checked at compile time

```rust
let degrees: DomainInt<u16, [1..=360]> = 5;     // ok
let bad:     DomainInt<u16, [1..=360]> = 400;   // error
```

```
error[E0XXX]: literal `400` is outside the domain of `DomainInt<u16, [1..=360]>`
 --> src/main.rs:3:45
  |
3 |     let bad: DomainInt<u16, [1..=360]> = 400;
  |                                          ^^^ not in [1..=360]
  |
  = help: for values only known at run time, use `DomainInt::<u16, [1..=360]>::try_from(value)`
```

Because a literal is checked against the *expected* type, it also works in
arguments, struct fields, return positions, and `match` arms:

```rust
fn days_in_month(m: DomainInt<u8, [1..=12]>, leap: bool) -> DomainInt<u8, [28..=31]> {
    match m.get() {
        2 if leap => 29,
        2         => 28,
        4 | 6 | 9 | 11 => 30,
        _         => 31,
    }
}
```

Literals only become a `DomainInt` when the expected type is already known to be
one. `let x = 5; let d: Skill = x;` is an error, because `x` is an `i32`.

## Run-time values use `TryFrom` and `FromStr`

```rust
let port = DomainInt::<u16, [1..=65535]>::try_from(raw)?;   // Result
let port: DomainInt<u16, [1..=65535]> = "8080".parse()?;    // validates
```

Safe code can never obtain a value outside the domain. For code that has proved
the value itself, an `unsafe` constructor exists (see the reference section).

## Converting between domains

If one domain is a subset of another, converting is free of checks and spelled
`.into()`:

```rust
let attr: Attribute = 14;
let skill: Skill = attr.into();     // [1..=20] ⊆ [1..=100]
```

Going the other way needs `try_from`, which can fail. Storage types may differ
(`DomainInt<u8, [1..=20]>` into `DomainInt<i64, [1..=100]>`); the check is about
values, not storage.

## Arithmetic

For a **contiguous** domain (one interval) the usual operators work and follow
the existing integer rules relative to the domain's bounds: overflow panics in
debug builds and wraps in release builds. "Overflow" means leaving the domain,
and wrapping means wrapping *within the domain*:

```rust
let skill: Skill = 100;
skill + 1       // debug: panic. release: wraps to 1 (the domain's minimum)
skill.saturating_add(1.into())  // 100
```

All the usual families exist: `checked_*`, `saturating_*`, `wrapping_*`,
`overflowing_*`. `Wrapping<T>` and `Saturating<T>` work as before, so a field
that should *always* saturate can say so in its type:

```rust
struct Arm {
    base:  Wrapping<DomainInt<i32,   [0..=359]>>,     // rotates freely
    elbow: Saturating<DomainInt<i32, [0..=180]>>,     // 180 degrees of travel
    wrist: Saturating<DomainInt<i32, [-135..=135]>>,  // 270 degrees of travel
}

let mut arm = Arm { base: Wrapping(0), elbow: Saturating(0), wrist: Saturating(0) };
arm.base += Wrapping(90);
submit_commands(arm.to_api_format());   // values are 32-bit integers, as the controller requires
```

For a **non-contiguous** domain (several intervals) direct arithmetic is not
provided, because the result of `a + b` can land in a gap. Convert to a
contiguous domain, compute, and convert back with a checked conversion:

```rust
let n: NonZeroI8 = 5;                           // DomainInt<i8, [-128..=-1, 1..=127]>
let wide: DomainInt<i8, [-128..=127]> = n.into();
let sum = wide + 1;
let back = NonZeroI8::try_from(sum)?;           // fails if the sum is 0
```

## How to think about it

- A `DomainInt` is an integer plus a proof that it is in a set. The proof is
  checked once, at the point the value is created.
- Prefer the narrowest domain that is true. Widen only when you need to compute.
- Teaching order: literals; `try_from`/`FromStr`; `Saturating`/`Wrapping`;
  subset `.into()`; non-contiguous domains.
- For learners of Rust this unifies what they already see as unrelated things:
  `u8`, `NonZeroU8`, and a custom range are all points in the same family.
- Maintainability: invariants appear in signatures (`fn set_skill(s: Skill)`)
  instead of comments and defensive checks.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This RFC has two parts. **Part 1** defines `DomainInt` and the two compiler
features. **Part 2** redefines the existing integers in terms of it. Part 2 is
normative for this RFC but may be implemented and stabilized after Part 1
(see *Staging*).

## Part 1: The `DomainInt` type

### Parameters

```rust
pub struct DomainInt<const S: IntKind, const D: Domain>(/* compiler-defined */);
```

- **Storage `S`** is a compiler-known descriptor of one of the primitive integer
  layouts: `i8, i16, i32, i64, i128, isize, u8, u16, u32, u64, u128, usize`.
  It is a descriptor and *not* an ordinary type parameter, so that the primitive
  integer types can themselves be defined as instantiations of `DomainInt`
  without circularity (see Part 2). In user-facing syntax it is spelled with the
  primitive's name (`DomainInt<u8, ...>`). **[R1]**
- **Domain `D`** is an ordered list of inclusive integer intervals. It is a
  compiler-known const kind rather than a user-definable `ConstParamTy`, so this
  RFC does not depend on `adt_const_params` or slice-valued const parameters.
  **[R2]**

### Well-formedness and canonical form

A domain is **canonical** if its intervals are sorted ascending, non-overlapping,
and non-adjacent (`[1..=5, 6..=10]` is non-canonical and equals `[1..=10]`).

1. The compiler normalizes every written domain to canonical form before
   type-checking. `DomainInt<S, D1>` and `DomainInt<S, D2>` are the **same type**
   if and only if `D1` and `D2` have the same canonical form. This holds through
   generics and associated types. **[R3]**
2. A new warn-by-default lint, `non_canonical_domain`, fires on a *written*
   non-canonical domain.
3. Every value in `D` must be representable in `S`; otherwise it is a
   well-formedness error at the use site.
4. An interval whose lower bound exceeds its upper bound is an error. A domain
   with no intervals is the **empty domain**: it is permitted without a warning
   (generic code such as `DomainInt<usize, [0..=N-1]>` is legitimately empty at
   `N = 0`), is uninhabited, and has behavior analogous to `!`. **[R4]**
5. Interval syntax is Rust's inclusive range `a..=b`. Half-open `a..b` is
   accepted and normalized, but `..=` is the documented form.

### Associated constants

```rust
impl<const S: IntKind, const D: Domain> DomainInt<S, D> {
    pub const MIN: S::Prim;          // smallest value in the domain
    pub const MAX: S::Prim;          // largest value in the domain
    pub const IS_CONTIGUOUS: bool;   // true iff exactly one interval
    pub const CONTAINS_ZERO: bool;
    pub const CONTAINS_NEGATIVES: bool;
    pub const CONTAINS_POSITIVES: bool;
}
```

`MIN`/`MAX` are the smallest and largest member of the domain, not of the
storage. These constants are readable in `const` contexts. Without
const-expression `where` clauses they cannot gate impls or methods; behavior
that differs by contiguity is provided by compiler-built-in impls (below).
**[R5]**

### Literals

An integer literal expression whose **expected type** (from the let-annotation,
argument type, field type, return type, or match-arm expectation) is
`DomainInt<S, D>` with concrete `D` has type `DomainInt<S, D>`, and is a
compile-time error if its value is not in `D`. Negative literals (`-135`) are
handled as a single literal, not as unary negation applied to a positive literal.

Rules:

- The literal is checked during const-evaluation of the domain and lowered to a
  constant of the storage type. There is no run-time cost.
- The rule applies only when the expectation is already known at the literal.
  Integer inference variables (`{integer}`) are *not* extended to unify with
  `DomainInt`. `let x = 5; let d: Skill = x;` remains a type error.
- Suffixed literals (`5u8`) keep their primitive type.
- In generic code where `D` is not concrete, a literal cannot be given a
  `DomainInt` type. **[R6]**
- Through a wrapper (`Saturating<DomainInt<..>>`, `Wrapping<..>`), literals work
  where the wrapper's constructor propagates the expectation
  (`Saturating(50)`), but not for `let s: Saturating<Skill> = 50;`.

### Conversions

1. **Subset conversion.** For `DomainInt<S1, D1>` and `DomainInt<S2, D2>` where the
   set of values `D1` is a subset of `D2` (compared as mathematical integers,
   not by storage), the compiler provides a built-in
   `impl From<DomainInt<S1, D1>> for DomainInt<S2, D2>`, and hence `Into`. When
   both types are identical, the existing reflexive `impl<T> From<T> for T` is
   used. The built-in impl applies only when the types differ. **[R7]**
2. **Checked conversion.** For all other pairs,
   `impl TryFrom<DomainInt<S1, D1>> for DomainInt<S2, D2>` returns
   `Err(TryFromDomainError)` for values outside `D2`.
3. **From primitives.** `TryFrom<P>` for every primitive integer `P` returns
   `Err` for values outside `D`. `From<P>` exists exactly when the whole
   domain of `P` is a subset of `D` (this is what makes `u8` and
   `DomainInt<u8, [0..=255]>` interchangeable).
4. **To primitives.** `.get()` returns the storage primitive. There is a
   built-in `From<DomainInt<S, D>> for P` when `D` fits in `P`. `as` casts from
   `DomainInt` are not supported in this RFC. **[R8]**
5. **Parsing.** `FromStr` parses as the storage type, then validates the domain.
6. **Unsafe construction.** `unsafe fn new_unchecked(v: S::Prim) -> Self`. Safety
   contract: `v` is in the domain. Violating it is undefined behavior, because the
   domain is a validity invariant (below).

### Arithmetic and operators

Operators are defined only between two values of the **same** type. Operands of
a different (sub)domain must be converted explicitly with `.into()`. A literal
right-hand operand is checked against the same type as the left-hand operand.
**[R9]**

For contiguous domains, each operation computes the exact mathematical result
and then applies the overflow policy relative to `[MIN, MAX]`. Let
`W = MAX - MIN + 1`.

| Operation family | Result outside the domain |
|---|---|
| `+ - * ...` (operator, debug) | panics (`attempt to add with overflow`) |
| `+ - * ...` (operator, release, `overflow-checks` off) | wraps: `MIN + (r - MIN) mod W` |
| `checked_*` | `None` |
| `saturating_*` | clamps to `MIN` or `MAX` |
| `wrapping_*` | wraps as above |
| `overflowing_*` | `(wrapped, true)` |
| `Wrapping<DomainInt>` | always wraps as above |
| `Saturating<DomainInt>` | always clamps |

When `MIN = S::MIN`, `MAX = S::MAX`, the policy is identical to the existing
primitive behavior. This guarantees that `u8` as `DomainInt<u8, [0..=255]>`
behaves exactly as `u8` does today. **[R10]**

Other operators:

- Division and remainder panic on a zero divisor. A quotient outside the domain
  is an overflow.
- Unary `-` is defined for contiguous domains, and overflows when the negation
  leaves the domain.
- Shifts and bit operations (`!`, `&`, `|`, `^`, `<<`, `>>`) are provided only
  for domains that are the full range of their storage, as there is no natural
  meaning for `!x` in `[1..=20]`. For other domains, convert to the storage
  primitive via `.get()`. **[R11]**
- `Ord`, `PartialOrd`, `Eq`, `Hash` compare the numeric value. Cross-domain
  comparison is not provided.

For **non-contiguous** domains, no operator traits are implemented. The
following are provided:

- `checked_*` methods, returning `None` when the result is outside the domain
  (including when it falls in a gap);
- conversion to the hull, `DomainInt<S, [MIN..=MAX]>`, via the built-in subset
  `Into`;
- `TryFrom` back from any domain.

`saturating_*` is not defined for a general non-contiguous domain, since the
nearest member is ambiguous. **[R12]**

### Layout, ABI, and validity

1. `DomainInt<S, D>` has the same size, alignment, and ABI as the primitive of
   storage `S` (`repr(transparent)`-like). FFI uses the primitive directly.
2. **The domain is a validity invariant.** Producing a value outside the domain
   (via `new_unchecked`, `transmute`, `MaybeUninit::assume_init`, a pointer
   cast, or otherwise) is undefined behavior, exactly as for `NonZero` and `bool`
   today.
3. **Niche selection.** rustc represents a scalar's valid range as a single,
   possibly wrapping, interval of the storage's bit patterns. Given a domain,
   the compiler computes the **largest circular gap**: the largest run of
   storage values, considered cyclically from `S::MAX` around to `S::MIN`, that
   contains no member of the domain. The valid range is the complement of that
   gap. The remaining gaps (smaller holes between intervals) are *not* encoded
   in layout.
4. Therefore, for domains with more than one gap, only the largest gap is a
   validity invariant for layout purposes. Smaller gaps are a **safety**
   invariant: safe code cannot construct a value in them, but unsafe code that
   does is not undefined behavior solely because of that hole, and the compiler
   will not use those values as niches. The API rule remains that every value
   must be in the domain. **[R13]**
5. Examples: `DomainInt<i8, [-128..=-1, 1..=127]>` has exactly one gap
   (`{0}`) and so has the same valid range and niche as `NonZeroI8`.
   `DomainInt<u32, [0..=0xD7FF, 0xE000..=0x10FFFF]>` has the large gap above
   `0x10FFFF` as its niche, matching `char`; the surrogate hole is safety-only.
   `DomainInt<u8, [1..=100]>` makes `Option<_>` one byte.
6. The empty domain is uninhabited. Its size is unspecified in this RFC.

### Traits and std integration

- `Copy`, `Clone`, `Debug`, `Display`, `LowerHex`/`UpperHex`/`Binary`/`Octal`,
  `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`, `FromStr`, `TryFrom`, `Sum`,
  `Product` (checked per the overflow policy), `Step` (contiguous domains only,
  enabling `for x in a..=b`).
- `Default` is implemented only if the domain contains zero, returning zero.
  **[R14]**
- `Saturating<T>`/`Wrapping<T>` gain impls for `DomainInt`; these are
  built-in (not a blanket impl over all `T`) because the existing impls are
  per-primitive. **[R15]**
- `serde` and `clap` integration are for those crates and are out of scope.

### Pattern matching

`match` over a `DomainInt` uses the domain for exhaustiveness: the scrutinee's
possible values are exactly `D`, so a match covering `1..=12` for
`DomainInt<u8, [1..=12]>` is exhaustive without a wildcard arm. This requires
adjusting the exhaustiveness checker's notion of the scrutinee's range, and is
included here because it is a large part of the ergonomic payoff. If it proves
too large it can be split into a follow-up. **[R16]**

### Corner cases

| Case | Behavior |
|---|---|
| Single-value domain `[5..=5]` | Valid; all operations overflow except identity-like results. |
| Domain equal to storage range | Behaves identically to the primitive. |
| Domain at `S::MIN`/`S::MAX` | Wrap and saturate follow the domain bounds, which coincide with storage. |
| `u128` / `i128` bounds | Domain bounds must be representable. Internal representation is unspecified (offset-from-storage-MIN as `u128` is a candidate). **[R17]** |
| `usize`/`isize` | Domain bounds beyond the target's pointer width are errors; code portable across targets must use bounds valid on all targets. **[R18]** |
| Empty domain | Allowed, uninhabited, no literals or constructors succeed. |
| Generic `D` | `MIN`/`MAX` and subset conversions are evaluated after monomorphization; literals are disallowed. |

## Part 2: Unification of the existing integer types

Part 2 states the following equalities as *type identities*:

| Existing type | Defined as |
|---|---|
| `u8` | `DomainInt<u8, [0..=255]>` |
| `i8` | `DomainInt<i8, [-128..=127]>` |
| `NonZeroU8` | `DomainInt<u8, [1..=255]>` |
| `NonZeroI8` | `DomainInt<i8, [-128..=-1, 1..=127]>` |
| ... and all other widths | by the same pattern |

Primitive integer types remain language items and keywords; the change is that
the compiler treats them as *equal to* the corresponding `DomainInt`, not that
the keywords disappear. Consequences:

1. **Behavior is unchanged.** Arithmetic on `u8` is exactly the full-range
   case in the table above. `NonZero*` keeps its existing API (see 4). Layout,
   niches, ABI, and `transmute` rules are identical to today.
2. **`{integer}` inference and fallback** are unchanged: an integer variable still
   resolves only to primitive types, defaulting to `i32`.
3. **Path resolution** (`u8::MAX`, `i32::from_str_radix`) continues to work via the
   inherent impls on the full-range instantiations.
4. **`NonZero<T>`.** `NonZero<T>` is generic over `T: ZeroablePrimitive`, and the
   domain depends on `T`. To avoid requiring `generic_const_exprs`, `NonZero<T>`
   becomes an alias through the existing sealed trait:

   ```rust
   pub type NonZero<T> = <T as ZeroablePrimitive>::NonZeroRepr;
   // impl ZeroablePrimitive for u8 { type NonZeroRepr = DomainInt<u8, [1..=255]>; }
   ```

   This is observably different from a struct: user code that implements a trait
   for `NonZero<T>` generically would no longer compile. This is a known
   compatibility cost; a crater run is required. **[R19]**
5. **Inherent methods** specific to `NonZero` (`new`, `get`, `new_unchecked`,
   `checked_add`, `saturating_add`, `leading_zeros`, ...) are provided by inherent
   impls on the specific instantiations in `core` (permitted in the defining
   crate), or by built-in methods gated on the domain's properties. **[R20]**
6. **Non-contiguous arithmetic** matches current `NonZero` behavior, which has no
   general `Add` operators and provides `checked_*`.
7. **Diagnostics and `Debug` output** for `u8` and `NonZeroU8` are unchanged:
   the compiler prints the alias/primitive name when the type is a known
   standard instantiation. **[R21]**

### Staging

1. **Stage 1:** implement `DomainInt` (parameters, canonicalization, consts,
   literals, conversions, arithmetic, layout, patterns) as unstable.
2. **Stage 2:** add compatibility assertions (`size_of`, niches, `ABI` tests) that
   `DomainInt<u8,[1..=255]>` matches `NonZeroU8` exactly.
3. **Stage 3:** unify the types per Part 2, behind a feature gate, run crater.
4. **Stabilization** may occur per stage; Part 2 requires a clean crater result.

# Drawbacks
[drawbacks]: #drawbacks

- **Compiler surface.** Literal checking against an expected type, a built-in
  const-evaluated subset impl, canonical domain types, and exhaustiveness changes
  are each significant.
- **Type explosion and API friction.** Libraries must pick domains; conversions
  between near-identical domains accumulate.
- **Overlap with other work.** Pattern types and `NonZero` generalization cover
  part of this space and risk fragmenting the ecosystem.
- **Arithmetic cost.** Wrapping within a non-power-of-two domain requires a
  modulo rather than a mask, and plain `+` now behaves differently in debug and
  release for narrow domains.
- **Compile-time cost.** Domains are interval sets that are const-evaluated and
  compared during type checking.
- **Unification risk.** Redefining the primitives raises the cost of any mistake
  and touches inference, resolution, diagnostics, and `NonZero<T>`'s public shape.
- **Split invariant** for multi-gap domains (largest gap validity, others safety)
  is subtle and could surprise unsafe code authors.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why this design

- **One concept.** Width plus a set of valid values describes `u8`, `NonZeroU8`,
  and a custom range alike. Unifying removes parallel APIs and lets one set of
  rules (literals, conversion, overflow) apply to all.
- **Disjoint unions are required for unification.** `NonZero` is the disjoint
  domain `[MIN..=-1, 1..=MAX]`; single-interval designs cannot express it, so they
  cannot subsume existing types.
- **Validity invariant** gives niches and compiler knowledge, and matches how
  `NonZero` and `char` already work.
- **Explicit conversions** keep the type system predictable and avoid
  type-level interval arithmetic.

## Alternatives

1. **Crates only** (`deranged`, `bounded-integer`, `nutype`). No compiler changes,
   but no literal checks, no niches beyond what attributes allow, no unification.
2. **Pattern types as the primitive**, with `DomainInt` as library sugar on top.
   This is the strongest alternative: pattern types already provide a
   compiler-level "integer restricted by a pattern" with layout support. This RFC
   would then reduce to a library API plus literal inference plus the unification
   story. The reasons to define `DomainInt` instead are the const-evaluated subset
   relation, the unification of primitives, and a stable domain parameter;
   whether those justify a separate primitive is a central question for review.
   **[R22]**
3. **Generalize `NonZero<T>` to `Niche<T, const NICHE: T>`** or similar
   single-value niches. Cheaper, but cannot express ranges.
4. **Full refinement types** (arbitrary predicates plus a solver). Far more
   expressive and far larger; changes Rust's cost model.
5. **User-extensible literals only**, leaving ranges to libraries. Fixes the
   `NonZero` literal complaint but no unification or niches.
6. **Safety-only invariant** (no niches). Simpler (no validity rules, no
   split-invariant discussion), but loses a main benefit and the layout
   compatibility needed to subsume `NonZero`.

## Impact of not doing this

Status quo: parallel hand-written types and crates, no compile-time literal checks
for any constrained integer, and `NonZero` remains a special case.

# Prior art
[prior-art]: #prior-art

**Languages**

- **Ada** is the closest ancestor. `subtype Skill is Integer range 1 .. 100;`
  gives range-constrained subtypes with `Constraint_Error` on violation, and
  `type Angle is mod 360;` gives wrapping types. **Ada 2012** adds
  `Static_Predicate`, which permits non-contiguous sets such as `1 .. 5 | 10 .. 20`.
  Disjoint domains are therefore *not* novel; what this RFC adds is the
  integration with Rust's layout/niche model and unification with `NonZero`.
  Lesson: Ada's constraint checks are mostly run-time, and predicates were added
  because ranges alone were not enough, which supports including disjoint
  sets from the start.
- **SPARK** (Ada subset with proof) shows range types combined with static
  proof; the "domain checker" future direction resembles it.
- **Pascal, Modula-2, Delphi** subrange types, and **Nim** `range[1..20]`.
  Mostly run-time checked; Nim checks literals at compile time.
- **Zig** has arbitrary-width integers (`u3`) but no value-range types; relevant
  to the `S` descriptor design.
- **Refinement-type systems** (Liquid Haskell, F\*, Dafny, Idris/Agda `Fin n`)
  express arbitrary predicates; different cost model.
- **C++**: `boost::constrained_value`, Boost.SafeNumerics; run-time checked.

**Rust today**

- `NonZero<T>` (generic since 1.79), `Wrapping<T>`, `Saturating<T>` (stable
  since 1.74).
- `char` has a restricted valid range in layout.
- Compiler-internal niche types and the perma-unstable
  `rustc_layout_scalar_valid_range_start/end` attributes. `core` also has an
  internal module of niche-restricted integer types. These are what rustc already
  uses to implement what this RFC exposes. **[R23]**
- **Pattern types** experimental work.
- Crates: `deranged` (used by `time`), `bounded-integer`, `nutype`,
  `ranged_integers`, `intervals`-style crates.

**What to research and summarize here** (expected by reviewers):

1. Earlier RFCs and internals threads on ranged integers, refinement types, and
   user-defined literals, and why each stalled. **[R24]**
2. The current status of pattern types and any project goal covering them.
3. Experience reports from `deranged`/`time` on API shape and conversions.
4. The `NonZero<T>` generalization discussion, for the compatibility concerns.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

**To resolve in the RFC process**

- Whether to build on pattern types (alternative 2) or define `DomainInt` as an
  independent primitive.
- Final syntax for storage and domain (`DomainInt<u8, [1..=20]>` versus
  `u8 in 1..=20`-style sugar, or a `domain!` macro), and how `IntKind` is spelled.
- Whether release-mode wrapping inside the domain is the right default for plain
  operators, or whether plain operators should panic always for narrow domains.
- Whether Part 2 (unification) is part of the initial acceptance or a separate
  follow-on RFC gated on Part 1.
- The exact split-invariant policy for multi-gap domains.

**To resolve during implementation**

- Domain representation (`u128`/`i128` bounds, `usize` portability).
- Interaction with `generic_const_exprs`-like features for generic domains.
- Diagnostics and error codes, lint naming.
- Size and layout of the empty domain.
- Exhaustiveness checker integration, or whether it moves to a follow-up.

**Out of scope**

- Flow-sensitive refinement (`if x < 361 { let d: Dom = x; }` still needs
  `try_from`).
- Automatic domain inference across function bodies ("domain checker").
- Cross-domain operators with computed result domains.
- Arbitrary-width integers.
- Domains for `char`, floats, and enums.

# Future possibilities
[future-possibilities]: #future-possibilities

- **Domain checker / interval analysis.** Compute result domains of arithmetic
  at compile time. Caution: Rust treats function signatures as the API boundary;
  inferring return domains from bodies would leak implementation into the
  interface and create semver hazards, so any such feature should be opt-in
  at signatures.
- **Safe indexing.** `DomainInt<usize, [0..=N-1]>` indexing `[T; N]` without bounds
  checks, depending on generic domains.
- **Enum domains.** Restricting a type to a subset of an enum's variants.
  Related to enum-variant types; discriminants are not layout-guaranteed
  without `repr`, so this needs its own design.
- **Multi-range validity in layout**, letting every gap be a niche.
- **More predicates** (multiples of N, powers of two), `char` and float domains.
- **Arbitrary-width storage** when Rust gains arbitrary-width integers.
- **User-visible literal trait**, generalizing the special-cased literal rule to
  user types.

---

# Appendix A: Author review notes (delete before submission)

## A. Design decisions you must make

**R1: Storage as a descriptor, not a type.** I earlier suggested `DomainInt<i32, ..>`
with a type parameter. That conflicts with unification: if `u8` is defined as
`DomainInt<u8, [0..=255]>`, the storage `u8` in the definition is itself the type
being defined (circular). Your original instinct, `S` as a non-type value, is
needed for Part 2. The draft keeps the `u8` *spelling* as sugar. Decide whether
that sugar is acceptable or whether the RFC should expose `IntKind::U8`-style
syntax.

**R22: Pattern types.** Before submission, determine the current state of
pattern types and whether `DomainInt` should be defined as sugar over them. If
reviewers see two overlapping primitives they will ask for justification. Your
argument in this draft is: unification, const-evaluated subset relation, and a
stable domain parameter. Check that this is still true.

**Unification in or out (R19/R20/R21).** Part 2 is the highest-risk part: it
changes `NonZero<T>` from a struct to a projection alias (breaking for user
generic impls over `NonZero<T>`), touches inference and diagnostics, and requires
crater. I kept it in per your instruction and staged it. Consider whether the RFC
is more likely to be accepted if Part 1 is accepted first, with Part 2 as a
committed follow-up.

**R10: Release-mode wrapping.** Wrapping within a non-power-of-two domain costs a
modulo. It also makes `x + y` differ from `Wrapping<DomainInt>` only in debug.
Alternatives: always panic on overflow for non-full-range domains, or make
`overflow-checks` apply regardless of profile for narrow domains. I kept your
chosen behavior but reviewers will push on it.

**R13: Split invariant.** The largest-gap rule depends on rustc's layout model
being a single wrapping interval. Verify in the current rustc layout code. If
that is wrong, the section simplifies or changes. State plainly to unsafe-code
authors which holes are UB and which are not.

## B. Things to verify (I have not re-checked these)

- **R2:** Current status of `adt_const_params`, slice-valued const params, and
  `generic_const_exprs`.
- **R13:** rustc's scalar valid range representation; that `char`'s valid range is
  `0..=0x10FFFF` only; the exact wrapping semantics.
- **R19:** Whether `NonZero<T>` is currently a struct over `T::NonZeroInner` (it
  was reshaped in 2024); check current form and `ZeroablePrimitive`.
- **R23:** Present names and stability of the niche-type internals
  (`core::num::niche_types` or equivalent) and `rustc_layout_scalar_valid_range_*`.
- **R24:** Search rust-lang/rfcs and internals.rust-lang.org for: ranged integers,
  refinement types, pattern types, user-defined literals, `NonZero` generalization.
  I did not cite specific RFC numbers because I am not certain of them.
- Stabilization versions I cited (`NonZero<T>` 1.79, `Saturating` 1.74).
- The Ada 2012 `Static_Predicate` syntax against the current Ada reference manual.

## C. Smaller design questions

- **R3:** Type identity through canonicalization in generic contexts, associated
  types, and trait selection. Specify exactly where normalization happens.
- **R4:** Empty domain: size, `Default`, and the `!`-like behavior. Is it legal to
  have `Option<EmptyDomainInt>`?
- **R5:** The consts cannot gate impls without const-expression `where` clauses.
  Check which of the Part 2 `NonZero` methods depend on gating.
- **R6:** Literal rule is expectation-only. Check method-call receivers,
  closures, and generic function arguments where the expectation is an inference
  variable.
- **R7:** Overlap of built-in subset `From` with the reflexive blanket impl and
  with user impls (`impl From<MyType> for DomainInt<..>`); orphan rules.
- **R8:** Should `as` casts be allowed? They silently discard the domain.
- **R9:** Literal RHS with offsets: `skill + 0` is invalid if `0` is outside the
  domain. Consider a `.add_raw(i32)`-style API or an offset-type convention.
- **R11:** Bitwise operations on narrow domains are disabled. Check this against
  how `NonZero` implements `BitOr` today.
- **R12:** `NonZero` has `saturating_*` today for some types; its saturating
  result is unambiguous only because the gap is one value at the edge of the sign.
  Decide how Part 2 retains these methods without generalizing saturation.
- **R14:** `Default` for domains that exclude zero.
- **R15:** `Wrapping`/`Saturating` impls are per-primitive macros; extending them
  to `DomainInt` may require a trait or built-in impls.
- **R16:** Exhaustiveness is potentially a separate RFC.
- **R17/R18:** Bound representation, `usize` portability.
- **R20/R21:** Inherent impls on concrete instantiations, and keeping
  `NonZeroU8` as the displayed name in diagnostics.

## E. Examples to consider adding (with relevance notes)

- **`deranged` / `time` crate:** the strongest real-world precedent for month,
  hour, minute. Study its API for conversion ergonomics.
- **`char` as `DomainInt<u32, [...]>`:** shows the largest-gap rule against a
  real type. Note that `char` itself is not proposed to change.
- **`std::cmp::Ordering`** (`-1..=1`) as a domain-backed enum: illustrates the
  enum-domain future direction.
- **Ports and HTTP status codes:** common, easy to read.
- **`Layout` alignment** (power of two): a deliberate non-example of what
  intervals cannot express.
- **Array indexing:** `DomainInt<usize, [0..=N-1]>` as the stretch example.
- **Before/after code size:** a measured comparison of a hand-written newtype with
  all operator impls versus the `DomainInt` version would be a strong addition
  to Motivation.
