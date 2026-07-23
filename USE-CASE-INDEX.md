# Specialization use-case index — one row per crate

This is the per-crate index behind [USE-CASES.md](USE-CASES.md). USE-CASES.md is the narrative:
it groups the ecosystem into a dozen or so distinct idioms and shows one real example of each.
This file is the underlying data, classifying many crates using the same criteria. This is useful to
gauge the depth and breadth of specialization, and to get some idea of how often each motivation comes up.

**Scope.** 142 crates — the sample of ecosystem crates that *author* specialization, minus 17 rows that
carry no use (vendored `std`/`libcore` forks, `rustfmt` formatting fixtures, `serde` `RUSTC_WITH_SPECIALIZATION`
shims, and two crates that enable the feature with no resolvable overlap).

**Columns.**

- **Motivation** — one of the six from USE-CASES.md: **performance**, **opt-in behavior**,
  **correctness**, **pseudo-inheritance**, **workaround**, **richer-libs**. Where a crate has a
  secondary motivation, the note says so.
- **Overlap** — what the more specific impl keys on: a **trait** the type has, or the **concrete
  type** (or a const).
- **Flags** — the cross-cutting properties from USE-CASES.md, plus two bookkeeping ones:
  - `default type` — the override changes an associated *type*, so the difference is observable at
    compile time (needs full `specialization`).
  - `cross-crate` — the override lives in a different crate from the blanket default.
  - `pseudo-reflection` — a fact about type identity is the *output*.
  - `std-mirror` — the crate reimplements a fast path that already exists inside `std`.
  - `nightly-gated` — the specialization is behind a Cargo feature or build cfg; the default build
    takes the generic path.
  - `no in-crate overlap` — the crate writes a `default` but nothing in it overrides that default
    (the override is downstream, or all sibling impls are `#[cfg]` twins or disjoint types). These
    rows say what the author *reached for*, not what the compiler resolves.

Crate names are as published on crates.io; the classification is of one version, and cites the
featured trait rather than every specializing impl in the crate.

***NOTE:** The classifications were done by an LLM after several rounds of refinement, and the document was largely written
by one as well. It is likely that some classifications will be wrong or imprecise, but on-the-large the larger patterns found
here should be relevant and interesting.

---

## Performance — same result, faster (48)

