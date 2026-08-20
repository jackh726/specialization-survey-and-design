# Specialization - use cases that no shipped code expresses

This file is a complement to existing use case documentation. It collects the things people *want to
do* that reach for `specialization` (or something like it), and that nothing ships today - either
because the use sits outside the feature as designed, or for lack of will to build it.

It is written from the demand side: each entry says *what* someone wants to accomplish, *why* they
reach for specialization to do it, and - because this doc is specifically for uses that do *not* show
up in shipped code - *why no concrete shipped instance was found*: the feature cannot express it, the
type information it needs is already gone, the only real-world instances are workarounds, and so on.
That last part is also a check on the entry: a desired use with no account of its absence is a red
flag, because it is probably either actually shipped (and belongs in the use-case catalogue, not
here) or not really a use. Where a use would need some feature or extension that does not exist, that
is described *in general* - the shape of the missing capability - rather than as a specific design;
the design space is catalogued separately. Each entry ends with a **Needs:** line naming that
capability, so a use can be read straight through to the feature work it implies.

Code snippets are not expected to compile; they are simply examples written to show what the
requested behavior may look like. Each entry links its own evidence inline: a forum thread, a tracker
issue, or a shipped workaround.

**Note:** This document is written largely by an LLM with minor human editing and review.

---

## Auto-convert any backend error with `?`

**What:** in a driver or library, write `some_device.read()?` and have an arbitrary backend error
become the crate's own error type, without a hand-written `From` impl per backend.

**Why:** ergonomics. This is the single most-cited reason people reach for specialization.

