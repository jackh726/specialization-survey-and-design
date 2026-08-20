# Specialization survey and design

A survey of Rust's `specialization` / `min_specialization` features: what people use them for
today, what they wish they could use them for, and what stands in the way.

| Doc | What it is |
| --- | --- |
| [USE-CASES.md](USE-CASES.md) | The narrative catalogue: a dozen or so distinct idioms specialization is used for in shipped code, one real example each. |
| [USE-CASE-INDEX.md](USE-CASE-INDEX.md) | The per-crate data behind USE-CASES.md — one row per crate, classified by motivation, overlap, and flags. |
| [DESIRED-USE-CASES.md](DESIRED-USE-CASES.md) | Things people *want* to do that no shipped code expresses, and why each is absent. |
| [DESIRED-EXTENSIONS.md](DESIRED-EXTENSIONS.md) | The design-space companion: the feature extensions those desired uses would need. |
| [ISSUES.md](ISSUES.md) | Triage of known issues across `specialization`, `min_specialization`, and `try_as_dyn`. |