| Crate | Overlap | Flags | Note |
|---|---|---|---|
| `ahash` | concrete type | | `CallHasher`: `u64`/`str`/`[u8]` bypass the generic `Hasher` path. |
| `drtahash` | concrete type | | The same `CallHasher` idiom in an ahash fork. |
| `supply-chain-trust-example-crate-000069` | concrete type | | Republished `ahash` copy; same `CallHasher` fast paths. |
| `acgmath` | concrete type | | Macro-stamped operator impls; SIMD for concrete vector element types. |
| `bvh` | concrete type | nightly-gated | `Ray<f32>`/`Ray<f64>` intersect AABBs with SIMD instead of scalar math. |
| `custom_float` | concrete type | | Intrinsic `sqrt`/division overrides against a naive `powf` fallback. |
| `ostd` | concrete type | | Per-scalar single-instruction segment asm instead of an irq-guarded pointer read/write. |
| `mem_cmp` | trait, concrete type | | `MemEq`: bytewise `memcmp` instead of field-by-field comparison. |
| `sp-im` | trait | | `A: Eq`/`Copy` overrides add pointer-equality and linear-search shortcuts. |
| `strchunk` | concrete type | | Comparisons against concrete right-hand types monomorphize instead of routing through `Borrow<str>`. |
| `fmt-cmp` | concrete type | | `SpecEq`: concrete types compare with `==` instead of streaming both `Display`s. |
| `minivec` | trait, concrete type | std-mirror | Vendored `Vec` carrying std's `Clone`/`from_iter` fast paths. |
| `rune-alloc` | trait, concrete type | std-mirror | `SpecExtend`/`SpecFromElem` reimplemented for an allocator-aware `Vec`. |
| `alloc-wg` | trait, concrete type | std-mirror | Same, for an allocator-generic `alloc`. |
| `ve` | trait, concrete type | std-mirror | `IsZero`/`u8` paths use `with_capacity_zeroed`/`write_bytes`. |
| `slice-deque` | trait, concrete type | std-mirror | `SpecExtend`/`SpecFromElem` in a vendored deque. |
| `slice-ring-buffer` | trait, concrete type | std-mirror | Same, in a ring buffer. |
| `small_vec2` | trait, concrete type | std-mirror | `SpecExtend`/`SpecFromIter`/`SpecFrom`/`SpecWriteSlice` all reimplemented. |
| `two-sided-vec` | trait | std-mirror, no in-crate overlap | `IntoIter` allocation reuse and `Copy` slice copy against the generic clone path. |
| `staticvec` | trait | std-mirror | `Copy` overrides bulk-copy where the blanket clones. |
| `skew-heap` | trait | std-mirror | `SpecExtend` for a skew heap. |
| `btreemap_monstousity` | trait | std-mirror | Vendored `liballoc` `BTreeMap`/`BinaryHeap`. |
| `btree_monstousity` | concrete type | std-mirror | `IsSetVal`/`SetValZST`: skip value work when the map is really a set. |
| `placid` | trait | | `SpecInitSlice`: `TrivialClone` elements go through `copy_nonoverlapping`. |
| `emplacable` | trait | | `CopyToBuf`: `T: Copy` memcpys instead of cloning element by element. |
| `inplace-box` | trait | | `U: IsInPlaceBox` moves out of the source instead of copying into a new box. |
| `ax-io` | trait, concrete type | std-mirror | `io::copy`-style specialization: `&[u8]`/`Vec<u8>`/`BufReader` skip the stack buffer. |
| `axio` | trait, concrete type | std-mirror | The same reader/writer specialization in a second no-std io crate. |
| `data-streams` | concrete type | | `VecDeque<u8>` and slice sources copy in bulk. |
| `portable-io` | trait, concrete type | | `SizeHint` returns real bounds for `&[u8]`/`Take`/`Chain` instead of `(0, None)`. |
| `bulks` | trait, concrete type | `default type` | Exact lengths for `ExactSizeIterator`/`Vec`/slices; the collect result type is computed per impl. |
| `arrav` | concrete type | | `SpecializedLen::fast_len` for a fixed-size array type. |
| `glidesort` | trait | | Detects `T: Copy` at compile time to pick a sorting strategy. |
| `cpython` | trait | | `FromPyObject` takes the buffer protocol when the element type is `Copy`. |
| `alrulab-core` | concrete type | std-mirror | `ToString<N>`: manual digit and `&str` formatting instead of going through `Display`. |
| `jmespath` | concrete type | | `str`/`i32`/`Value` build a `Variable` directly rather than via `Serialize`. |
| `tucan` | concrete type | | `&str` takes a string-tuned interning path to the same interned value. |
| `into_string` | concrete type | | `AsCowStr`: `str` borrows instead of allocating through `to_string`. |
| `casperlabs-types` | concrete type | | Byte-typed vectors and arrays serialize in bulk; identical wire bytes. |
| `radiation` | concrete type | nightly-gated | `Vec<u8>` bulk-copies where `Vec<T>` parses element by element. |
| `cornflakes` | concrete type | no in-crate overlap | `DataSize` computes the same byte size per type without the generic walk. |
| `bitman` | trait | no in-crate overlap | Integer impls take the trait's default bit operations. |
| `extprim` | concrete type | nightly-gated | `u128`/`i128`/floats convert directly instead of through `ToPrimitive`. |
| `image_hasher` | concrete type | nightly-gated | `GrayImage` returns `Cow::Borrowed`, skipping a clone before hashing. |
| `lightws` | concrete type | nightly-gated | A masking client role writes the same frame head after masking the buffer. |
| `bart` | concrete type | | Numeric types skip the HTML-escape scan; numbers have no unsafe bytes to find. |
| `unroll-fn` | const | | `Const<0>` terminates a loop-unrolling recursion — the one arguable shipped const-value overlap. |
| `supply-chain-trust-example-crate-000029` | trait | std-mirror | Republished `smallvec` copy; `SpecFrom`'s `Copy` override memcpys. |

## Opt-in behavior — different, but not required (28)

