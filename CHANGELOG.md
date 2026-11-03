# Changelog

All notable changes to tokenizers-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-11-03

This release requires novo 0.23.0 or later.

## 0.1.0 — 2026-10-29

The first release with bodies, and `stability = "experimental"`.  Every function in the interface now does what its comment says.

### Added

- Byte-level BPE with the GPT-2, Llama 3 (cl100k), Qwen2 and white-space pre-tokenizer rules, added tokens, `add_prefix_space`, and `of_gpt2_files`.
- SentencePiece unigram by Viterbi over 32-bit scores, with byte fallback, the dummy prefix, and one unknown piece for a run of uncovered characters.
- WordPiece with BERT's normalisation done one character at a time, and offsets into the original text.
- `tokmodel.of_tokenizer_json` for BPE, Unigram and WordPiece models, including `added_tokens`, the byte-level pre-tokenizer variants and the options of a WordPiece or Unigram model.
- `tokutf8`, the code-point reader and character classes, with `tokvocab.split_on_added`, `tokbpe.pieces_with_spans`, `tokbpe.pretokenize_spans`, `tokpiece.pieces_with_spans` and `tokunigram.pieces_with_spans`, which the models share.

### Changed

- `unicode-nv` ^0.1.4 is now a dependency.  It supplies `\p{L}` and `\p{N}`, canonical decomposition and lowercase mapping.
- `TokUnigram` has a new field, `min_score`, the lowest piece score, from which a character no piece spells is scored.
- `TokPretokenLlama3` and `TokPretokenQwen2` follow the cl100k pattern: a word may follow one character that is not a letter, a number or a line break, symbols take the line breaks after them, and white space that ends in a line break is one piece.  Their comments said only that the contractions and the digit run differ.
- `tokpiece.normalize` collapses white space and works on one character at a time, so the lowercase of a capital sigma is always `σ`.
- Every added token is matched in the input, special or not; `special` decides only whether `decode_skipping_specials` leaves it out.
- The parameter of `tokunigram.of_json_pieces` and of the two `tokspecial` readers that takes JSON text is named for what it holds, because `json` named the module.
- A pre-tokenizer pattern for GPT-2 splits `"a    b"` into `a`, three spaces and ` b`, as the reference's regular expression does; the earlier test expected two pieces.

### Measured

- Line coverage of `src/` is 100%.
- Against tiktoken and HuggingFace tokenizers on the cl100k_base vocabulary: the ids of 1,227 texts agree.
- Against sentencepiece 4 unigram models (byte fallback and dummy prefix on and off): the ids of 2,436 texts agree.
- Against HuggingFace tokenizers' WordPiece, cased and uncased: the ids of 1,232 texts agree.
- Against the `regex` module on the three pre-tokenizer patterns: 9,096 splits agree.

### Known differences from the references

- A Unigram `tokenizer.json` names normalizers (NFKC, a space-collapsing replace, prepending) that this package does not apply, so its ids differ from HuggingFace tokenizers' unless the text is normalized first.  SentencePiece itself, with its normalization turned off, agrees.
- With `add_dummy_prefix` on, a text that already starts with a space gets two marks, as SentencePiece gives it; HuggingFace's Metaspace adds one.
- Offsets are in bytes, and a token that is part of a multi-byte character covers only its bytes.

## 0.0.3 — 2026-10-25

A file that names a sibling module's type, function or constant now has the `use` line for that module (E2033, E2003).  No signature and no behaviour changed.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

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
