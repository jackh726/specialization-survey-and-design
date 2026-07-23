# Specialization in Rust - actual and desired use cases

Typically in Rust, you cannot write two impls that overlap (i.e. where, for some type, either
impl would be an equally "correct" choice) for a given trait. The `specialization` and
`min_specialization` features carve out one principled exception: overlap is allowed when one
impl matches a strict *subset* of the types the other matches. The more specific impl then wins,
so there is a definite answer instead of an ambiguous one.

Progress on both features has stalled for many years, in large part due to known and theoretical
soundness issues. Even so, specialization is widely considered one of Rust's most desirable
unstable features, because of how much it can express.

This document catalogs *why* people reach for specialization, across the
standard library, ecosystem crates on crates.io and github, and other theoretical
use cases.

It is a descriptive catalogue: it records what the uses are and why, not how they could be served
without specialization. All code shown is distilled from real sources.

Each section below covers one distinct use, with a real example and a note on why that use is worth
calling out separately.

## Methodology

Uses were collected from four places.

**The standard library.** Every specialized trait in `std` was inventoried by hand from source at a
pinned revision, with per-trait line cites.

**crates.io, exhaustively.** Every published crate was downloaded, streamed, and scanned for the
feature gates and for `default fn` / `default impl` / `default type`. 2324 crates matched some
marker; 1459 of those have no feature gate at all, and since `default fn` is gated syntax that
cannot compile without one, those matches are comments, string literals and identifiers — noise.
That leaves **865 crates that really enable specialization**, which split in two:

- **350 author specialization themselves** — they write the `default` impl. These are the source of
  the use cases here.
- **515 enable the feature but write no `default`** — they enable it only so that coherence will
  accept an impl of theirs that overlaps *someone else's* blanket default. Roughly 248 of these are
  downstream of one trait, Solana's `AbiExample`.

For each of the 350, tooling extracted the specialized trait declaration, its full impl family, and
the blanket/override pair, all sliced from real source. A stratified sample of **159 of them was
classified by hand**; the 142 that turned out to carry a real use are the rows of
[USE-CASE-INDEX.md](USE-CASE-INDEX.md). The 26 auto-published `rustc-ap-*` / `ra-ap-*` compiler
crates were excluded as not ecosystem usage.

**GitHub.** The same markers were searched across GitHub and cross-referenced against the crates.io
repositories, yielding 803 matching repositories with no crates.io publication. These turned out to
be dominated by `rust`/`std` forks, committed vendor directories, forks of published frameworks,
and blockchain applications inheriting `frozen-abi`. They reinforced the categories below rather
than adding to them.

**Stated demand.** Shipped code understates the demand, because the uses specialization cannot yet
express leave no code behind. Three further sweeps covered that: forum and tracker threads asking
for specialization; crates that fake it on stable (autoref/`spez` tricks, `TypeId` + `transmute`,
negative-reasoning marker traits); and crates that wrote the specialization but keep it behind a
nightly cfg or a comment saying "delete this when specialization lands". The four use cases in the
2026 project-goals document are included here as well.

A few caveats. The scan and extraction are regex-driven, good for bucketing but not a parser. Crate
counts over-weight frameworks — one extension point accounts for ~248 of the 865. And the
hand-classified sample is ~48% of authors, so treat proportions as shape, not census.


***NOTE:** Initial classifications were done by an LLM, and several rounds of refinement resulted in the current classification criteria. Use cases were selected through a mix of manually identifying key known examples, as well as picking representative use cases for different combinations of classification criteria (mixed between LLM- and human-chosen examples). Final classifications for examples listed were human-decided. Commentary on each example is large human-written, but some LLM-written with human-revision. Background, methodology, and overview sections were largely LLM-written with human editing and review.

## Categorization Overview


There are generally two axes to categorize specialization uses on: what the motivations of the use of specialization are and what the impls overlap on. There are also a number of other interesting properties to vary for some subset of specialization use-cases that are worth calling out.