| Crate | Overlap | Flags | Note |
|---|---|---|---|
| `debugit` | trait | | Prints `type_name::<T>()` unless `T: Debug`, in which case it prints the value. |
| `dbg` | trait | | `WrapDebug<T>` prints `[<unknown> of type …]` as its fallback. |
| `mockiato` | trait | | `MaybeDebug` writes `?`; `DefaultReturnValue` returns `None` unless the type is `()`. |
| `gcmodule` | trait | | `OptionalDebug` returns an empty string for GC diagnostics unless `T: Debug`. |
| `nvml` | trait | | `PersistentObject<T>` prints only the OID unless the payload is `Debug`. |
| `kas-core` | trait | | `TryFormat<T>` prints the type name as its fallback. |
| `ari` | trait | | `DefaultDebug<T>` type-name fallback; also a `Copy` `clear_vec` fast path (performance). |
| `hina` | trait | | The same `DefaultDebug` idiom, with the same `Copy` `clear_vec` aside. |
| `refs` | trait | | `Rglica<T>` prints the type name unless the target is `Debug`. |
| `buf_redux` | trait | | `BufReader`/`BufWriter` print `(no Debug impl)` unless the inner reader is `Debug`. |
| `anterofit` | trait | | `InterceptDebug` writes `Interceptor` unless the interceptor is `Debug`. |
| `rsubstitute_core` | trait | | `IDebugPrinter` returns `<unknown>` when a mock argument is not `Debug`. |
| `string-err` | trait | | `StringError` prints the inner error only when it is `Sized + Debug`. |
| `im` | trait | nightly-gated | `HashMap`/`HashSet` `Debug` and comparison upgrade when the key is also `Ord`. |
| `kawaii` | trait, concrete type | | Two blanket renderings (type name, `Debug`) with numeric and string overrides. |
| `syndicate` | concrete type | | Trace payloads render any `Debug` value as `Opaque`; `AnyValue` renders structurally. |
| `dabus` | trait | | `DynDebug`/`PossiblyClone` fire when `T: Debug`/`Clone`; otherwise a placeholder or a panic. |
| `tea-codec` | trait | | `*OrDefault` traits return an inert value unless `T` implements the probed trait. Separately, an `Equality<X, Y>: NotEqual` marker fakes negative reasoning to get a blanket `From` past the `From<T> for T` wall (workaround). |
| `frui_core` | trait | | `CheapEq`/`WidgetStateOS` are no-ops unless the type is a `Widget`/`RenderState`. |
| `print_bytes` | trait | | `ToConsole` returns `None` unless `T: AsHandle + Write`, i.e. unless the sink is a console. |
| `real_time_fir_iir_filters` | trait | | `DeserializeOrZeroed` zeroes the value unless `T: Deserialize`. |
| `datatest` | trait | | A generated test name when `T: ToString`, a positional one otherwise. |
| `termcandy` | trait | | A default widget draw when `W: Widget`. |
| `mutagen` | trait | | `Defaulter`/`MayClone` probe `Default`/`Clone` to decide what mutations can be injected. |
| `xarray` | trait | | `TryClone` returns `Some` only when the item is `Clone`, which is what enables copy-on-write. |
| `rusty-p4` | trait | | `ServiceStorage` downcasts to a real service only for `Send + Sync + 'static` types. |
| `wasm4pm-compat` | concrete type | | `EvidenceKind` labels a value `raw` unless it is an `Admitted<T>`. |
| `df_rocket_okapi` | concrete type | no in-crate overlap | Concrete response types (`Json`, `String`, `Vec<u8>`, `()`) describe their own OpenAPI schema. |

## Correctness — the right answer differs by type (30)

