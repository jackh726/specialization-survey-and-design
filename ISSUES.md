**Note:** This document serves to triage known issues related to specialization,
including the `specialization`, `min_specialization`, and `try_as_dyn` features
(the last because it is a potential replacement for a *subset* of specialization
uses).

# Specialization issue triage

## Methodology

- **The set.** 103 numbered issues: open issues under the three specialization labels (75 after
  de-duplication), 9 closed `I-unsound` issues, 5 closed design-decision threads, the 4 `try_as_dyn`
  issues, and 10 more found by full-text search. Plus 59 bug reports that exist only as comments on
  the tracking issue and were never given a number, and the tracking issue's own unresolved
  `try_as_dyn` questions.
- **Labels are not a sufficient index.** A label-only first pass missed 6 genuine issues, recovered
  by searching code and issue text for `min_specialization`, `rustc_specialization_trait`,
  `rustc_unsafe_specialization_marker`, and `default impl/fn/type`. Labels are reliable for the
  feature surface but not for std-internal bugs, which are filed under `T-libs` / `A-iterators` /
  `I-unsound` and name the symptom, not the mechanism. A future pass should also grep the std source
  for the `Spec*` helper traits, `InPlaceIterable` / `SourceIter`, and the marker attributes.
- **Verification.** Every issue was reduced to a minimal reproduction, compiled, and run where
  behaviour was at stake, on nightly 1.98.0 (`61d7280f3`) with `-Znext-solver` as a second config.
  Nothing is invented: code comes from the issue, its comments, or a maintainer's regression test.
- **Status tokens** (used per entry below): `repro` same symptom today; `fixed` no longer occurs;
  `next-solver` fails today but is clean under `-Znext-solver`; `changed` still wrong but
  differently; `no-repro` could not reproduce; no token means nothing to compile. A `fixed` on an
  open issue means it is closeable now.

These issues were triaged and this document written in large part by an LLM,
with review and oversight. Followup actions on any issues were/are all done
manually.

## What the triage found

**Shape of the open set.** 84 open numbered issues, by disposition:

- 13 are already fixed on nightly (closeable now).
- 7 are fixed by the next solver and nothing else (one bug; close on migration).
- 9 are stale as written (7 behave differently now, 2 no longer reproduce): they need a retitle, not a fix.
- 4 are tracking or design-meta.
- 51 genuinely reproduce.

**Far fewer causes than reports.** The 51 reproducing issues (plus the 7 the next solver fixes)
come down to about 11 recurring root causes, plus a tail of roughly 7 one-off issues that share a
cause with nothing else. A narrow fix usually closes several issues at once. The largest single
cause, projection through a `default type`, covers roughly a quarter of them by itself. Duplication
runs two ways: several ICEs are one compiler assertion reached from different inputs, and where a
`min_specialization` report and a full-`specialization` report describe the same hole they are one
cause.

**By type, the causes are mostly not bugs.** Of the ~11: one is the region-erasure soundness hole,
two are ICE-class compiler assertions, one is diagnostics, one is a silent-non-firing performance
pattern, and the remaining six are feature gaps or unresolved design decisions (`default type`
projection, entangled defaults, negative reasoning and lattice overlap, the const-trait interaction,
`Drop`, and the permanent-commitment problem of shipping a blanket impl). Some causes surface only
alongside another unstable feature; the ones that recur are const traits, RPITIT/AFIT, GATs,
`marker` traits, and negative impls.