### Core axes

**Motivation**

| motivation | why |
|---|---|
| **performance** | the generic path is correct but can be faster for some types |
| **opt-in behavior** | different behavior is desired for some types, but it is not *required* |
| **correctness** | the correct answer genuinely differs per type |
| **pseudo-inheritance** | a base impl with default behavior, overridden down a type hierarchy |
| **workaround** | faking a feature the language lacks |
| **richer-libs** | type-level machinery as the library's actual product |

**What overlaps**

| overlap | meaning |
|---|---|
| **trait** | a trait the type *has* |
| **concrete type/const** | the type's exact identity, or a const value |

### Compile-time observed specialization with `default type`

Some uses of the full `specialization` feature require changing an associated type with `default type`. This results in differences observable *at compile-time*, and is somewhat independent from the axes above.

### Non-locality of runtime-observed specialization

For most uses of specialization, those not observable at compile-time, there is a direct *runtime* effect. There is an additional relevant property independent from the above:

- **Local or non-local.** If specialization does not apply uniformly, can that result in *incorrect* behavior? This can happen if a type is created or modified into a specialization-dependent state that is later relied upon.

(Given that specialization is almost always local-only, this will be pointed out in the places that deviate.)

### API-transparency

Some APIs, including the standard library, choose to *only* expose specialization through marker traits that cannot be implemented publicly, in order to either ensure correctness or guarantee no reliance on specialization.

### Pseudo-reflection

There are a few uses of specialization that stand in for a **reflection**-like feature, asking only whether two types are *equal* rather than what a type *is*. The answer is a fact about type identity, recovered through overlap resolution.

---

## Fast paths from known type identity

Motivation: performance
Overlap: concrete types

This pattern shows up anytime that you have a known *specific* type that can be handled in a more performant way. Some uses are explicitly made to be *private*, but the same pattern can extend to be a *public* interface, too.

`ahash` is an example of an ecosystem crate that does hashing slightly differently from `core`. Specialization is used to bypass the more expensive hashing through a `Hasher` for types of size less than `u64`.

```rust
pub(crate) trait CallHasher {
    fn get_hash(...) -> u64;
}
impl<T: Hash + ?Sized> CallHasher for T {
    default fn get_hash(...) -> u64 { /* go through Hasher */ }
}
impl CallHasher for u64 {
    fn get_hash(value: &..., rs: &RandomState) -> u64 { rs.hash_as_u64(value) }  // overlaps T = u64
}
```

Another example that could be imagined is using specialization to guarantee SIMD operations for types with appropriate instructions:

```rust
trait Dot { fn dot(&self, rhs: &Self) -> f32; }
impl<T: Into<f32> + Copy> Dot for [T] {
    default fn dot(&self, rhs: &[T]) -> f32 { /* scalar: sum a*b, one lane at a time */ }
}
impl Dot for [f32] {
    fn dot(&self, rhs: &[f32]) -> f32 { /* f32x8 fused-multiply-add over chunks, then reduce */ }
}
```

The `acgmath` crate actually supports SIMD for *custom* types through specialization.

**Why it's here:** This is the plainest case, and the one people reach for first: one algorithm, written generically, with a hand-tuned path for the types that have one. Usually, calling the "wrong" impl is mild: it's only a performance loss. However, that exact fact makes it hard to know that the specialized impl is used and the mirrored behavior would need to be maintained over time.

---

## A marker trait flags the fast path

Motivation: performance
Overlap: trait

Aside from specialization on *specific types*, it's also very common to use specialization for performance over a marker trait.

This pattern is used often in the standard library; it's common to have an unsafe `Trusted*` trait, which is implemented for known types and is used to bypass extra checks that would otherwise be required for soundness in the face of bad safe impls. For example, `Vec::extend` can use `TrustedLen` specialization to bypass per-element memory reservation. These marker traits are not public.