| Crate | Overlap | Flags | Note |
|---|---|---|---|
| `oasis-cbor` | concrete type | | `Vec<u8>`/`[u8; N]` encode as a CBOR byte string, not as an array. |
| `tea-runtime-codec` | concrete type | | Byte and string types pass through raw instead of via `serde_json`. |
| `azalea-buf` | concrete type | no in-crate overlap | Per-type big-endian encodings for the Minecraft wire protocol. |
| `casperlabs-contract-ffi` | concrete type | no in-crate overlap | Per-type binary parsing (CLType tag, length-prefixed integers, `URef`). |
| `datafusion-arrow` | concrete type | | `BooleanType` bit-packs one bit per element where the blanket writes a byte. |
| `serde_traitobject` | trait, concrete type | | Sized types, `str` and `[T]` go straight through serde; the blanket writes a vtable and type id — a different wire form. |
| `erdos` | trait | | `D: Abomonation` encodes with abomonation rather than bincode. |
| `anypack` | concrete type | | Concrete `From` overrides pack values into `Any` differently. |
| `msica` | concrete type | | `FromResidual` maps a known error kind to a status code instead of a generic failure. |
| `arc-reactor` | concrete type | nightly-gated | `(u16, T)` sets a status code; other overrides build entirely different error bodies. |
| `amadeus-parquet` | concrete type | `default type` | Each record type selects its own Parquet `Schema`/`Reader` type; boxed fixed-size arrays avoid the stack. |
| `llua` | concrete type | | `()` pushes zero Lua values and tuples push N where the blanket pushes one; arity errors corrupt the stack. |
| `discrete_system` | const | | `[T; 1]` returns its input unchanged instead of rotating a delay line. |
| `tallystick` | trait | | `T: Real` gets a real floor and fractional part; the blanket assumes integer semantics. |
| `weighted_levenshtein` | trait, concrete type | | Domain types supply real edit costs instead of a flat cost of 1, changing the distance. |
| `maths-traits` | trait | | `Z: Zero` detects zeros, which changes the pow/mul algebra taken. |
| `ezel` | concrete type | | `f64` reads the DataFrame column; the blanket panics for a `Column`. |
| `arendur` | trait | | `T: Primitive` records a primitive hit and returns itself as a light. |
| `mbox` | concrete type | | `[T]`/`str` run `drop_in_place` through the fat pointer before freeing — required for correct deallocation. |
| `libhermit-rs` | trait | | A subtable level recurses; the blanket reads a leaf page-table entry. |
| `rusty-hermit` | trait | | The same page-table walk in the kernel's other crate. |
| `rdma-core` | concrete type | | Fourteen concrete `*mut ibv_*` types read the context pointer directly rather than through the protection domain. |
| `krnl-core` | concrete type | | `AsScalar` performs the real cast for concrete numeric types; the blanket is unreachable. |
| `dll-syringe` | concrete type | | The float-register ABI mask is derived from the concrete function signature. |
| `compact` | trait, concrete type | | `T: Copy` compacts with a bulk copy; types with a dynamic part relocate it. |
| `partial-uninit` | concrete type | | A no-op blanket so generic code compiles; concrete overrides do the real partial initialization. |
| `azalea-inventory` | concrete type | no in-crate overlap | Fifty-nine concrete components supply real per-item default data. |
| `sqlxo_traits` | trait | | `T: Deletable` yields the soft-delete marker field that the generated query keys on. |
| `midenc-hir-analysis` | trait, concrete type | | Executable ops re-enqueue their blocks; per-type lattice anchors classify differently. |
| `network-collections` | concrete type | no in-crate overlap | Macro-stamped per-type `Deserialize` impls. |

## Pseudo-inheritance — a base impl, overridden by more specific types (13)

| Crate | Overlap | Flags | Note |
|---|---|---|---|
| `rucene` | trait | | `default impl<T: FilterDirectory> Directory` supplies delegating methods; concrete directories override what they need. |
| `kg-diag` | trait | | `Detail`/`Diag` defaults; concrete impls override only severity and code. |
| `enso-generics` | trait | no in-crate overlap | Lone `default fn` head/tail clone helpers, written to be overridden. |
| `enso-logger` | trait | no in-crate overlap | Default log formatting, overridable per level and backend. |
| `midenc-hir` | trait | | `AsAny`/`DynHash`/`DynPartialEq` boilerplate defaults; entity hooks overridden by real operations. |
| `mayda` | concrete type | | Blanket methods panic `not implemented` so the generic codec type compiles; macro-stamped concrete impls do the real work. |
| `shio` | trait | no in-crate overlap | `FromState`'s default delegates to `request.try_get`. |
| `pessimize` | trait | nightly-gated | A register-spill fallback for arbitrary types; concrete types override with inline asm. |
| `observatory` | trait | cross-crate, no in-crate overlap | `IsUnchanged` returns false; downstream types override change detection. |
| `promptly` | trait | cross-crate | A default `Promptable` prompt, overridden per type. |
| `pi_world` | trait | cross-crate | An ECS `FromWorld` default with per-type overrides. |
| `try-clone` | trait | cross-crate | `TryClone` forwards to `Clone`; types like `OwnedFd` override with a genuinely fallible impl. |
| `authzen-diesel-core` | concrete type | `default type`, cross-crate | The soft-delete filter *type* itself is overridden per query type. |

## Workaround — faking a feature the language lacks (13)