A blanket `impl<E: Error> From<E> for MyError` collides with `core`'s reflexive `impl<T> From<T> for
T`: nothing tells the compiler that `E` is never `MyError`, so the two impls are seen as overlapping.

```rust
// the wish, in a driver crate:
impl<E: Error> From<E> for MyError /* where E is never MyError */ {
    fn from(e: E) -> MyError { MyError::Backend(e.to_string()) }
}
```

It recurs across [#46368](https://users.rust-lang.org/t/impl-t-from-t-for-myerror-conflict/46368),
[#124913](https://github.com/rust-lang/rust/issues/124913),
[#58904](https://github.com/rust-lang/rust/issues/58904), and the `Into`/`TryFrom` variants, and the
usual answer - "specialization will eventually fix this" - is wrong: the two impls partially overlap,
and specialization only orders impls where one is a strict subset of the other. What ships instead is
marker-trait forgery: `syllogism`'s `IsNot<T>`, `tea-codec`'s `Equality<X, Y>: NotEqual`, and
hand-written `False`-trait encodings that do not reliably work (one was reported as *"the types that
should be accepted are also being rejected"*,
[#24412](https://internals.rust-lang.org/t/negative-trait-bounds-using-feature-specialization/24412)).

**Needs:** negative reasoning - a way for an impl to apply only where a type does *not* implement
some trait, or is not some other type. That is a different feature from specialization, and because
the two impls only partially overlap, specialization's ordering would not resolve this even if it
shipped. The same want in argument position is an API that accepts only types *without* a
capability, e.g. `fn take<T: !Copy>(t: T)` to enforce a move-only invariant.

---

## Lift a conversion over a generic container

**What:** convert `C<T>` into `C<U>` by converting the contents, with one impl rather than one per
pair: `Option<T>` to `Option<U>`, `Vec<T>` to `Vec<U>`, `Wrapper<T>` to `Wrapper<U>`, so that
`.into()` works on the container the way it works on the element.

**Why:** ergonomics. It is asked of `std` for `Option` and `Vec`, and the same shape recurs in
every crate that has a generic wrapper.

```rust
impl<T, U> From<Option<T>> for Option<U> where T: Into<U> {
    fn from(o: Option<T>) -> Option<U> { o.map(Into::into) }
}
```

**Why it isn't found as a shipped use:** it is unutterable, and for two independent reasons that the
`Option` case happens to trip at once.

*The reflexive collision.* Where both sides share a type constructor, the impl covers the diagonal
`C<T>` to `C<T>`, which `core`'s `impl<T> From<T> for T` already covers, so E0119. Neither impl is a
subset of the other - the reflexive one also covers `u8` to `u8`, which the lift does not - so this
is partial overlap, and impl ordering has nothing to resolve. It bites `std` itself as much as
downstream crates: the ask for `impl<T, U: From<T>> From<Vec<T>> for Vec<U>` was answered *"this impl
would overlap and thus conflict with the reflexive `impl<T> From<T> for T`"*, and separately *"`From`
impls like this are widely desired ... but can't happen yet"*
([#19116](https://internals.rust-lang.org/t/impl-t-u-from-t-from-vec-t-for-vec-u/19116)). The
`Option` version got the same answer
([#19442](https://internals.rust-lang.org/t/blanket-implementation-for-impl-t-u-into-t-into-option-t-for-option-u/19442)),
and so did the user-wrapper versions: `From<Control<M>> for Control<N>` over a `Mode` parameter
([#74336](https://users.rust-lang.org/t/cannot-impl-t-u-from-type-t-for-type-u/74336)),
`From<Vec3<B>> for Vec3<A> where A: From<B>`
([#42861](https://github.com/rust-lang/rust/issues/42861)), and `From<Wrapper<T>> for Wrapper<U>`,
which is one of the two motivating examples for a proposed `T != U` bound
([#22881](https://internals.rust-lang.org/t/t1-t2-type-non-equality-bound/22881)).

*The orphan rule.* Where the container is foreign, the impl is not the crate's to write at all,
whatever happens to the overlap: `impl<T, U> From<Option<T>> for Option<U>` has no local type ahead
of its uncovered parameters, so E0117. The same wall stands in the other direction, converting *into*
a foreign type: `impl<T: BasicDatum> TryFrom<T> for i32` draws E0119 against `core`'s
`impl<T, U: Into<T>> TryFrom<U> for T` *and* E0210 for the orphan violation, and the thread ends with the author giving up
on the conversion traits entirely - *"I think I will simply not use `From`/`Into` and
`TryFrom`/`TryInto`"*
([#67824](https://users.rust-lang.org/t/generic-impl-tryfrom-impossible-due-to-orphan-rules-what-should-i-do/67824)).

Two boundaries are worth stating precisely, because the wall is narrower than it is usually
described. `impl<T> From<T> for MyWrapper<T>` is *accepted*: `T` can never equal `MyWrapper<T>`, and
coherence knows it. So is a lift between *different* constructors where one is local, such as
`impl<T, U: From<T>> From<Vec<T>> for MyVec<U>`. What is unutterable is specifically the
same-constructor lift, plus anything the orphan rule puts out of reach.

There is also a standing objection to fixing this with specialization even if it shipped, from the
same thread: *"specialization is used in the standard library only for performance optimizations,
not to write publicly visible trait implementations that are impossible without specialization"* - a
publicly observable impl that exists only by specialization is a much larger commitment than an
optimization that could be withdrawn.

**Needs:** for the reflexive collision, a rule for partial overlap together with an intersection impl
for the diagonal - `impl<T> From<C<T>> for C<T>`, which is a strict subset of the reflexive impl and
so is orderable once the partial overlap itself is admitted. Negative reasoning is the alternative
route people actually ask for, as a `where T != U` bound. For foreign containers, neither helps: that
half is a question about the scope of the orphan rule, and no specialization design touches it.

---

## Recover a value's `Debug` after its type has been erased

**What:** format an already-*erased* value with `{:?}` when its real type implements `Debug`, and fall
back to a placeholder otherwise - the canonical case being a panic payload (`Box<dyn Any>`) from
`catch_unwind`.

**Why:** diagnostics.

**Why it isn't found as a shipped use:** the *non*-erased version of this - "print `T`'s `Debug` if
`T: Debug`, else the type name" - is found everywhere, as the `debugit`/`spez` idiom (~16 crates),
because there the value's type is still a generic parameter to branch on. That is a shipped use and
lives in the use-case catalogue. The *erased* version is absent because once the payload is
`Box<dyn Any>` its type is gone: there is no type parameter and no impl to specialize, so
specialization cannot express it at all. What ships instead is runtime downcasting (`downcast-rs`,
`trait-cast`), which only works against a target enumerated up front, not "any `Debug` type".

```rust
match catch_unwind(handler) {
    Err(payload) => match debug_of(&payload) {   // payload: Box<dyn Any + Send>
        Some(d) => error!("handler panicked: {d:?}"),
        None    => error!("handler panicked: <opaque payload>"),
    },
    Ok(resp) => resp,
}
```

Asked since 2016 - *"Debug of Any could delegate Debug impl to the real object if object actually
implements Debug"* ([#4029](https://internals.rust-lang.org/t/specialization-for-better-debug-for-any/4029)).
**Needs:** reflection on an already-erased value - "does the real type behind this `dyn Any`
implement `Debug`?" - which is not a specialization question at all, because there is no impl left to
specialize. The nightly `try_as_dyn` probe is its natural home but does not yet answer for `dyn` self
types ([#144361](https://github.com/rust-lang/rust/issues/144361)).

---

## One `unwrap_or` that takes a value or a lazy closure

**What:** a single method that accepts either a ready default value or a closure that computes one,
instead of the `unwrap_or` / `unwrap_or_else` pair.

**Why:** API ergonomics - fewer near-duplicate methods.

```rust
trait ByNeed<T> { fn compute(self) -> T; }
impl<T> ByNeed<T> for T { fn compute(self) -> T { self } }
impl<F, T> ByNeed<T> for F where F: FnOnce() -> T { fn compute(self) -> T { self() } }