```rust
trait SpecExtend<T, I> {
    fn spec_extend(&mut self, iter: I);
}
impl<T, I: Iterator<Item = T>> SpecExtend<T, I> for Vec<T> {
    default fn spec_extend(&mut self, iter: I) { /* push one at a time */ }
}
impl<T, I: TrustedLen<Item = T>> SpecExtend<T, I> for Vec<T> {
    fn spec_extend(&mut self, iter: I) { /* reserve(upper) up front, then write each */ }
}
```

Similar specialization is done for `Copy` (actually `TrivialClone`, which is implemented for a known set) to bypass individual-element cloning in favor of mem-copying a slice.

This latter case also shows up similarly in the ecosystem quite often. `cpython` (and previously `pyo3`) use specialization in a similar manner, creating a `Vec` from a *buffer* more performantly than from a general *sequence*.

**Why it's here:** This is the next plainest case: a fast path keyed not on type *identity*, but on type *capability*.

---

## Fast paths from either a trait implementation or on concrete types

Motivation: performance
Overlap: trait or concrete types

Of course, there are *also* cases where specialization happens on *both* concrete types and marker traits.

In the standard library `vec![T; n]` can be optimized in three different ways:
- When `T` is a single byte, then the entire allocation can be set to that byte value
- When `T` is `()`, the length of the newly created `Vec` can just be set to `n`
- When `T` has a value of `0`, then the entire allocation can be zero-allocated without additional work

```rust
trait SpecFromElem: Sized {
    fn from_elem<A: Allocator>(elem: Self, n: usize, alloc: A) -> Vec<Self, A>;
}
impl<T: Clone> SpecFromElem for T {
    default fn from_elem<A: Allocator>(elem: T, n: usize, alloc: A) -> Vec<T, A> {
        /* clone `elem` into each of the n slots */
    }
}
impl<T: Clone + IsZero> SpecFromElem for T {
    default fn from_elem<A: Allocator>(elem: T, n: usize, alloc: A) -> Vec<T, A> {
        if elem.is_zero() { /* one alloc_zeroed */ }
        else { /* build through iteration */ }
    }
}
impl SpecFromElem for u8 {
    fn from_elem<A: Allocator>(elem: u8, n: usize, alloc: A) -> Vec<u8, A> {
        if elem == 0 { /* one alloc_zeroed (calloc) */ }
        else { /* one write_bytes (memset) */ }
    }
}
```

Surprisingly, this doesn't come up much otherwise.

**Why it's here:** This is the simplest case that mixes specialization on both type identity and capability; and so, any alternative features that target *only one* of those would need to be *mixed*, whereas specialization/min_specialization handle this "trivially".

---

## StepBy: soundness from consistent specialization

Motivation: performance
Overlap: concrete type

In the standard library, `Iterator::step_by` is optimized for `StepBy<Range<{integer}>>`. However, to do this, the `StepBy` struct is set up in a specialized way that is relied upon in the specialized impl used for iteration. Thus, if the *setup* is called without specialization but the *iteration* is (or vice versa), this could lead to either unsoundness or incorrectness.

