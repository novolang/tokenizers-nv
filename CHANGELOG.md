# Changelog

All notable changes to tokenizers-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `tokmodel` — `TokModel`, the three models behind one type, and the
  `encode`/`decode` family over it. `TokEncoding` carries ids, tokens,
  offsets, lengths, type ids and an attention mask, because building
  them all costs one pass and asking for them separately would cost
  four.
- `tokvocab` — `TokVocab` as a list AND a sorted index, so token-to-id
  is a binary search rather than the scan a bare `[Str]` forces.
  `of_lines`, `of_json_object` and `of_tokens` take values, never paths.
- `tokbpe` — byte-level BPE with the byte-to-unicode map BUILT FROM THE
  RULE rather than read from anywhere, merges ranked, and the
  pre-tokenizer patterns written out as `TokPretokenRule` variants
  instead of compiled from a regex.
- `tokunigram` — SentencePiece unigram: Viterbi over piece log
  probabilities, the `▁` whitespace substitution, and `byte_fallback` as
  a field because it is what separates a lossless model from a lossy one.
- `tokpiece` — WordPiece generalised: the continuation prefix, the unk
  id, the maximum word length and the four normalisation rules are all
  fields, and nothing wraps a sequence unless asked.
- `tokspecial` — the six named roles as fields rather than a list, and
  `TokTemplate` as the caller's choice rather than the tokenizer's habit.
- `tokfault` — seven variants, all of them about loading or decoding.
  Encoding has no failure mode and no `Result`.

### Decided

- **An enum, not a trait, and the deciding reason is that the kind is
  data.** Which of the three a program holds is read out of
  `tokenizer.json` at run time; a trait bound is resolved at compile
  time, and expressing the run-time choice needs a `dyn Trait` a `core`
  package cannot afford. SPEC §3.6 (a bound carries an effect argument,
  never a type argument) and the cross-module generic emission limit
  each independently rule out the alternatives.
- **`compiler/stdlib/wordpiece.nv` is neither wrapped nor replaced; it
  coexists.** Wrapping would inherit its ASCII-only rule, which is a
  wrong answer rather than a missing feature, and replacing a
  auto-prepended stdlib type is not a package's decision. The README
  carries the table.
- **Encoding is total.** Every model here has an answer for every input,
  so a caller never handles a failure from the step before the model.
- **The pre-tokenizer patterns are named rules, not a regex.** Keeps the
  dependency list empty and makes each pattern testable on its own.

### Known

- **The stdlib page for `WordPiece` should point here for the general
  case.** That edit belongs to the lane that owns the stdlib docs and is
  recorded here so it is not forgotten.
- The exact-id test vectors need the real vocabulary files, which this
  suite does not ship. The cases that need no vocabulary — the byte map,
  the `▁` rules, the Viterbi arithmetic, the merge-by-rank rule — assert
  exact answers; the rest assert shape, and the implementation lane adds
  the ids when it has the files.
- No `@tier(embedded)` claim. A vocabulary is tens of thousands of
  strings, so the memory rather than the arithmetic is what a device
  would have to answer for.