| Crate | Overlap | Flags | Note |
|---|---|---|---|
| `linux-support` | trait | | `Drop` is routed through a `SpecDrop` trait because a `Drop` impl cannot carry bounds (E0367). |
| `clear_on_drop` | concrete type | nightly-gated | The `Sized` and `?Sized` impls differ only in an asm operand constraint, to keep the optimizer from eliding the wipe. |
| `bsalib` | trait | `default type`, no in-crate overlap | The feature is enabled only to give an associated type a default. |
| `typed-eval` | trait | no in-crate overlap | A blanket registration default whose siblings are `cfg(not(nightly))` twins. |
| `const-alg` | concrete type | no in-crate overlap | Concrete matrix impls with no blanket for them to override. |
| `treeid` | concrete type | no in-crate overlap | A lone override of a foreign trait. |
| `pi_ecs` | concrete type | no in-crate overlap | A lone `default fn` in an impl of a foreign trait. |
| `atlas-frozen-abi` | concrete type | cross-crate, pseudo-reflection | `AbiExample` panics by default for *every* type so a recursive ABI digester compiles; downstream crates derive real overrides. |
| `cbe-frozen-abi` | concrete type | cross-crate, pseudo-reflection | Same, in another Solana chain fork. |
| `jupnet-frozen-abi` | concrete type | cross-crate, pseudo-reflection | Same. |
| `safecoin-frozen-abi` | concrete type | cross-crate, pseudo-reflection | Same. |
| `trezoa-frozen-abii` | concrete type | cross-crate, pseudo-reflection | Same. |
| `clone-solana-frozen-abi` | concrete type | cross-crate, pseudo-reflection | Same; `AbiEnumVisitor` layers a second set of defaults. |

The six `*-frozen-abi` rows are chain forks that fell into the sample. Upstream, this one idiom
accounts for roughly 248 further crates that enable specialization *only* to host a derived
override of `solana-frozen-abi`'s blanket default — by crate count, the largest single use of
specialization in the ecosystem.

## Richer-libs — type-level machinery as the product (10)

| Crate | Overlap | Flags | Note |
|---|---|---|---|
| `option_trait` | concrete type | `default type` | `Maybe*` generalizes `Option` to the type level; the overlap computes the output type. |
| `rats` | concrete type | `default type` | `IntoKind` lowers types into higher-kinded `Kind`s; `Vec` and boxed futures override to avoid a double box. |
| `metatype` | concrete type | `default type`, pseudo-reflection | A per-kind pointer-metadata type, plus a `default const METATYPE` tag read at runtime. |
| `specialize` | concrete type | `default type`, pseudo-reflection | `obj.is::<T>()` returns a runtime bool; the downcast target is an associated type. |
| `scrapmetal` | concrete type | pseudo-reflection | `Cast<T>` returns `Ok` when `U` is `T` and `Err` otherwise. |
| `slice_trait` | concrete type | pseudo-reflection | `Same<T>`, the same type-equality probe in const form. |
| `const-type-layout` | concrete type | `default type`, pseudo-reflection | Compile-time type layout, per kind. |
| `enso-prelude` | concrete type | `default type`, pseudo-reflection | `TypeDisplay` names, overridable per type. |
| `rebound` | concrete type | pseudo-reflection | `ReflectedImpl<N>` returns real associated-fn and const metadata for concrete types; the blanket returns nothing. |
| `trait-cast` | concrete type | pseudo-reflection | `Sized` targets forward to `Any::downcast`; unsized targets go through a vtable traitcast. |

---

## What the distribution shows

- **The two kinds of overlap are close to evenly split**: 68 rows key on a concrete type or const,
  57 on a trait the type has, 17 on both. They cluster differently, though — trait overlap
  dominates performance (marker traits like `Copy`/`TrustedLen`) and opt-in behavior (probing
  `Debug`), concrete overlap dominates correctness (codecs) and richer-libs.
- **Performance is a third of the sample, and a third of that is `std`'s own work carried into the
  ecosystem** — 16 crates reimplement `SpecExtend`/`SpecFromElem`/`io::copy` because they ship or
  fork a container. These want no new expressiveness; they want access to what `std` already does.
- **One idiom dominates opt-in behavior**: 14 of the 28 rows are the same "print the value if
  `T: Debug`, otherwise print the type name" fallback, arrived at independently.
- **`default type` is rare but concentrated** — 10 rows, and 6 of them are richer-libs, where the
  associated type *is* the library's output. Nothing substitutes for it there.
- **`no in-crate overlap` is common (17 rows, 12%)**: the author writes a `default` whose
  override lives downstream, or writes one to reserve the ability. Cross-crate overriding is a real
  use, not an artifact.