Most uses of specialization do not rely on specialization holding *consistently* for correctness or soundness (though, that's not the same as requiring that specialization holds *always*). `StepBy` is one of the few cases where the type's validity requires the consistent application of specialization.

```rust
struct StepBy<I> {
    iter: I,
    ...,
}
impl<I> StepBy<I> {
    fn new(iter: I, step: usize) -> StepBy<I> {
        let iter = <I as SpecRangeSetup<I>>::setup(iter, step);
        StepBy { iter, ... }
    }
}
trait SpecRangeSetup<T> {
    fn setup(inner: T, step: usize) -> T;
}

impl<T> SpecRangeSetup<T> for T {
    #[inline]
    default fn setup(inner: T, _step: usize) -> T {
        inner
    }
}
impl SpecRangeSetup<Range<$t>> for Range<$t> {
    fn setup(mut r: Range<$t>, step: usize) -> Range<$t> {
        ...
    }
}

unsafe trait StepByImpl<I> {
    type Item;
    fn spec_next(&mut self) -> Option<Self::Item>;
    ...
}
unsafe impl<I: Iterator> StepByImpl<I> for StepBy<I> {
    type Item = I::Item;
    default fn spec_next(&mut self) -> Option<I::Item> { ... }
}
unsafe impl StepByImpl<Range<$t>> for StepBy<Range<$t>> { ... }
```

**Why it's here:** This pattern is the exception to the rule that specialization is a local decision. Importantly, it means that we need to consider if specialization is applied *consistently*, as opposed to "just" yes or no.

---

## Non-correctness behavioral opt-in

Motivation: opt-in behavior
Overlap: trait

There are many cases where crates want to opt-in to behavioral differences that are ultimately not related to correctness.

A very common pattern of this is having some type that is generic over a different type within, and wanting to implement `Debug` for that type in such a way that *if the inner type is also `Debug`*, more information is printed.

The `DebugIt` type from the `debugit` crate accepts any `T`, printing its value when `T: Debug` and its type name otherwise:

```rust
impl<T> fmt::Debug for DebugIt<T> {
    default fn fmt(&self, f: &mut Formatter) -> fmt::Result { write!(f, "{}", type_name::<T>()) }
}
impl<T: fmt::Debug> fmt::Debug for DebugIt<T> {
    fn fmt(&self, f: &mut Formatter) -> fmt::Result { self.value.fmt(f) }
}
```

This pattern comes up extremely often, sometimes applying to `Display` or otherwise. In the standard library, it's very rarely used, with one exception for panic message printing of `BTreeMap`/`BTreeSet`.

**Why it's here:** This is the first case where the specialized impl produces an observably *different* result, not just a faster one. The difference is only cosmetic, though, so it still doesn't matter whether specialization applies consistently. It is worth noting mostly because it is, by count, the most common use in the whole ecosystem.

---

## Injecting custom behavior into error propagation

Motivation: opt-in or correctness
Overlap: concrete types

The `?` operator is implemented using the `FromResidual` trait. For `Result<T, E>`, the standard library implements `FromResidual<Result<!, F>>` for all `F: From<E>`. Some users desire to inject custom logic into the result propagation for some custom errors, usually capturing source or backtrace information.

The `msica` crate uses a custom Result-like type `CustomActionResult` and specializes on the propagation for their `Error` type:

```rust
impl<E: std::error::Error> FromResidual<Result<Infallible, E>> for CustomActionResult {
    default fn from_residual(_: Result<Infallible, E>) -> Self {
        CustomActionResult::Failure
    }
}
impl FromResidual<Result<Infallible, Error>> for CustomActionResult {
    fn from_residual(residual: Result<Infallible, Error>) -> Self {
        match residual.unwrap_err().kind() {
            ErrorKind::ErrorCode(code) => CustomActionResult::from(code.get()),
            _ => CustomActionResult::Failure,
        }
    }
}
```

This is also desired for porting C++ libraries, like [abseil](https://abseil.io/docs/cpp/guides/status).

Note that `msica` can only do this because both the trait impls and the target type are its own. To
get the same behavior on the standard library's `Result` (which is what people usually want, and
what a `StatusOr`-style port would need), `std` would have to mark the method in its own blanket
impl as `default fn`:

```rust
impl<T, E, F: From<E>> ops::FromResidual<Result<convert::Infallible, E>> for Result<T, F> {
    default fn from_residual(residual: Result<convert::Infallible, E>) -> Self { ... }
}
```

Nothing downstream can opt into this; it is a decision `std` makes per method, and a semver-visible
one, since marking a method `default` is a promise that it can be overridden.

**Why it's here:** This is the first case where the crate that wants the override does not own the blanket impl: `std` does. So the interesting question is less about what specialization can express than about whether `std` is willing to mark its own methods `default fn`: a per-method, semver-visible choice, distinct from turning specialization on at all.

---

## Type-specific serialization behavior

Motivation: correctness
Overlap: concrete types

In some crates, serialization or deserialization (of some form) depends on some *inner type* being serialized or deserialized.

In the `oasis-cbor` crate, a `Vec<u8>` is deserialized as a `ByteString`, but a generic `Vec<T>` is deserialized as an `Array`:

```rust
impl<T: Decode> Decode for Vec<T> {
    default fn try_from_cbor_value(v: Value) -> Result<Self, DecodeError> {
        match value {
            Value::Array(v) => v.into_iter().map(T::try_from_cbor_value).collect(),
            _ => Err(DecodeError::UnexpectedType),
        }
    }
}
impl Decode for Vec<u8> {
    fn try_from_cbor_value(v: Value) -> Result<Self, DecodeError> { 
        match value {
            Value::ByteString(v) => Ok(v),
            _ => Err(DecodeError::UnexpectedType),
        }
    }
}
```

This is also a pretty common pattern, found in codecs, serializers, numeric libraries, and language bindings.

**Why it's here:** This is where the observably different result becomes *correctness*-relevant: the byte-string encoding is the right one for `Vec<u8>`, so a missed specialization is a bug rather than a slowdown. Unlike `StepBy`, though, the decision is still *local*: each value is encoded on its own, so the only thing that matters is whether specialization applies, not whether it applies consistently.

---

## In-place deserialization

Motivation: performance & correctness
Overlap: concrete types

Sometimes, crates want to fallback to the *safe* path for correctness, but allow a faster path in the face of problems. In the case of `amadeus-parquet`, types are transparently boxed to prevent overflow. And parsed arrays are deserialized directly into a `Box` to prevent large values from being placed on the stack. This is also decided at *compile-time* using `default type` specialization.

```rust
impl<T: ParquetData> ParquetData for Box<T> {
    default type Schema = BoxSchema<T::Schema>;
    default type Reader = BoxReader<T::Reader>;
}
impl<const N: usize> ParquetData for Box<[u8; N]> {
    type Schema = FixedByteArraySchema<[u8; N]>;
    type Reader = BoxFixedLenByteArrayReader<[u8; N]>;
}
```

**Why it's here:** This is the first case where the override changes a *type* rather than a value. That makes it observable at compile time, so any alternative feature that reasons only about runtime behavior cannot cover it.

---

## Using specialization to print type names

Motivation: workaround
Overlap: concrete types

The `solana-frozen-abi` crate has the ability to generate tests that ensure an ABI does not change, but it needs to know the type names of a type and its fields. It does this by having an `AbiExample` trait that is required to be implemented by any type *contained* in the "object graph" that generates an "example" type.

```rust
impl<T> AbiExample for T {                                  // every type, by default
    default fn example() -> Self { panic!("derive AbiExample for {}", type_name::<T>()) }
}
impl<T: AbiExample> AbiExample for Option<T> {
    fn example() -> Self {
        println!("AbiExample for (Option<T>): {}", type_name::<Self>());
        Some(T::example())
    }
}
#[derive(AbiExample)]
struct Foo;
```

Two things to note about this crate:

First, this seems like it is a substitute for reflection of some kind.

Second, this crate had the largest footprint in the survey: roughly 248 Solana consumers and
chain forks enable specialization solely to host a derived override.

**Why it's here:** This is the most widely-deployed use found, but is basically just a workaround for being able to query the recursive layout of a type.

---

## Bounds on Drop

Motivation: correctness, workaround
Overlap: capability

You cannot bound a `Drop` impl (E0367), so `linux-support` routes `Drop` through a specializable
trait to get bound-dependent cleanup:

```rust
impl<R, F> SpecDrop for Socket<R, F> { default fn spec_drop(&mut self) { self.manually_drop() } }
impl<R: Receives, F, P> SpecDrop for Socket<R, F> {
    fn spec_drop(&mut self) { self.remove_receive_queue(); self.manually_drop() }
}
impl<R, F> Drop for Socket<R, F> { fn drop(&mut self) { self.spec_drop() } }
```

A few other crates do similar things. There is orthogonal work to have better control over Drop semantics, so this is mostly just a workaround.

**Why it's here:** This is a similar pattern to much of std, where the specialization is not on the (`Drop`) impl itself, but rather on a specializable trait. While the behavior is correctness-based, the pattern itself is a workaround for not being able to specialize on `Drop`.

---

## A type-level Option

Motivation: richer-libs
Overlap: concrete types

Here, specialization is used to create a library that generalizes `Option` into a trait.

```rust
impl<T, Lhs: PureMaybe<T>> MaybeFilter<T> for Lhs { default type Output = Option<T>; /* .. */ }
impl<T> MaybeFilter<T> for () where T: NotVoid { type Output = (); /* .. */ }
impl<T> MaybeFilter<T> for T { type Output = Option<T>; /* .. */ }
```

There are also a small number of other crates that use specialization for type-level richness.

**Why it's here:** This pattern shows that associated type specialization can result in richer libraries, resulting in cleaner code without potential extra runtime overhead.

---

## Type equality as the answer

Motivation: richer-libs
Overlap: concrete types

A reflection-like use: `scrapmetal`'s `Cast<T>` reports whether `U` is the same type as `T`:

```rust
impl<T, U> Cast<T> for U { default fn cast(self) -> Result<T, Self> { Err(self) } }
impl<T> Cast<T> for T   { fn cast(self) -> Result<T, Self> { Ok(self) } }
```

`specialize` exposes the same as `obj.is::<T>()`; `metatype` recovers a type's kind. On stable people
fake it with `TypeId` and `castaway`.

**Why it's here:** This case asks only whether two types are *equal* and returns that fact as its answer. This is essentially reflection.

---

## Extendable in-place initialization

Motivation: richer-libs & performance
Overlap: concrete types

Crubit's `Ctor` trait has similar aspirations. Certain types want to be able to implement in-place initialization in different manners. Any type can be constructed to a destination itself; but, some types like `fn(*mut T)` can construct a value of `T`:

```rust
pub unsafe trait Ctor: Sized {
    type Output: ?Sized;
    unsafe fn ctor(self, dest: *mut Self::Output);
}
impl<T> Ctor for T {
    type Output = T;
    default unsafe fn ctor(self, dest: *mut Self::Output) {
        dest.write(self);
    }
}
impl<T> Ctor for fn(*mut T) {
    type Output = T;
    unsafe fn ctor(self, dest: *mut Self::Output) {
        (self)(dest)
    }
}
```

**Why it's here:** This is a similar pattern that uses richer library features to specifically target performance.

---

## A base implementation, overridden down a type hierarchy

Motivation: pseudo-inheritance
Overlap: concrete types

Specialization was originally motivated in part as a way to get inheritance-like code reuse: a base
impl supplies default behavior, and more specific impls override it. `rucene` (a Lucene port) uses
this for its `Directory` abstraction.

```rust
pub trait Directory: Display {
    fn list_all(&self) -> Result<Vec<String>>;
    fn file_length(&self, name: &str) -> Result<i64>;
    fn open_input(&self, name: &str, ctx: &IOContext) -> Result<Box<dyn IndexInput>>;
}
default impl<T: FilterDirectory> Directory for T {
    fn list_all(&self) -> Result<Vec<String>> { self.dir().list_all() }
    fn file_length(&self, name: &str) -> Result<i64> { self.dir().file_length(name) }
    fn open_input(&self, name: &str, ctx: &IOContext) -> Result<Box<dyn IndexInput>> { self.dir().open_input(name, ctx) }
}
impl Directory for FSDirectory { /* a real on-disk directory */ }
impl<D: Directory, RL> Directory for RateLimitFilterDirectory<D, RL> { /* rate-limits writes, delegates the rest */ }
```

**Why it's here:** This was one of specialization's original motivations, yet the survey found strikingly little of it. This is the pattern that essentially just translates a Java-like class inheritance into Rust.