**Soundness is not closed, but it is contained.** The sound subset (`min_specialization`) still has
a live hole in its always-applicable check: [#149257](https://github.com/rust-lang/rust/issues/149257),
which SIGILLs on nightly. std has shipped a handful of specialization
soundness bugs of its own; most are fixed, a couple are still live. Across nine years, none of the
59 unfiled comment-reports is a soundness bug: every unsoundness arrived through a numbered issue.

**What users trip over is not `default`.** Of the 29 duplicate or not-a-bug comment-reports, the
largest group (7) wrote a concrete impl that is not a provable subset of a blanket impl whose bound
is a foreign trait. The missing feature there is negative reasoning, not specialization; `default`
would not have helped.

## Root causes

### 1. Region erasure (the soundness hole)

Typeck sees lifetimes; codegen erases them, so an impl whose applicability depends on a lifetime can
be selected during typeck yet be irreproducible at codegen. **#40582, #45982, #79457, #149257 are
one hole from four angles**, differing only in where the region-sensitivity sits: the specializing
impl's type, the base impl's `'static` bound, or a predicate.

The proposed fix is **always applicable impls** (#48538, nikomatsakis 2018): an impl qualifies if it
imposes no conditions on its input types beyond well-formedness. The check *is* implemented, but
only as `min_specialization`'s four tests; **intersection impls, the other half of the proposal, have
zero implementation**. #48538's rule is under two-sided pressure: #149257 shows it is too permissive,
#113452 shows it is too restrictive, and both point at the same syntactic clause-matching code.

Two issues in this area are *not* lifetime-blindness and would not be fixed by it: #85863 and #85873
are subtype/supertype coercion of HRTB function pointers across a covariant type, where both types
are already `'static`.

An alternative does exist: the always-applicable rule can be dropped if the specialization decision
is made during *analysis* (where regions are still known) rather than at codegen.

### 2. `default type` is not projectable

Generic code cannot rely on a `default` associated type's value, because a downstream impl may
override it. Correct by design, and the most common thing users trip over. The mechanism is
`Ancestors::leaf_def` in
`compiler/rustc_middle/src/traits/specialization_graph.rs`: a definition is usable only once
*finalized*, where the first descendant that supplies the item non-`default`, **or omits it
entirely**, finalizes it. The test is purely syntactic and never asks whether a child impl could
exist. That is why #106700 is the one defensible complaint in the cluster: its defining impl is for
a concrete type, so no override is possible, yet the projection stays opaque.

Eight issues are this rule (#85228, #98389, #106700, #132804, #46707, #50318, #52396, #33481), and
the consumer-side ICEs (#88296, #152405, #156484) are downstream of it: a lint or rustdoc receiving
a rigid projection it assumed had been normalized. RPITIT/AFIT desugars to an associated type, which
is how #108309 smuggles `default type` past `min_specialization`'s ban on it.

### 3. Entangled defaults

A `default fn` may not rely on a sibling `default type` in the same impl, since an override may
replace one and not the other. RFC 1210's own `Add`/`add_assign` reuse example does not typecheck
for this reason, noticed by RalfJung in 2020, confirmed by nikomatsakis, never fixed. The proposed
remedies are `default { … }` item groups, override-any-means-override-all, and inferred groupings;
none was adopted. #150087 is the current user-facing face of this; #70442 is its dual, asking when a
default *stops* being overridable. #125295 proposed whole-impl granularity as the answer and was
closed as not actionable, on the grounds that it does nothing about region erasure.

This is the longest-running unresolved thread in the corpus: 11 of the 59 unfiled reports are it,
spanning 2016-07 to 2023-02. The decision of record is nikomatsakis's, that default groups were
"kicked around … and ultimately didn't adopt any solution because we figured we could get to it
later". **Both of RFC 1210's flagship examples are instances of it, and neither has ever compiled.**

### 4. Specialization leaking into inference

Specialization is supposed to be transparent to inference. Seven issues are the counterexamples, and
they share one cause: the old solver's winnowing drops the parent `default` candidate whenever the
specializing impl applies *after* constraining inference variables, leaving one candidate and
committing the variable. PR #140306 fixed this in the new solver by only letting a specializing
candidate evict its parent when it applies with no non-region inference constraints, which is
"specialization does not affect inference", encoded. **No issue thread cites it.**

The cost, visible in #46363: under that rule, adding `default` to an existing blanket impl *weakens*
downstream inference, and is therefore a breaking change.

### 5. Negative reasoning is unavailable

#42721 and #46813 are the bound side and impl side of one gap. Specialization structurally cannot
substitute: where neither impl is a subset of the other there is no graph edge to record. Nightly
has `#![feature(negative_bounds)]` but it is half-implemented: coherence will *accept* a negative
bound, but the solver cannot *prove* one, so the impl is uncallable. Exclusion groups, negative
supertrait bounds, and reverse-polarity specialization were all floated on #31844 and none adopted.
The coherence objection is stability, not mechanics: with negative reasoning, adding an impl becomes
a breaking change.

### 6. Specialization needs a decidable answer at monomorphization

#147507 (an inductive cycle) and #96235 (a `#[marker]` trait's union of impls) both drive
monomorphization to a specialization query with no unique answer, which nothing downstream handles,
so it ICEs, in `Instance::resolve` and the monomorphize collector respectively. Distinct panic
sites, one gap. lcnr, in thread: "a pretty big and fundamental issue with specialization". Not fixed
by the new solver. `try_as_dyn` inherits this rather than escaping it, since it runs the same solver
(#151440 cross-references #147507 explicitly).

### 7. Adding a blanket impl is a breaking change regardless of `default`

aturon's 2017 result, reached after the hope that `default` could make blanket impls safe to add.
A downstream impl may be justified by the *absence* of an upstream one, so `default` cannot serve as
a "safe to add later" marker and specialization does not buy back the `#[fundamental]` restriction.

There are three std API changes cited here as "blocked on specialization": #52454, #45742, #129039.
Each are "blocked" by a *different* mechanism:

- **#52454** (mark core's `Into::into` `default`) does not add a blanket at all: the blanket already
  exists, its method was stabilized without `default` (so overriding it is E0520), and adding
  `default` now weakens downstream inference (a consequence of root cause 4). It is blocked on *stable
  specialization plus the permanence of the `default` choice*, not on a missing feature.
- **#129039** (`PartialOrd<[U]> for [T]`) is a **min_specialization limitation**: giving
  `SlicePartialOrd` the right-hand-side parameter it needs makes the `AlwaysApplicableOrd` fast path
  map two parent parameters to one, which min_spec's `check_duplicate_params` rejects ("specializing
  impl repeats parameter `A`") even though **full specialization accepts it** (verified: the MCVE
  compiles under `feature(specialization)`, fails only under `min_specialization`). No
  breaking-change hazard and no new feature; the fix is to relax or parameterize the min_spec rule.
- **#45742** (AsRef/AsMut over `Deref`) is the only one that genuinely needs a new language feature:
  the proposed blanket overlaps existing impls without being a superset, so chain-rule specialization
  cannot order them, so it needs **intersection/lattice impls**.

So this root cause is real as a stabilization obstacle, but of the three it is only loosely the
blocker for #45742 (via coherence) and not the blocker for #52454 (inference/permanence) or #129039
(a min_spec strictness the full feature does not share).

## Cross-feature interactions

Issues that need a *second* unstable feature to appear (the `requires` column): points where the
design must account for work in flight.

| Interacting feature | Issues | What it contributes |
|---|---|---|
| `const_trait_impl` | #147130, #148200 | constness is not a dimension the specialization graph can vary along, in either direction; #148200 is a hole in the const-check guarantee |
| RPITIT / AFIT | #108309 | desugars to an associated type, smuggling `default type` past min_spec |
| `marker_trait_attr` | #96235 | a marker trait's union of impls yields ambiguity at mono |
| `trait_alias` | #74809 | alias not expanded when building the specialization graph (now fixed) |
| `track_caller` | #70293 | not inherited through specialization ancestors |
| `negative_impls` | #46813 | conditional negative impls silently discard their where clause |
| `rustc_attrs` | #102252, #90665 | `rustc_specialization_trait` on a circular associated type |

## Sources beyond the tracker

- RFC 1210 (rust-lang/rfcs#1210); tracking issue [#31844][31844]
- aturon, *Specialization, coherence, and API evolution*, 2017-02-06:
  <http://aturon.github.io/tech/2017/02/06/specialization-and-coherence/>
- nikomatsakis, *Maximally minimal specialization: always applicable impls*, 2018-02-09:
  <https://smallcultfollowing.com/babysteps/blog/2018/02/09/maximally-minimal-specialization-always-applicable-impls/>
- aturon, *Sound and ergonomic specialization for Rust*, 2018-04-05:
  <http://aturon.github.io/tech/2018/04/05/sound-specialization/>
- lcnr, *On always-applicable trait impls*, 2026-03-06 (argues the always-applicable rule is not
  needed; resolve specialization during analysis via caller-propagated `maybe` bounds instead):
  <https://lcnr.de/blog/2026/03/06/always-applicable.html>
- nikomatsakis, *Supporting blanket impls in specialization*, 2016-10-24
- rust-lang/chalk#9: two encodings of specialization in the logic engine
- rust-lang/types-team#89, #119: "path towards MVP stabilization (if any)"
- rust-lang/effects-initiative#6: relationship to keyword generics and `const_eval_select`
- rust-lang/unsafe-code-guidelines#307: whether `Copy` impls are semantically relevant; bears on #132442
- PR #68970 (`min_specialization`), PR #140306 (specialization in the new solver),
  PR #121047 (stop assembling `default impl` as a candidate), PR #135634 (`TrivialClone`)
- 2026 project goal: <https://rust-lang.github.io/rust-project-goals/2026/specialization.html>;
  status issue rust-lang/rust-project-goals#652

[31844]: https://github.com/rust-lang/rust/issues/31844

---


# Per-issue index

One row per issue, grouped by root-cause cluster. Columns:

- **state**: `open` or `closed` on GitHub as of this pass.
- **status**: whether it still reproduces on nightly, independent of GitHub state: `repro`, `fixed`,
  `next-solver` (fails today, clean under `-Znext-solver`), `changed`, `no-repro`, or blank for
  nothing-to-compile.
- **tags**: the `default` construct the reduction uses (`def-type`, `def-impl`, `def-const`; plain
  `default fn` is left untagged); the feature it needs (`spec`, `min-spec`, `both`, `dyn`); `std`
  for a bug in std's own specializations; any secondary feature; and `no-mcve` where there is no
  reduction.
- **dupe of**: set only on issues to close as a duplicate, pointing at the canonical issue that is
  kept. The canonical's own cell is blank.
- **fix**: a merged PR, `#140306` for the next-solver inference fix, `blocked: <what>`, or a
  proposed direction.
- **place**: a few words from the title.

How to close each row:

- **state `closed`**: done.
- **`dupe of` is set**: close as a duplicate of that issue; do not write a test for it, the canonical
  carries the test.
- **`open`, status `fixed`, no `dupe of`**: add a regression test and close when it merges (cite the
  `fix` PR if shown).
- **`open`, status `next-solver`**: close when the new solver becomes the default.
- **`open`, status `changed`**: retitle or correct; the issue as written is now wrong.
- **`open`, status `repro` or `no-repro`, no `dupe of`**: real backlog; keep open (`fix` names what
  it waits on, if anything).

The tracking issue #31844 is the parent of all of these and is not repeated as a row.

## Region erasure and the soundness hole

| # | state | status | tags | dupe of | fix | place |
|---|---|---|---|---|---|---|
| [#40582](https://github.com/rust-lang/rust/issues/40582) | open | repro | `spec` |  | blocked: always-applicable-impls | Specialization and lifetime dispatch |
| [#45982](https://github.com/rust-lang/rust/issues/45982) | closed |  | `spec` `no-mcve` |  | prop: always-applicable-impls | (RFC 1210) specialization: restrictions… |
| [#79457](https://github.com/rust-lang/rust/issues/79457) | closed | fixed | `both` |  | #111252 | With min_specialization enabled, an… |
| [#149257](https://github.com/rust-lang/rust/issues/149257) | open | repro | `both` |  | needs make-min_spec-is_global-region-aware | min_specialization is unsound due to… |
| [#48538](https://github.com/rust-lang/rust/issues/48538) | open |  | `both` `no-mcve` |  | prop: always-applicable-impls / partially landed:#68970 | 🔬 implement "always applicable impls |
| [#113452](https://github.com/rust-lang/rust/issues/113452) | open | repro | `min-spec` |  | needs implied-bounds-from-local-impls | min_specialization on local types should take… |
| [#105782](https://github.com/rust-lang/rust/issues/105782) | closed | fixed | `def-type` `spec` |  | #130654 | specialization: default items completely drop… |

## Soundness bugs in std's own specializations

| # | state | status | tags | dupe of | fix | place |
|---|---|---|---|---|---|---|
| [#85863](https://github.com/rust-lang/rust/issues/85863) | closed | fixed | `min-spec` `std` |  | #86765 | iter::Fuse is unsound with how specialization… |
| [#85873](https://github.com/rust-lang/rust/issues/85873) | closed | fixed | `min-spec` `std` |  | #85874 | TrustedRandomAccess optimization for Zip… |
| [#85969](https://github.com/rust-lang/rust/issues/85969) | closed | fixed | `min-spec` `std` |  | #85975 | Using Zip & Take allows accessing the… |
| [#137255](https://github.com/rust-lang/rust/issues/137255) | closed | fixed | `min-spec` `std` |  | #141076 | Panic-safety issue with Zip specializations |
| [#144012](https://github.com/rust-lang/rust/issues/144012) | open | repro | `min-spec` `std` |  |  | The Zip iterator does not update the… |
| [#67194](https://github.com/rust-lang/rust/issues/67194) | closed | fixed | `min-spec` `std` |  | #68358 | PartialEq implementation for RangeInclusive… |
| [#132442](https://github.com/rust-lang/rust/issues/132442) | closed | fixed | `min-spec` `std` |  | #135634 | Array and Vec's Clone specialization is maybe… |
| [#33017](https://github.com/rust-lang/rust/issues/33017) | closed | fixed | `def-type` `spec` |  | #84496 | Trait bounds not checked on specializable… |
| [#74299](https://github.com/rust-lang/rust/issues/74299) | closed | fixed | `def-type` `spec` |  | #121848 | Incoherent impls are allowed on default… |

## Associated types, `default type`, and projection

| # | state | status | tags | dupe of | fix | place |
|---|---|---|---|---|---|---|
| [#32483](https://github.com/rust-lang/rust/issues/32483) | open | fixed | `def-type` `spec` |  | landed | Specialization: allow some projection when… |
| [#50318](https://github.com/rust-lang/rust/issues/50318) | open | fixed | `def-type` `def-impl` `spec` |  | landed | assigning associated type in a default impl… |
| [#52396](https://github.com/rust-lang/rust/issues/52396) | open | fixed | `def-impl` `spec` |  |  | Default impls cannot take into account… |
| [#46707](https://github.com/rust-lang/rust/issues/46707) | open | repro | `def-type` `spec` |  | needs sealed/final-impl analysis | associated types are not evaluated even on… |
| [#85228](https://github.com/rust-lang/rust/issues/85228) | open | repro | `def-type` `spec` |  |  | Specialization on associated type |
| [#98389](https://github.com/rust-lang/rust/issues/98389) | open | repro | `def-type` `spec` |  |  | Specialized associated type doesn't have… |
| [#106700](https://github.com/rust-lang/rust/issues/106700) | closed | repro | `def-type` `spec` |  | needs treat `default` on a maximally-specific impl as vacuous | Specialized associated type cannot be… |
| [#132804](https://github.com/rust-lang/rust/issues/132804) | open | repro | `def-type` `spec` |  |  | Incorrect expected type for associated type… |
| [#108309](https://github.com/rust-lang/rust/issues/108309) | open | changed | `both` `rpitit` |  | #108551 | Weird interaction between specialization and… |
| [#106710](https://github.com/rust-lang/rust/issues/106710) | open | changed | `spec` |  | landed | Cow<'a, T> isn't specializable while… |
| [#68309](https://github.com/rust-lang/rust/issues/68309) | open | repro | `spec` |  | prop: `default fn f;` forwarding syntax | Specialization should allow delegation to the… |
| [#37653](https://github.com/rust-lang/rust/issues/37653) | open | fixed | `def-impl` `spec` |  | #45404,#46455 | support default impl for specialization |

## Specialization leaking into inference

| # | state | status | tags | dupe of | fix | place |
|---|---|---|---|---|---|---|
| [#36262](https://github.com/rust-lang/rust/issues/36262) | open | next-solver | `both` |  | #140306 | Specialization influnces inference |
| [#38516](https://github.com/rust-lang/rust/issues/38516) | closed | next-solver | `def-impl` `both` | #36262 | #140306 | Specialization does not find the default impl |
| [#40718](https://github.com/rust-lang/rust/issues/40718) | closed | next-solver | `both` | #36262 | #140306 | Type inference incorrectly selects… |
| [#41597](https://github.com/rust-lang/rust/issues/41597) | closed | next-solver | `both` | #36262 | #140306 | Specialization results in spurious mismatch… |
| [#46363](https://github.com/rust-lang/rust/issues/46363) | open | repro | `both` |  | #140306 | Adding a specialized impl can break… |
| [#67918](https://github.com/rust-lang/rust/issues/67918) | closed | next-solver | `def-impl` `both` | #36262 | #140306 | Specialization works only if type annotation… |
| [#43048](https://github.com/rust-lang/rust/issues/43048) | open | repro | `both` |  | blocked: cross-crate negative reasoning | Possibly spurious overlapping impl (even with… |
| [#91973](https://github.com/rust-lang/rust/issues/91973) | closed | next-solver | `def-impl` `both` | #36262 | #140306 | Strange behavior when specializing over a… |
| [#55243](https://github.com/rust-lang/rust/issues/55243) | closed | next-solver | `def-impl` `both` | #36262 | #140306 | Derived trait shadows a blanket default impl,… |
| [#81376](https://github.com/rust-lang/rust/issues/81376) | open | repro | `both` | #43048 | blocked: cross-crate negative reasoning | Conflicing implementation through specialized… |

## ICEs in min_specialization and the specialization graph

| # | state | status | tags | dupe of | fix | place |
|---|---|---|---|---|---|---|
| [#132519](https://github.com/rust-lang/rust/issues/132519) | open | repro | `def-impl` `spec` |  | needs next-solver | When translating generic parameters from… |
| [#126268](https://github.com/rust-lang/rust/issues/126268) | open | repro | `min-spec` | #102252 | needs next-solver | assertion failed: !obligations.has_infer() in… |
| [#103708](https://github.com/rust-lang/rust/issues/103708) | open | repro | `min-spec` |  | blocked: impl_wf_check permits unconstrained lifetime params | min_specialization ICE is not fully resolved… |
| [#157731](https://github.com/rust-lang/rust/issues/157731) | open | repro | `min-spec` `assumptions` `next-solver` | #103708 |  | assumptions on binders: is not fully resolved |
| [#96235](https://github.com/rust-lang/rust/issues/96235) | open | no-repro | `spec` `std` `marker` `attrs` `no-mcve` |  |  | in rustc_monomorphize/src/collector.rs when… |
| [#150387](https://github.com/rust-lang/rust/issues/150387) | open | repro | `both` |  | prop: forbid specializing Drop impls | from specializing Drop impl with impossible… |
| [#147507](https://github.com/rust-lang/rust/issues/147507) | open | repro | `spec` |  |  | from inductive cycle in specialization |
| [#102252](https://github.com/rust-lang/rust/issues/102252) | open | repro | `min-spec` `attrs` |  | needs next-solver | rustc_specialization_trait and circular… |
| [#119344](https://github.com/rust-lang/rust/issues/119344) | open | repro | `spec` |  |  | query cycle: cycle detected when building… |
| [#125014](https://github.com/rust-lang/rust/issues/125014) | open | next-solver | `def-type` `spec` |  | next-solver | :coherence: impl was matchable against Binder… |

## ICEs in downstream consumers, and compiler hangs

| # | state | status | tags | dupe of | fix | place |
|---|---|---|---|---|---|---|
| [#88296](https://github.com/rust-lang/rust/issues/88296) | open | repro | `def-type` `spec` |  |  | in improper_ctypes lint for specialised… |
| [#152405](https://github.com/rust-lang/rust/issues/152405) | open | repro | `def-type` `spec` | #88296 |  | in improper_ctypes lint with specialization… |
| [#156484](https://github.com/rust-lang/rust/issues/156484) | open | repro | `spec` |  | prop: reject-empty-specializing-impls | rustdoc: Deref impl without Target type |
| [#77026](https://github.com/rust-lang/rust/issues/77026) | open | fixed | `def-impl` `spec` |  | #121047 | compiler crash using specialization |
| [#80700](https://github.com/rust-lang/rust/issues/80700) | open | fixed | `def-type` `def-impl` `spec` |  | landed | Overflow in checking recursive trait… |
| [#98478](https://github.com/rust-lang/rust/issues/98478) | closed | fixed | `def-impl` `spec` | #48515 | #121047 | default impl SuperTrait for impl SubTrait =>… |
| [#117909](https://github.com/rust-lang/rust/issues/117909) | closed | fixed | `def-impl` `spec` | #48515 | #121047 | Compiler hang with recursive trait… |

## Diagnostics

| # | state | status | tags | dupe of | fix | place |
|---|---|---|---|---|---|---|
| [#33481](https://github.com/rust-lang/rust/issues/33481) | open | repro | `def-type` `spec` |  |  | Unhelpful error message for… |
| [#55140](https://github.com/rust-lang/rust/issues/55140) | open | repro | `def-impl` `both` |  |  | One of the E0599 notes disappears when… |
| [#58809](https://github.com/rust-lang/rust/issues/58809) | open | changed | `def-impl` `spec` `fn-traits` `fnonce` | #85228 |  | Specialized FnOnce impl compiles failed with… |
| [#90665](https://github.com/rust-lang/rust/issues/90665) | open | repro | `both` `attrs` |  | needs keep nested-obligation info when winnowing yields zero candidates | Diagnostic forgets about transitive trait… |
| [#97296](https://github.com/rust-lang/rust/issues/97296) | open | changed | `min-spec` |  | needs explain-rustc_specialization_trait-restriction | Diagnostic regression when rustc cannot… |
| [#117841](https://github.com/rust-lang/rust/issues/117841) | open | repro | `def-type` `spec` |  |  | Invalid help suggestion for specialization on… |
| [#130102](https://github.com/rust-lang/rust/issues/130102) | open | repro | `min-spec` |  | needs point-at-elided-dyn-lifetime-and-suggest-'a | Confusing error message when using… |
| [#90817](https://github.com/rust-lang/rust/issues/90817) | open | fixed |  |  | landed | Cycle detected when computing whether impls… |
| [#74809](https://github.com/rust-lang/rust/issues/74809) | open | fixed | `spec` `alias` |  | landed | Error in specialization type checking when… |
| [#48515](https://github.com/rust-lang/rust/issues/48515) | open | fixed | `def-impl` `spec` |  | landed | Evaluation overflow with specialization… |

## Feature gaps

| # | state | status | tags | dupe of | fix | place |
|---|---|---|---|---|---|---|
| [#42721](https://github.com/rust-lang/rust/issues/42721) | open | repro | `both` `neg` |  | blocked: negative-reasoning-semantics | Need negative trait bound |
| [#46813](https://github.com/rust-lang/rust/issues/46813) | open | repro | `spec` `auto` `neg` |  | prop: #79098 forbid conditional negative impls | Negative impls of Auto Traits (OIBIT) don't… |
| [#46893](https://github.com/rust-lang/rust/issues/46893) | open | repro | `def-impl` `both` |  | needs skip-dropck-check-when-a-default-impl-covers-it; blocked-on:needs_drop-must-not-vary-with-generics | Can't specialize Drop |
| [#48444](https://github.com/rust-lang/rust/issues/48444) | open | changed | `spec` |  | prop: reject-empty-specializing-impl | specialization permits empty impls when… |
| [#45542](https://github.com/rust-lang/rust/issues/45542) | open | repro | `def-impl` `both` |  | blocked: potential-specialization-in-coherence | cannot specialize an impl of a local… |
| [#70293](https://github.com/rust-lang/rust/issues/70293) | open | repro | `both` |  | needs run should_inherit_track_caller over specialization ancestors | #[track_caller] should inherit through… |
| [#68564](https://github.com/rust-lang/rust/issues/68564) | open | repro | `def-impl` `spec` |  |  | specialization: default impl is used,… |

## std API changes blocked on specialization, and two design alternatives

| # | state | status | tags | dupe of | fix | place |
|---|---|---|---|---|---|---|
| [#150087](https://github.com/rust-lang/rust/issues/150087) | open | repro | `def-type` `spec` |  | needs default-item-groups | Specializing Option<T> for bool with… |
| [#129039](https://github.com/rust-lang/rust/issues/129039) | open | repro | `both` `std` `attrs` |  | prop: parameterize AlwaysApplicableOrd | impl PartialOrd<[U]> for [T] |
| [#52454](https://github.com/rust-lang/rust/issues/52454) | open | repro | `both` `std` `const` |  | blocked: stable-specialization | Blanket impl of Into::into for From should… |
| [#45742](https://github.com/rust-lang/rust/issues/45742) | open | repro | `spec` `std` |  | blocked: intersection/lattice-specialization | Blanket impl of AsRef for Deref |
| [#125295](https://github.com/rust-lang/rust/issues/125295) | closed |  | `spec` `no-mcve` |  |  | [Specialization] Alternative Minimal… |
| [#70442](https://github.com/rust-lang/rust/issues/70442) | closed | fixed | `def-type` `spec` |  | #70535 | blank specializing impls don't "lock in"… |
| [#70419](https://github.com/rust-lang/rust/issues/70419) | closed | fixed | `def-impl` `spec` |  | #70535 | Instance::resolve doesn't have the exact same… |

## Performance, and the const-trait interaction

| # | state | status | tags | dupe of | fix | place |
|---|---|---|---|---|---|---|
| [#45431](https://github.com/rust-lang/rust/issues/45431) | open | repro | `min-spec` `std` |  | blocked: assoc-type-specialization-would-make-Vec-invariant | RawVec stores a capacity field even if… |
| [#73995](https://github.com/rust-lang/rust/issues/73995) | open | no-repro | `min-spec` `std` |  |  | SpecForElem for i16/u16 and other digits |
| [#67307](https://github.com/rust-lang/rust/issues/67307) | closed | fixed | `min-spec` `std` `no-mcve` |  | #71321 | RcFromIter and ArcFromIter have unused… |
| [#94313](https://github.com/rust-lang/rust/issues/94313) | open | repro | `spec` `marker` |  | blocked: lattice-specialization | Zero-length arrays are non-Copy |
| [#149752](https://github.com/rust-lang/rust/issues/149752) | open | repro | `min-spec` `std` |  | blocked: perfect-derive-or-safe-specialization | TrivialClone is not derived for generic types… |
| [#157754](https://github.com/rust-lang/rust/issues/157754) | closed | repro | `min-spec` `std` |  | needs enumerate-range::RangeIter-in-step_by | step_by() on new ranges is less optimized… |
| [#58659](https://github.com/rust-lang/rust/issues/58659) | open | repro | `both` `std` |  | blocked: specialization-stabilization | Document expected relationships between… |
| [#147130](https://github.com/rust-lang/rust/issues/147130) | open | repro | `both` `const` |  |  | cannot specialize const trait impl with… |
| [#148200](https://github.com/rust-lang/rust/issues/148200) | open | repro | `def-impl` `both` `const` |  | prop: constness-aware-Instance::try_resolve | min_specialization with const traits can call… |

## try_as_dyn

| # | state | status | tags | dupe of | fix | place |
|---|---|---|---|---|---|---|
| [#144361](https://github.com/rust-lang/rust/issues/144361) | open |  | `dyn` |  | prop: TypingMode::Reflection + TryAsDynCompatible | try_as_dyn |
| [#144361-Q1](https://github.com/rust-lang/rust/issues/144361) | open |  | `dyn` `no-mcve` |  |  | Where should these functions live? |
| [#144361-Q2](https://github.com/rust-lang/rust/issues/144361) | open | repro | `dyn` |  |  | try_as_dyn observes through RPIT |
| [#144361-Q3](https://github.com/rust-lang/rust/issues/144361) | open |  | `dyn` `no-mcve` |  | prop: try_as_dyn_static | A separate try_as_dyn_static? |
| [#144361-Q4](https://github.com/rust-lang/rust/issues/144361) | open |  | `dyn` `no-mcve` |  | prop: TypingMode::Reflection + is_fully_generic_for_reflection | Removing the T: 'static requirement |
| [#144361-Q5](https://github.com/rust-lang/rust/issues/144361) | open | repro | `dyn` |  |  | dyn/unsized self types are unsupported |
| [#144361-Q6](https://github.com/rust-lang/rust/issues/144361) | open | repro | `dyn` |  |  | try_as_dyn is a negative-reasoning primitive |
| [#144361-Q7](https://github.com/rust-lang/rust/issues/144361) | open |  | `dyn` `no-mcve` |  |  | dyn→dyn would launder safe abstractions |
| [#151440](https://github.com/rust-lang/rust/issues/151440) | open | repro | `dyn` |  | blocked: solver cycle handling | How should try_as_dyn handle inductive… |
| [#155496](https://github.com/rust-lang/rust/issues/155496) | open | repro | `dyn` |  |  | ValidationError in check_validity_requirement |
| [#152030](https://github.com/rust-lang/rust/issues/152030) | closed | fixed | `dyn` |  | #152120 | when coercing a too large array to… |

## Issues the labels missed

| # | state | status | tags | dupe of | fix | place |
|---|---|---|---|---|---|---|
| [#89948](https://github.com/rust-lang/rust/issues/89948) | open | repro | `min-spec` `std` `trustedlen` |  | blocked: always-applicable-impls | SpecExtend for TrustedLen is unsound |
| [#135103](https://github.com/rust-lang/rust/issues/135103) | open | fixed | `min-spec` `std` |  | #135104 | The implementation of InPlaceIterable for… |
| [#92488](https://github.com/rust-lang/rust/issues/92488) | open | fixed | `min-spec` `tra` |  | landed | no region-bound-pairs for HirId when… |
| [#155252](https://github.com/rust-lang/rust/issues/155252) | open | repro | `next-solver` |  |  | Got a scalar pair where a scalar… |
| [#130799](https://github.com/rust-lang/rust/issues/130799) | open | repro | `adt-const` (spec is only a workaround) |  | blocked: const-generic-exhaustiveness | [adt_const_params] consider to avoid using… |
| [#85731](https://github.com/rust-lang/rust/issues/85731) | open |  | `min-spec` `std` `trusted-step` `no-mcve` |  | blocked: min_specialization | trusted_step |
| [#88901](https://github.com/rust-lang/rust/issues/88901) | closed | fixed | `min-spec` `std` |  | #105102 | A Lifetime-generic Copy impl can allow fields… |
| [#122420](https://github.com/rust-lang/rust/issues/122420) | closed | fixed |  |  | #122461 | UB in <Range as Iterator>::advance_by… |
| [#50781](https://github.com/rust-lang/rust/issues/50781) | closed | fixed |  |  | #125380 | Trait objects can call nonexistent concrete… |
| [#107887](https://github.com/rust-lang/rust/issues/107887) | closed | repro |  |  | landed | project for trait object bound candidates is… |

## Bug reports filed only as tracking-issue comments

59 reports that never got an issue number, triaged as duplicate, not-a-bug, design discussion, or genuinely unreported.

59 reports, `U005`-`U317`, 2016-03-23 … 2025-03-24. None of them was ever given its own issue
number. `state` is recorded as `UNFILED` throughout (the containing tracking issue #31844 is open).

### Summary

#### Classification counts

| class | n | ids |
|---|---|---|
| `DUP` of a numbered issue | 18 | U005 U024 U025 U030 U094 U107 U117 U124 U144 U146 U148 U156 U163 U165 U279 U283 U315 U317 |
| `NOT-A-BUG` | 11 | U020 U050 U075 U077 U096 U131 U140 U169 U246 U254 U255 |
| `DESIGN` | 24 | U017 U026 U031 U032 U036 U040 U043 U045 U047 U048 U052 U076 U081 U088 U091 U179 U180 U181 U195 U200 U213 U277 U278 U285 |
| `REAL` | 6 | U151 U154 U273 U282 U287 U294 |

Of the 6 `REAL`: **3 still reproduce on nightly and are unfiled and unfixed** (U154, U273, U287),
**1 reproduces but is not specialization-specific** (U282), and **2 are stale-fixed** (U151, U294;
both were compiler stack overflows, both now produce clean errors).

#### Which rule the 29 `DUP`+`NOT-A-BUG` reporters actually hit

| rule they hit | n | ids |
|---|---|---|
| **R1** "concrete impl must be provably a *subset* of the blanket impl": blocked because the blanket's bound is a **foreign trait** and `Type: !ForeignTrait` is not provable (`note: upstream crates may add…`) | 7 | U005 U094 U096 U124 U146 U148 U156 |
| **R2** no **lattice rule**: the two impls overlap but neither is a subset of the other | 5 | U020 U050 U075 U165 U144(pt 1) |
| **R3** a `default type` is **not projectable** outside (or inside) the impl that declares it | 7 | U017 U025 U107 U279 U283 U315 U317 |
| **R4** downstream/orphan knowability once the trait has a **type parameter** | 1 | U140 |
| **R5** misc. (implicit `Sized`, auto-trait propagation, per-impl not per-method, edition) | 6 | U077 U169 U246 U254 U255 U131 |
| **R6** spec-graph query cycle | 1 | U117 |
| **R7** inference committed to the single specializing impl | 1 | U144(pt 2) |
| already self-filed by the reporter | 2 | U024 (#36587) U030 (#33162) |

**R1 is the single biggest recurring stumbling block** (7 of 59) and is the same problem as
#45542 / #81376 / #43048.

#### Requested theme tallies

**(a) `Display` / `From` / `Into` / `ToString` / `FromStr` specialization failing (6 reports)**
U005 (`FromStr`), U094 (`Display`), U096 (`Display`, control case), U124 (`From`), U156
(`ToString`), U163 (`From`+`Into`).
Two distinct causes hide under this one theme, and the reporters do not distinguish them:
- **five** of them (U005 U094 U096 U124 U156) fail for **R1**: the *user's own* blanket impl is
  `default`, but the concrete impl is not provably a subset of it because `(): !Display`,
  `i32: !Borrow<[i32]>`, `&str: !FromStr` etc. cannot be proven. Making the blanket impl `default`
  does not help and never could; the missing feature is negative reasoning, not `default`.
- **exactly one** (U163) is the "the blanket impl is not `default` and cannot be made so" problem
  proper: `core`'s `impl<T, U: From<T>> Into<U> for T` is not `default`, which is #52454.

**(b) `min_specialization` (or specialization) for inherent impls: 4 comments, 3 distinct askers,
0 numbered issues.**
U154 (@gnzlbg, 2019-08-28), U273 (@kalcutter, 2022-04-16), U277 (@Logarithmus, workaround), U278
(@zirconium-n, rebuts the workaround). I searched rust-lang/rust for `inherent impl
specialization`, `min_specialization inherent`, `specialize inherent methods`, `E0592
specialization`: **there is no numbered issue for this anywhere.** RFC 1210 has a section on it; it
has only ever been requested in this thread; it fails with E0592 identically under `specialization`
and `min_specialization`. MCVE written.

**(c) a `default fn` cannot rely on a sibling `default type` ("entangled defaults" / "default
groups"): 11 comments, spanning 2016-07 → 2023-02**
U025 (@tomaka, the original report), U026 (@Aatch), U031 (@nrc), U032 (@nikomatsakis, diagnosis +
`default { … }` proposal), U047 (@dtolnay), U088 (@withoutboats), U091 (@nikomatsakis, "the only
real solution"), U107 (@burns47), U181 (@RalfJung), U195 (@nikomatsakis, "we later backed off…and
ultimately didn't adopt any solution"), U285 (@bjorn3). Numbered issue: #52396 (+#85228, #150087).
This is the **most repeated design gap in the thread**, and the RFC's own two headline examples
(`Example::generate` and `default impl Add`) are both instances of it that have never compiled.

**(d) specialization not firing / no way to observe which impl ran (5 comments)**
U144 (pt 2, inference commits to the specializing impl), U179 + U180 (a `cfg(test)` flag / a
`thread_local!` Cell as the only way to assert a specialized impl fired), U200 ("quite hard to
figure out which impls are specializing which"), U213 ("impl chasing"). Nobody in the thread ever
proposed a language-level answer; the two concrete answers are both test-harness hacks.

**(e) @Aandreba 2023-05-12 (U294), the 54KB compile-time-Fibonacci crash, FIXED: does not
reproduce.** It was a real SIGSEGV: the backtrace is an unbounded recursion through
`rustc_infer::infer::combine::ConstInferUnifier::try_fold_const`, a function that no longer exists.
On nightly 1.98.0 the same program produces two clean `E0275 overflow evaluating the requirement`
errors with a recursion-limit help. It matches **no numbered ICE issue** in the set; it is not a
`translate_substs` / "expected specialization failed to hold" / `has_infer` failure, and the
recursion is in the `generic_const_exprs` where-clause chain, not in specialization; `default const`
only makes the impls coherent enough to reach it. MCVE written recording the fix.

#### Cross-cutting notes

- **Nothing in this thread is a soundness report.** Across 59 comments and nine years, not one
  unfiled report is an unsoundness. Every unsoundness in the survey came in through a numbered issue.
- **Two unfiled compiler crashes (U151, U294) were both fixed without anyone filing them**, i.e. by
  unrelated compiler work. Both were reported as questions ("should this not cause a stack
  overflow?"), which is probably why neither was triaged.
- The design half of the thread converges on **three** wanted features that were each asked for
  independently many times: the lattice rule (U020 U031 U040 U045 U050 U075 U165), negative
  reasoning / negative supertrait bounds (U005 U031 U040 U043 U045 U094 U124 U146 U148 U156), and
  default groups (theme (c)). None landed.

### The 6 REAL reports

| # | status | who | what |
|---|---|---|---|
| U154 | `repro` | @gnzlbg 2019-08-28 | `default fn` / `default const fn` in an inherent impl is silently accepted under `specialization` and means nothing |
| U273 | `repro` | @kalcutter 2022-04-16 | asks for inherent-impl specialization (E0592); no numbered issue exists for this anywhere |
| U287 | `repro` | @clarfonthey 2023-02-17 | `error: cannot specialize on trait `Sync`` has no note, no error code and no reference to the rustc_specialization_trait rule |
| U282 | `repro` | @drdozer 2022-12-05 | an explicit negative impl stops ruling out overlap once the trait gains a type parameter (downstream-knowable check ignores it) |
| U151 | `fixed` | @landaire 2019-06-27 | `default impl` for `[T; 1]` stack-overflowed rustc in 2019; now compiles and prints [0] |
| U294 | `fixed` | @Aandreba 2023-05-12 | 54KB SIGSEGV backtrace looping in ConstInferUnifier::try_fold_const; now two clean E0275s |
