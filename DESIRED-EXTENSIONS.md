# Specialization - desired extensions and neighboring features

This file is the design-space companion to [DESIRED-USE-CASES.md](DESIRED-USE-CASES.md). Where that
doc records *what* people want to do and *why*, this one records the *feature extensions* those
wants would need: capabilities of `specialization` (or of a neighboring feature) that do not exist
today. Each is cross-cutting - one extension typically serves several of the uses - so they are
organized by the shape of the missing capability, not by any single use.

Each entry states what the extension would add, what it asks of the design, and which demand it
serves (linking the use in DESIRED-USE-CASES.md). Code is pseudo-Rust: it does not compile on any
channel, and is written to show the shape of the requested behavior.

The entries fall into three groups: extensions to specialization as designed; overlaps its ordering
rule cannot resolve; and neighboring features the demand lands on when it turns out not to be
specialization at all. A final section records design properties and affordances that are worth not
losing, and maps RFC 1210's limitations and extensions onto the doc set.

**Note:** This document has been written by an LLM and has not yet undergone human review or editing.

---

# Part I - Extensions to specialization as designed

These are things that were mostly expected from specialization, or floated as its possible
extensions, and never built.

## Specializing an inherent method without inventing a trait

Serves: the std-fast-path and precise-`size_hint` uses (DESIRED-USE-CASES.md), and most of the
shipped `Spec*` helper-trait boilerplate.

Specialization applies to trait impls. An inherent method cannot be specialized, so every use that
wants one routes through a helper trait invented for the purpose.

RFC 1210's appendix proposes the desugaring that would allow it directly, and picks the same example
its Motivation section uses:

```rust
impl<T, I: IntoIterator<Item = T>> Vec<T> {
    default fn extend(&mut self, iter: I) { /* push one at a time */ }
}
impl<T: Copy> Vec<T> {
    fn extend(&mut self, slice: &[T]) { /* one memcpy */ }
}
```

**What it asks of the design:** that inherent items be specializable, on the folklore reading that
an inherent item is an anonymous single-item trait. The RFC notes this would also remove the need to
refactor `Extend` into a shape that takes the iterator as a trait parameter, which is the refactor
that cost it the ability to say "extendable by *any* iterator" (see the HRTB note below).

## Making a trait's own default body overridable

Serves: the precise-`size_hint` use (DESIRED-USE-CASES.md).