impl<T> Option<T> {
    fn unwrap_or<U: ByNeed<T>>(self, def: U) -> T { /* one method, not two */ }
}
```

The two impls overlap at `F: FnOnce() -> F`, in both directions - neither is a subset of the other,
so specialization's ordering has nothing to resolve. This is RFC 1210's own `ByNeed` limitation, and
the same shape blocks `AsRef<T> for T` alongside the lifting impl over `&T`, and `PartialOrd for T
where T: Ord` alongside the reference-lifting impls (the live `std` case is
[#45742](https://github.com/rust-lang/rust/issues/45742)).

**Needs:** a rule for *partially* overlapping impls: accept both, given an impl covering the region
where they overlap. That is strictly more than specialization's strict-subset ordering allows, and
the RFC that proposes the ordering names it only as a possible later addition.

---

## A recursive algorithm with hand-written small-size base cases

**What:** a generic algorithm over `[T; N]` / `Matrix<T, N>` that uses a closed-form implementation
for small sizes and a recursive one for larger sizes - e.g. a determinant with 1x1/2x2/3x3 base cases
and a minor-expansion recursion above.

**Why:** performance and correctness (the base cases are both faster and where the recursion bottoms
out).

```rust
impl<T, const N: usize> Determinant for Matrix<T, N> {
    default fn det(&self) -> T { /* recurse: split into (N-1)-minors */ }
}
impl<T, const N: usize> Determinant for Matrix<T, N> where N <= 3 {
    fn det(&self) -> T { /* closed form */ }
}
```

That is a users.rust-lang thread nearly verbatim - *"For bigger matrices, I want to (using
`specialization`) use a recursive algorithm that splits up the matrix into smaller matrices"*
([#106819](https://users.rust-lang.org/t/recursive-const-generics-and-specialization/106819)). Today
this only works when the base case is a *concrete* size named as its own type (`Const<0>`, `[T; 1]`),
because there is no way to make impls overlap on a *predicate over a const value* (`N <= 3`).

**Needs:** impl ordering that can consult a const value, not only type structure. Everything else
here is ordinary specialization; the single missing piece is that `N <= 3` cannot make one impl more
specific than another.

---

## Core trait impls for fixed-size arrays, including the empty array

**What:** `Default`, `Serialize`/`Deserialize`, and similar for `[T; N]` across *all* `N`, including
`[T; 0]`.

**Why:** low-level binary protocols use fixed-size arrays directly, and the gaps force wrapper types.

*"Many core traits (`Default`, `Distribution<[_;_]>`, `Serialize`/`Deserialize`) … are not
implemented for arrays"*
([#18212](https://internals.rust-lang.org/t/proposal-const-specialization/18212)), because `[T; 0]`
has an unconditional impl that overlaps a bounded `[T; N]` impl without either being a subset. The
same shape is [#94313](https://github.com/rust-lang/rust/issues/94313): `[T; 0]` is not `Copy`/`Clone`
for non-`Copy` `T`.

**Needs:** both of the preceding capabilities at once - ordering by a const value, *and* a rule for
overlaps that are not strict subsets. This is the partial-overlap problem wearing a const hat.

---

## A container indexed by the type of its elements

**What:** a heterogeneous, type-level set or map, where a value is stored and later fetched *by its
type* - the primitive under HList/frunk-style records and ECS resource maps.

**Why:** richer libraries - the type-level machinery is the product.

```rust
trait Set { fn get<T>(&self) -> Option<&T>; }
impl Set for () { fn get<T>(&self) -> Option<&T> { None } }
impl<S: Set, H> Set for (S, H) {
    fn get<T>(&self) -> Option<&T> { /* return the H, if the query type is H; else recurse into S */ }
}
```

Shipped type-level libraries get partway with associated-type overrides (a generalized `Option` at
the type level), but the "fetch by type" step needs the impl to branch on whether a *method's* type
parameter equals the element type - which no impl-ordering rule expresses. The thread that asks for it
(*"I want to do some sort of type level Set"*,
[#23271](https://internals.rust-lang.org/t/yet-another-stupid-thought-about-specialization/23271))
even wants `&'static str` distinguished from `&'a str`, which runs straight into the compiler's
inability to dispatch on lifetimes.

**Needs:** a decision procedure over type identity, evaluated somewhere impl selection cannot reach,
plus the ability to branch on a lifetime the compiler deliberately erases. Adjacent to
specialization, but not the same feature.

---

## Cleanup that depends on a type bound, with `needs_drop` agreeing

**What:** a guard type whose `Drop` runs extra cleanup only when a type parameter has some capability,
and whose `mem::needs_drop` reports `false` when it does not - so a `Guard<u8>` carries no drop glue.

**Why:** correctness (the cleanup is bound-dependent) without paying for drop glue on the types that
do not need it.

```rust
impl<T: Handle> Drop for Guard<T> {
    fn drop(&mut self) { self.release() }
}
const _: () = assert!(!mem::needs_drop::<Guard<u8>>());   // u8: !Handle, so no glue at all
```

A `Drop` impl cannot carry bounds (E0367), so shipped code (`linux-support`) routes `Drop` through a
helper trait - but that workaround cannot change the compile-time `needs_drop` answer. Asked as
*"it would be nice to allow `Drop` to be implemented for specialized types"* with `needs_drop`
reporting `false` for the no-drop case
([#12873](https://internals.rust-lang.org/t/can-we-fix-drop-to-allow-specialization/12873)).

**Needs:** bounds on `Drop` (or an overridable destructor) *and* a `needs_drop` that follows them.
Not specialization: the blocker is that `needs_drop` feeds other reasoning and so cannot be allowed
to vary with generics ([#46893](https://github.com/rust-lang/rust/issues/46893)). A helper trait
fakes the first half and can never fake the second.

---

## Shrink a container's layout when the element type allows it

**What:** a container that drops a field, and shrinks, for element types that do not need it - the
canonical case being `Vec<ZST>`, which needs no capacity field.

**Why:** performance and memory layout.

```rust
struct RawVec<T> { ptr: Unique<T>, cap: <T as CapRepr>::Cap }
impl<T> CapRepr for T        { type Cap = usize; }
impl<T: IsZst> CapRepr for T { type Cap = (); }        // a Vec<()> needs no capacity field
```

[#45431](https://github.com/rust-lang/rust/issues/45431): `Vec<ZST>` is still 24 bytes because
`RawVec` cannot pick its capacity field's *type* per element type. This is the one place a
type-varying associated field would help and also where it is hardest: choosing a field's type this
way destroys the container's variance.

**Needs:** an associated type chosen per instantiation and usable in *field* position. That is the
sharp edge of the same per-impl associated-type machinery that already works in return position, and
the position is what makes it hard: a field's type is what fixes the container's variance.

---

## Reach the standard library's private speed fast paths for your own types

**What:** get, for a user-defined container or reader/writer, the same specialization-driven fast
paths `std` uses internally - bulk `memcpy` on `Copy` slices, exact-size reservation on `TrustedLen`
iterators, the `sendfile`/`copy_file_range` fd-to-fd copy path.

**Why:** performance. Container and codec authors want parity with `std` without re-vendoring it.

*"Given a pair of AsyncRead/AsyncWrite implementations, if both are file descriptors there are ways to
speed up copying between them"*, with the meta-goal of *"help[ing] user-defined container and iterator
types to enjoy the kind of optimizations that `std` containers and iterators have been getting for a
long time"* ([#20836](https://internals.rust-lang.org/t/unsafe-specialization/20836)). Today `std`
keeps these paths behind private, unimplementable marker traits, and the ecosystem's answer is to
vendor the container (16 such crates in the survey).

**Needs:** two things. That a specialized fast path be reachable by downstream code at all, which
would take specializing an inherent method without inventing a trait for it, or overriding a blanket
impl belonging to an upstream crate. And a deliberate decision about whether `std` exposes its
markers, which is entangled with the soundness of specializing on a safe trait, not only with
stabilization timing.

---

## A precise `size_hint` / `ExactSizeIterator` when the backend length is known

**What:** an iterator/decoder whose backing store has a known bounded length wants to report a precise
`size_hint` (or implement `ExactSizeIterator`) by overriding the default, instead of the useless
`(0, None)`.

**Why:** performance (callers can pre-allocate).

`constriction`'s source says exactly this - *"TODO: override `size_hint` … when specialization is
stable"* and *"return `impl ExactSizeIterator` … once specialization is stable"* - for decoders whose
backend is `BoundedReadWords`. Shipped code works around it with a parallel `SizeHint` trait
(`portable-io`).

**Needs:** a trait's *provided* default body to be overridable for some types, which today forces
either copying the body into an impl or inventing a parallel trait. Where the method is inherent
rather than a trait method, it also needs specializing an inherent method without inventing a trait
to hang it on.

---

## In-place construction for FFI types that cannot be moved

**What:** a uniform way to construct any type *in place* at a destination pointer - most values by
writing themselves, some sources (a constructor closure, an FFI `RvalueReference`) by running a
constructor at the destination without ever forming a value to move from. This is Crubit's `Ctor`,
one of the 2026 project-goal examples.

**Why:** C++ interop *correctness*, not just performance. Many C++ types must be constructed in place
and cannot be moved after construction, so a plain by-value construction would be unsound.

```rust
pub unsafe trait Ctor { type Output; unsafe fn ctor(self, dest: *mut Self::Output); }
impl<T> Ctor for T {                            // any value constructs itself in place
    /* Output = T */ unsafe fn ctor(self, dest: *mut T) { dest.write(self); }
}
impl<O, F: FnOnce(*mut O)> Ctor for FnCtor<O, F> {   // a constructor closure runs at the destination
    /* Output = O */ unsafe fn ctor(self, dest: *mut O) { (self.0)(dest); }
}
```

Unlike the rest of this file, ordinary specialization *could* express this directly. It is here
because the prominent real-world instance does not use it: every version of Crubit's `support/ctor.rs`
from 2023 to today implements `Ctor` with a `SelfCtor` auto trait and `impl !SelfCtor for …` opt-outs
- negative reasoning faking the overlap - and gates `negative_impls`/`auto_traits`, never
`feature(specialization)`. Crubit's own comment: *"`SelfCtor` is a workaround to implement
specialization of `Ctor`, and will go away if we ever get a useful form of specialization."* So the
want is real and shipped; what is missing is a stable, sound form of the specialization it would
otherwise use.

**Needs:** no new capability - specialization as designed, stabilized on a footing sound enough that
a correctness-bearing public API can rest on it. This entry is here because the demand is real and
the shipped code fakes it, not because the feature as designed would fall short.