A trait's provided method body is not specializable. To make one overridable, an impl has to restate
it, because `default` is a property of an impl item and a trait-provided body is not one.
[#68309](https://github.com/rust-lang/rust/issues/68309) asks for a forwarding form that marks the
inherited body overridable without copying it; the syntax has parsed since #67131 but has no
lowering.

```rust
impl<T> Iterator for MyThing<T> {
    fn next(&mut self) -> Option<T> { /* ... */ }
    default fn size_hint;        // inherit the trait's body, but allow an override below
}
```

The demand that motivates it is RFC 1210's "refined defaults" example, where a subtrait supplies a
better body for a supertrait method:

```rust
default impl<T> Iterator for T where T: ExactSizeIterator {
    fn size_hint(&self) -> (usize, Option<usize>) { (self.len(), Some(self.len())) }
}
```

**What it asks of the design:** that `default` reach trait-provided bodies, not just impl items.
Without it, every use of this shape either duplicates a body or invents a parallel trait, and the
duplicate has to be kept in sync with the original by hand.

Worth recording alongside it: RFC 1210's two flagship reuse examples, `Add`/`add_assign` and this
one, have never compiled, for an unrelated reason. A `default fn` may not rely on a sibling `default
type` in the same impl, since an override may replace one and not the other. That is already tracked
within open issues ([ISSUES.md](ISSUES.md) root cause 3).

## Reusing the body that was overridden

Serves: the error-propagation and any "default plus something" use (DESIRED-USE-CASES.md).

An override replaces a body; it cannot extend one. Where the specialized behavior is "the default,
plus something", the override has to reproduce the default - `msica`'s override fallback arm,
`_ => CustomActionResult::Failure`, is the blanket impl's entire body, copied.

```rust
impl<E: Error> FromResidual<Result<Infallible, E>> for CustomActionResult {
    default fn from_residual(r: Result<Infallible, E>) -> Self { CustomActionResult::Failure }
}
impl FromResidual<Result<Infallible, Error>> for CustomActionResult {
    fn from_residual(r: Result<Infallible, Error>) -> Self {
        let err = r.unwrap_err();
        match err.kind() {
            ErrorKind::ErrorCode(code) => CustomActionResult::from(code.get()),
            _ => super::from_residual(Err(err)),   // the body this impl overrode
        }
    }
}
```

Overrides that end in "otherwise, whatever the default did" are common enough in the index that the
duplication is systematic rather than incidental: it is the same cost the "duplication to dodge"
pattern imposes on crates that cannot specialize at all, moved inside a crate that can.

RFC 1210 raises this as a possible extension and observes that it is well-defined: all impls
overlapping a given one are totally ordered, so "the next one up" always names exactly one body.

**What it asks of the design:** a way to name the overridden item. It is cheap under the chain rule
and stops being well-defined under the lattice rule, where the overlapping set is no longer a chain,
which makes it one of the few places where two candidate extensions conflict.

## Requiring a bound on a `default type`

Serves: any richer-library use whose `default type` must cross a crate boundary (DESIRED-USE-CASES.md).

`default type` is all-or-nothing. Downstream code either knows the associated type exactly or knows
nothing about it, and since a `default type` is not projectable, generic code that only needs *a
bound* on it is stuck. RFC 1210 flags this in its extensions section: it would occasionally be
useful to say that every further specialization will at least guarantee some additional bound.

```rust
trait Decode {
    type Out;
    fn decode(&self) -> Self::Out;
}
impl<T: Read> Decode for T {
    default type Out: Debug = Vec<u8>;   // overridable, but every override must be Debug
    default fn decode(&self) -> Self::Out { /* ... */ }
}

fn log<T: Read>(t: &T) {
    println!("{:?}", t.decode());        // legal: Out is unknown, but it is Debug
}
```

The demand arrives most often through async traits. On internals, a request for a default async
method whose return type is an associated `impl Trait` stalls exactly here: *"this won't work as
`Self::FooMany` is still considered unknown by compiler"*
([#15058](https://internals.rust-lang.org/t/specialization-associated-types-again-and-type-alias-impl-taits/15058)).
The same collision is live in the compiler: RPITIT desugars to an associated type, which smuggles
`default type` past `min_specialization` ([#108309](https://github.com/rust-lang/rust/issues/108309)).
`constriction`'s second TODO is the same wish in return position: *"return `impl ExactSizeIterator` …
once specialization is stable"*. Today a `default type` is not projectable at all, which is
[ISSUES.md](ISSUES.md) root cause 2.

**What it asks of the design:** a bound on an overridable associated type that survives without
normalization. This is the one extension that would make `default type` usable across a crate
boundary at all, since today the type is opaque and useless to any caller that did not pick it.

## Overriding an upstream blanket, including the standard library's

Serves: the std-fast-path use, and any cross-crate override (DESIRED-USE-CASES.md).

`msica` can specialize `FromResidual` only because it owns both the trait impls and the target type;
getting the same on `Result` would require `std` to mark its own method `default fn`. Two further
asks sit on top of that.

The first is that overriding should not require the overriding crate to enable an unstable feature.
[#58659](https://github.com/rust-lang/rust/issues/58659) is blocked on precisely this: a blanket
`FromIterator` for `Default + Extend` cannot ship, because a downstream crate cannot override an
upstream default impl without itself turning on `min_specialization`.

```rust
// upstream, once:
default impl<A, T: Default + Extend<A>> FromIterator<A> for T {
    fn from_iter<I: IntoIterator<Item = A>>(iter: I) -> T { /* default + extend */ }
}

// downstream, on stable, with no feature gate of its own:
impl<A> FromIterator<A> for MyBag<A> {
    fn from_iter<I: IntoIterator<Item = A>>(iter: I) -> Self { /* sized up front */ }
}
```

The second is the mirror image, and it is the one `std` currently forecloses by design. `std`'s
speed specializations are keyed on private marker traits, so a user type can never reach them. The
clearest statement is on internals, asking for the fd-to-fd copy path: *"given a pair of
AsyncRead/AsyncWrite implementations, if both are file descriptors there are ways to speed up copying
between them"*, with the meta-goal of *"help[ing] user-defined container and iterator types to enjoy
the kind of optimizations that `std` containers and iterators have been getting for a long time"*
([#20836](https://internals.rust-lang.org/t/unsafe-specialization/20836)). The 16 `std`-mirror rows
in [USE-CASE-INDEX.md](USE-CASE-INDEX.md) are the ecosystem's answer: vendor the container, get the
fast path.

**What it asks of the design:** that a downstream override be ordinary stable code, and a decision
about API-transparency. `std` keeps `TrustedLen` and friends unimplementable so that no program can
depend on the specialization firing; opening them up is what this ask wants, and it is a different
question from whether the feature is sound. The same respondent who raised the copy case noted that
conditionally implementing an unsafe spec-trait on a *safe* trait turns the safe trait into a source
of unsoundness, so this one is blocked on the soundness hole specifically, not on stabilization
timing.

---

# Part II - Overlaps the chain rule cannot order

Specialization accepts overlap only when one impl matches a subset of the other's types. Two whole
families of demand are partial overlap, where neither impl is a subset, and the chain rule has
nothing to order.

## The lattice rule: impls that overlap without either being more specific

Serves: the value-or-closure (`ByNeed`) and fixed-size-array uses (DESIRED-USE-CASES.md).

RFC 1210's Limitations section is a list of these, and the RFC is explicit that *none* of them are
permitted by the rule it proposes. The `ByNeed` example is the sharpest, because it is an API people
still want:

```rust
trait ByNeed<T> { fn compute(self) -> T; }

impl<T> ByNeed<T> for T { fn compute(self) -> T { self } }
impl<F, T> ByNeed<T> for F where F: FnOnce() -> T { fn compute(self) -> T { self() } }

impl<T> Option<T> {
    fn unwrap_or<U: ByNeed<T>>(self, def: U) -> T { /* one method, not two */ }
}
```

These overlap at `F: FnOnce() -> F`, in both directions. The fix under a lattice rule is to supply
the intersection impl; under the chain rule there is nothing to write. The other RFC examples are
`AsRef<T> for T` alongside the lifting impl over `&T`, and `PartialOrd for T where T: Ord` alongside
the lifting impls over references.

The demand is not historical. [#45742](https://github.com/rust-lang/rust/issues/45742) is the
`AsRef`/`AsMut`-over-`Deref` blanket, still open, and is the only one of the specialization-blocked
`std` API issues that genuinely needs a new rule rather than a stabilization.
[#94313](https://github.com/rust-lang/rust/issues/94313) is `[T; 0]` not being `Copy`/`Clone` for
non-`Copy` `T`. On the forums it is asked in the abstract (three impls, for `T: A`, `T: B`, and
`T: A + B`) and answered *"This is the so-called lattice rule … It's not implemented and there are no
plans to implement it"*
([#55818](https://users.rust-lang.org/t/specialization-and-partially-overlapping-impls/55818)).

```rust
impl<T: A> Render for T      { default fn render(&self) -> String { /* via A */ } }
impl<T: B> Render for T      { default fn render(&self) -> String { /* via B */ } }
impl<T: A + B> Render for T  { fn render(&self) -> String { /* the combined form */ } }
```

**What it asks of the design:** the lattice rule, which RFC 1210 says is backwards compatible to add
later and then declines, for a reason that is still the reason:

```rust
impl<T, U> Foo for (T, U) where T: 'static {}
impl<T, U> Foo for (T, U) where U: 'static {}
impl<T, U> Foo for (T, U) where T: 'static, U: 'static {}
```

By codegen there is no impl to choose: not enough information to specialize, and no unspecialized
winner either. The lattice rule is therefore not an independent extension: it is downstream of
whatever is decided about lifetime-dependent dispatch.

## Const specialization: overlap decided by a const value

Serves: the recursive-base-case (matrix determinant) and fixed-size-array uses (DESIRED-USE-CASES.md).

Two shipped rows key on a const, and both do it by naming a concrete type: `unroll-fn` terminates a
loop-unrolling recursion with `impl UnrollImpl for Const<0>` against `impl<const N: usize> UnrollImpl
for Const<N>`, and `discrete_system` overrides `[T; L]` with `[T; 1]`. That works because `Const<0>`
and `[T; 1]` *are* concrete types. What does not exist is overlap on a predicate over the value.

```rust
impl<T, const N: usize> Determinant for Matrix<T, N> {
    default fn det(&self) -> T { /* recurse: split into (N-1)-minors */ }
}
impl<T, const N: usize> Determinant for Matrix<T, N> where N <= 3 {
    fn det(&self) -> T { /* closed form */ }
}
```

That is the matrix-determinant thread nearly verbatim: *"For bigger matrices, I want to (using
`specialization`) use a recursive algorithm that splits up the matrix up into smaller matrices"*
([#106819](https://users.rust-lang.org/t/recursive-const-generics-and-specialization/106819)), where
the recursion also has to survive `{DIM-1}` underflowing at `DIM == 1`.

The other half of the const demand is the array-size case, which is a lattice problem wearing a const
hat: *"many core traits (`Default`, `Distribution<[_;_]>`, `Serialize`/`Deserialize`) … are not
implemented for arrays"*, because `[T; 0]`'s unconditional impl and a bounded `[T; N]` impl overlap
without either being a subset
([#18212](https://internals.rust-lang.org/t/proposal-const-specialization/18212)).

**What it asks of the design:** whether the specialization graph can order impls by anything other
than type structure. A const predicate needs the ordering to consult const evaluation, which is a
different question from the one the graph answers today.

---

# Part III - Neighboring features the demand lands on

These are consistently voiced as specialization requests and are not specialization at all. Each
needs a different feature. They belong in the survey because the demand arrives at specialization's
door, and because designing specialization without knowing they are separate invites the category
error of counting them as covered.

## Negative reasoning: dispatch on a trait the type does *not* implement

Serves: the error-conversion use (DESIRED-USE-CASES.md).

A crate wants every backend error to become its own error type so that `?` just works, and collides
with `core`'s reflexive impl.

```rust
// core, for every type:
impl<T> From<T> for T { fn from(t: T) -> T { t } }

// the wish, in a driver crate:
impl<E: Error> From<E> for MyError where E: !Same<MyError> {
    fn from(e: E) -> MyError { MyError::Backend(e.to_string()) }
}
```

It recurs across [#46368](https://users.rust-lang.org/t/impl-t-from-t-for-myerror-conflict/46368),
[#124913](https://github.com/rust-lang/rust/issues/124913),
[#58904](https://github.com/rust-lang/rust/issues/58904), and the `Into`/`TryFrom` variants, and the
usual answer is "specialization will eventually fix this". It will not: neither impl is a subset of
the other, so there is no graph edge to record. What ships instead is marker-trait forgery:
`syllogism`'s `IsNot<T>`, `tea-codec`'s `Equality<X, Y>: NotEqual`, and the `False`-trait encodings
people write by hand, one of which was reported as *"the types that should be accepted are also being
rejected"*
([#24412](https://internals.rust-lang.org/t/negative-trait-bounds-using-feature-specialization/24412)).

The bound-position form of the same ask is an API that accepts only types *without* a capability:
`fn take<T: !Copy>(t: T)`, to enforce a move-only invariant.

**What it asks of the design:** negative trait bounds, tracked as
[#42721](https://github.com/rust-lang/rust/issues/42721) (bound side) and
[#46813](https://github.com/rust-lang/rust/issues/46813) (impl side). Nightly's `negative_bounds` is
half-built: coherence accepts a negative bound but the solver cannot prove one, so the impl is
uncallable. RFC 1210 considered negative bounds as the main alternative to specialization and
rejected them as *fundamentally closed*: they let a trait author specialize up front but not a
downstream crate. The objection is stability, not mechanics: with negative reasoning, adding an impl
becomes a breaking change.

## Type-level computation over type identity

Serves: the container-indexed-by-type use (DESIRED-USE-CASES.md).

`option_trait` shows how far type-level machinery gets with `default type`: a generalized `Option`,
computed at compile time. What people ask for next is a step past it: a type-level set or map, where
a value is fetched *by its type*.

```rust
trait Set { fn get<T>(&self) -> Option<&T>; }

impl Set for () { fn get<T>(&self) -> Option<&T> { None } }

impl<S: Set, H> Set for (S, H) {
    default fn get<T>(&self) -> Option<&T> { self.0.get::<T>() }  // keep walking outward-in
}
impl<S: Set, H> Set for (S, H) {
    fn get<H>(&self) -> Option<&H> { Some(&self.1) }              // when the query type *is* H
}
```

The two impls have identical headers and differ only in whether a *method* parameter is pinned to an
*impl* parameter, so there is nothing for the graph to order. That is why the thread that asks for it
(*"I want to do some sort of type level Set"*,
[#23271](https://internals.rust-lang.org/t/yet-another-stupid-thought-about-specialization/23271))
gets no working answer. The same request goes on to want `&'static str` distinguished from `&'a str`,
which is the region-erasure hole itself: the compiler cannot branch on whether `'a: 'b` holds, so it
either declines to know or assumes and defers to the borrow checker.

**What it asks of the design:** a decision procedure over type identity, evaluated where impl
selection cannot reach. The lifetime half is not an extension of specialization but the same wall
specialization is stuck behind, seen from the other side.

## Bounds on `Drop`, with `needs_drop` agreeing

Serves: the bound-dependent-cleanup use (DESIRED-USE-CASES.md).

`linux-support` routes `Drop` through a `SpecDrop` trait because a `Drop` impl cannot carry bounds
(E0367). The stated wish has a second half the workaround cannot fake: the compile-time answer has to
change too.

```rust
impl<T: Handle> Drop for Guard<T> {
    fn drop(&mut self) { self.release() }
}

const _: () = assert!(!mem::needs_drop::<Guard<u8>>());   // u8: !Handle, so no glue at all
```

*"it would be nice to allow `Drop` to be implemented for specialized types"* … with
`mem::needs_drop()` reporting `false` for the no-drop case
([#12873](https://internals.rust-lang.org/t/can-we-fix-drop-to-allow-specialization/12873)).

**What it asks of the design:** [#46893](https://github.com/rust-lang/rust/issues/46893) records why
this is not a specialization question. `needs_drop` feeds negative reasoning around `Copy`, so
allowing it to vary with generics is the blocker, not the `Drop` impl's bounds. The routing through a
helper trait is a workaround for a missing capability, and the capability - bounds on `Drop`, or an
overridable destructor - is the thing to build.

## Varying an impl along constness

Serves: the const-eval-available-fast-path use (DESIRED-USE-CASES.md).

Constness is not a dimension the specialization graph can vary along, in either direction. A
non-const impl is not accepted as a child of a const impl
([#147130](https://github.com/rust-lang/rust/issues/147130), which oli-obk calls working as
intended), and the `[const]` condition on a specializing impl is never proven, so const evaluation
can reach a non-const function as a post-monomorphization error
([#148200](https://github.com/rust-lang/rust/issues/148200)).

```rust
impl<T: Hash> Digest for T {
    default fn digest(&self) -> u64 { /* runtime path */ }
}
impl<T: [const] Hash> const Digest for T {
    fn digest(&self) -> u64 { /* same answer, available during const eval */ }
}
```

**What it asks of the design:** whether the graph orders impls on anything beyond types, the same
question the const-value case raises, reached from the effect side rather than the value side.

## Impls over every enum variant, treated as an impl over the enum

Serves: the const-enum-dispatch use (DESIRED-USE-CASES.md).

With `adt_const_params` a const parameter can be an enum. Writing an impl for every variant still
does not make the generic case hold: the trait solver does not case-split an exhaustive
`ConstParamTy` enum, so `PropWrapper<N>: Prop` is unprovable even though `Number` has only the two
variants that are covered.

```rust
#[derive(PartialEq, Eq, ConstParamTy)]
enum Number { Int, Float }

struct PropWrapper<const N: Number>;
trait Prop { type Ty; }
impl Prop for PropWrapper<{ Number::Int }>   { type Ty = usize; }
impl Prop for PropWrapper<{ Number::Float }> { type Ty = f32; }

struct Foo<const N: Number> { n: <PropWrapper<N> as Prop>::Ty }  // E0277: PropWrapper<N>: Prop
```

The reporter's workaround is a `default type` blanket over `PropWrapper<N>`, which makes the bound
hold; specialization stands in for the missing exhaustiveness reasoning
([#130799](https://github.com/rust-lang/rust/issues/130799)).

**What it asks of the design:** nothing of specialization. The ask is that a set of impls covering
every variant of a closed const-generic enum count as an impl over the enum, i.e. exhaustiveness
case-split on `ConstParamTy` values. It lands in `adt_const_params` and the trait solver.

## An associated type that decides a field's type

Serves: the shrink-a-container's-layout use (DESIRED-USE-CASES.md).

`default type` changes a type in return position. The demand that never got written down as a use
case is for one in *field* position, chosen per instantiation, so that a container can shrink when
the element type permits:

```rust
struct RawVec<T> { ptr: Unique<T>, cap: <T as CapRepr>::Cap }

impl<T> CapRepr for T   { default type Cap = usize; }
impl<T: IsZst> CapRepr for T { type Cap = (); }        // a Vec<()> needs no capacity field
```

[#45431](https://github.com/rust-lang/rust/issues/45431) is the live version: `Vec<ZST>` is still 24
bytes because `RawVec` cannot drop its `cap` field without an associated-type-chosen field type, and
choosing the field type that way destroys covariance.

**What it asks of the design:** an honest limit on `default type`. Its shipped uses (`amadeus-parquet`,
`option_trait`) put the type in return or parameter position, where variance is not at stake. Layout
is the case that shows the position matters.

---

# Cross-cutting design properties

These are properties rather than extensions, and each cuts across the entries above.

**The branch has to work inside generic code.** The stable workarounds split on exactly this line:
autoref and `spez` resolve against a *concrete* expression type, so `spez`'s own documentation
concedes it is *"useless in generic functions, as Rust resolves the specialization based on the
bounds defined on the generic context, not … the actual type"*. The `TypeId`-comparing crates reach
further in: `try-specialize` bills itself as *"zero-cost specialization in generic context on stable
Rust"*, and `castaway` gets partway there with its `LifetimeFree` marker. That is where the
speed-motivated crates cluster, and any candidate feature is judged on which side of the line it
falls.

**Asking for a blanket impl to exist.** RFC 1210 notes that refactoring `Extend` to take the iterator
as a trait parameter costs the ability to say a type is extendable by an *arbitrary* iterator: the
trait system can require any number of specific impls, but not a blanket one, except for lifetimes
via higher-ranked bounds. Extending that to type parameters, as in
`where for<I: IntoIterator<Item = u8>> T: Extend<u8, I>`, was out of scope for the RFC and has had no
proposal since.

**Specialization that cannot be opted out of is contested.** This is a constraint on the design, not
a use. The behavior itself ships - `oasis-cbor` and `candid` encode `Vec<u8>` as a byte string rather
than an array of numbers - but when serde proposed to auto-optimize `Vec<u8>` "once specialization
lands in stable", users argued against it, because a silently-applied specialization removes the
author's ability to override, and cited Go's `[]int8`-vs-`[]uint8` JSON divergence as the failure
mode ([serde-rs/bytes#8](https://github.com/serde-rs/bytes/issues/8)). For correctness-bearing
behavior, then, a specialization that cannot be opted out of stops being a free optimization and
becomes a wire-format decision someone can no longer control. It bears on any decision to apply
specialization implicitly to a stable trait.

**Design affordances flagged without a use case attached.** Some of the design's authoritative
documents highlight a capability as important without naming what it is for; those are worth
recording so the property is not lost, even where the survey found no concrete demand for it. The
archetype is the RFC's *lattice rule*, presented as a future possibility with no use case shown; the
survey did find demand (Part II), so it is catalogued as an extension rather than left as a bare
affordance. Two others remain bare. *Always-applicable impls* (nikomatsakis's *maximally minimal
specialization*, rederived as a solver mode in the oli-obk branch) is a soundness affordance: it does
not add a use, it makes the existing uses sound, and is tracked in [ISSUES.md](ISSUES.md) and the
feature matrix rather than here. It is also a *negative space*: lcnr's 2026 *On always-applicable
trait impls* argues the rule should **not** be adopted at all - resolve specialization during
analysis via `maybe` bounds instead, which is sound without constraining which impls may exist. That
case, and its concrete cost (an impl using a type parameter twice becomes lifetime-dependent, e.g.
`Box<&u32>: PartialEq`; adding a type-parameter default would become a breaking change), is in
[ISSUES.md](ISSUES.md) root cause 1 and evaluated in [maybe-bounds.md](maybe-bounds.md). The RFC's
own rejected alternatives, *explicit priority ordering* and *singleton-non-default-wins*, are pure
ordering mechanisms; the sweeps turned up no demand for either, and they are listed only in the table
below so that the gap is on the record.

---

# Where RFC 1210's use cases live

Checked against the RFC section by section, so that nothing it motivates is missing from the doc set.

| RFC 1210 | Where it is covered |
|---|---|
| Motivation / Performance: `AddAssign` blanket vs. a `clone`-free override | USE-CASES.md, fast paths from known type identity |
| Motivation / Performance: `Extend` with a slice fast path; `TrustedSizeHint` marker | USE-CASES.md, a marker trait flags the fast path (`SpecExtend`/`TrustedLen`) |
| Motivation / Reuse: conditional default `add_assign` via `default impl` | USE-CASES.md, a base implementation overridden down a type hierarchy (`rucene`). The RFC's own example does not compile: ISSUES.md root cause 3 |
| Motivation / Reuse: `size_hint` refined from `ExactSizeIterator` | here, making a trait's own default body overridable |
| Motivation / Groundwork: efficient inheritance | USE-CASES.md, pseudo-inheritance |
| `default type` and the projection hazard | USE-CASES.md, in-place deserialization and a type-level `Option`; the hazard is ISSUES.md root causes 2 and 4 |
| Limitations: `AsRef<T> for T` plus lifting over `&T` | here, the lattice rule (#45742) |
| Limitations: `ByNeed` | here, same section |
| Limitations: `PartialOrd` from `Ord` plus lifting | here, same section |
| Interaction with lifetimes | ISSUES.md root cause 1 (region erasure); the demand side is here, type-level computation |
| Extension: inherent impls | here, specializing an inherent method without inventing a trait |
| Extension: `super` | here, reusing the body that was overridden |
| Extension: refining bounds on associated types | here, requiring a bound on a `default type` |
| Extension: extending HRTBs to type parameters | here, asking for a blanket impl to exist |
| Alternative: the lattice rule | here, the lattice rule |
| Alternative: negative bounds | here, negative reasoning |
| Alternative: explicit ordering, singleton-non-default-wins | not use cases; no demand found for either in the sweep |

The RFC's Motivation is fully represented by shipped code (USE-CASES.md). Everything it lists as a
*limitation* or a *possible extension* is still demand-only, ten years on, and every one of those has
independent demand recorded in the sweeps. That is a fair summary of the state of the feature.
