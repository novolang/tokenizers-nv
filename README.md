# tokenizers-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

The tokenizer family behind one type: **byte-level BPE** as GPT-2 does
it, **SentencePiece unigram**, and **WordPiece** — with encode to ids,
decode from ids, special-token handling, and a vocabulary loaded from a
value the caller holds rather than from a file.

```
novo pkg add tokenizers-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use tokmodel
use tokspecial

// std.hf downloaded tokenizer.json; the caller read it; this package
// does the rest.  Which of the three models it turns out to be is read
// out of the file, not chosen here.
fn prompt_ids(tokenizer_json: Str, text: Str) -> Result<[Int], TokFault>
    let m = tokmodel.of_tokenizer_json(tokenizer_json)!
    let s = tokmodel.specials_of_tokenizer_json(tokenizer_json)!
    Ok(tokmodel.encode_with(m, text, s, TokTemplateBosOnly).ids)
```

## The layer, and why

`core`, and for a tokenizer that is a real constraint rather than an
easy one — **every reference implementation opens a path**:
`Tokenizer.from_file`, `sp.Load("m.model")`,
`BertTokenizer.from_pretrained`.

A vocabulary here is a **value**. `std.hf` downloads the file, the
caller reads it, and this package parses the text. What that buys is not
only the budget: the same tokenizer then runs in a wasm notebook with the
vocabulary fetched over HTTP, in a server with it embedded in the binary,
and beside a model on a device with it in flash — three deployments that a
`from_file` API serves in one.

No `[dependencies]` either. The reference reaches for a regex engine for
its pre-tokenizer patterns; this package writes those patterns out as
named `TokPretokenRule` variants instead, which keeps the list empty and
the behaviour inspectable.

## The load-bearing interface

```novo ignore
pub enum TokModel                      // the kind is DATA, not a type parameter
    TokBpeModel(bpe: TokBpe)
    TokUnigramModel(unigram: TokUnigram)
    TokWordPieceModel(wordpiece: TokWordPiece)

pub fn of_tokenizer_json(text: Str) -> Result<TokModel, TokFault>
pub fn encode(m: TokModel, text: Str) -> TokEncoding          // total — no Result
pub fn decode(m: TokModel, ids: [Int]) -> Result<Str, TokFault>
```

**An enum, not a trait**, and the deciding reason is that the kind is
data. A `tokenizer.json` says `"model": {"type": "BPE"}`, and which of
the three a program is holding is read out of that file at run time. A
trait bound — `fn encode<T: Tokenizer>(t: T, …)` — is resolved at
*compile* time, so it cannot express "whatever this file turns out to
say". Expressing that needs a trait *object*, and a `dyn Trait` is a
boxed dispatch a `core` package cannot afford. An enum makes the run-time
choice a `match`, which is exactly what it is.

Two more reasons, either of which would have been enough on its own:

- SPEC §3.6's grammar is `trait_bound ::= TYPE_IDENT [ '[' ident ']' ]` —
  a bound carries an **effect** argument and never a **type** argument.
  proptest-core-nv found this and reshaped around it. Anything generic
  over the vocabulary representation as well as the model could not be
  spelled at all.
- A generic function whose return type applies a generic head is not
  emitted for a cross-module call on this toolchain, and a library is
  nothing but cross-module calls.

What the enum costs is that a fourth model — a character tokenizer, a
regex one — is a variant here and a branch in six functions, rather than
an impl somebody writes in their own package. For a family whose
membership is set by what the reference supports, that is the right
trade.

**Encoding is total.** There is no `Result` on `encode`, because every
model here has an answer for every input: BPE falls back to single bytes,
unigram to bytes or its unk piece, WordPiece to its unk token. A caller
feeding a chat message into a model should not have to handle a failure
from the step before the model. What fails is *loading* — a file that is
not what it claims — and *decoding* an id no vocabulary has, which means
the ids and the vocabulary came from different checkpoints.

## `compiler/stdlib/wordpiece.nv`: coexist, and supersede

The stdlib already has a `WordPiece`. This package **neither wraps nor
replaces it. It coexists, and it is the one to use for anything new.**

**Why not wrap it.** `std.wordpiece` is BERT-uncased and nothing else: it
lowercases unconditionally, it is **ASCII-only** so every accented word
tokenises as unknown, its continuation prefix is `##` with no way to
change it, its `unk_id` defaults to 100 and only then looks up `[UNK]` —
so a vocabulary that puts it elsewhere and has fewer than 101 entries
gets a wrong id with no complaint — its `id_of` is a **linear scan** of
the vocabulary, and its `encode` wraps in `[CLS]` and `[SEP]` whether the
caller wanted that or not. A package that wrapped it would inherit every
one of those. The ASCII rule alone is decisive: it is not a missing
feature, it is a wrong answer, and a wrapper cannot be correct where what
it wraps is wrong.

**Why not replace it.** Replacing a stdlib module is not a package's
decision to make, and in this case it is not even mechanically open:
`WordPiece` is **auto-prepended to every program that mentions the name**,
with no `use` and no version. Removing it would break programs that never
imported anything.

**So they coexist**, and the division is clean:

| | `std.wordpiece` | `tokenizers-nv` |
| --- | --- | --- |
| for | a BERT-uncased demo in four lines, with no dependency | a program that loads a real checkpoint |
| vocabulary | `[Str]`, linear scan | `TokVocab`, binary search |
| text | ASCII only | normalisation as four named rules |
| prefix / unk | `##` / 100 | fields, read from the vocabulary |
| wrapping | always `[CLS] … [SEP]` | a `TokTemplate` the caller picks |
| family | WordPiece only | three models, one `TokEncoding` |

The stdlib page for `WordPiece` should point here for the general case;
that edit belongs to the lane that owns the stdlib docs, and it is named
in this package's changelog so it is not forgotten.

## What it ports

[tokenizers](https://github.com/huggingface/tokenizers) for the API shape
and `tokenizer.json` for the format;
[sentencepiece](https://github.com/google/sentencepiece) for the unigram
model. Both are permissive, and both carry the test vectors the
implementation lane should measure against: GPT-2's published byte map
and its encoder's own examples, the SentencePiece test model's expected
ids, and BERT's `unaffable → un ##aff ##able`.

### Three things the reference gets right that are easy to get wrong

- **BPE merges by RANK, not by length.** The lowest-ranked adjacent pair
  in the word merges first, repeatedly. Getting that wrong produces
  plausible tokens that are not the model's tokens, and the model then
  reads them as different words.
- **The pre-tokenizer is part of the model.** GPT-2's, Llama 3's and
  Qwen2's patterns differ — Llama 3 caps a digit run at three, Qwen2 at
  one — and running text through the wrong one gives different ids for
  the same string.
- **WordPiece is all-or-nothing on a word.** A word that matches for
  three pieces and then cannot be continued produces **one** unk token,
  not those three pieces and an unk. An implementation that emits as it
  goes cannot express that, which is why `piece_word` is a function over
  a whole word.

## What this removes from `orbit/ml`

`orbit/ml` already has a byte-level BPE, and it works — with a cost
this package is shaped to remove. Because GPT-2's byte-to-unicode map is
a bijection, BPE over the mapped code points is equivalent to BPE over
raw bytes, so ml skips the map and works in a hex byte space instead. The
price is a Python script (`tools/gen_tokenizer_files.py`) that has to be
run once per checkpoint, three files a fresh HuggingFace clone does not
have, and a failure mode where an unconverted checkpoint loads an empty
vocabulary and the model produces nonsense.

This package does the map natively and reads `tokenizer.json` as it
comes, so that preparation step disappears. It also covers the full
pre-tokenizer patterns rather than ml's ASCII subset, and it brings the
other two models with it.

## Related

- [`gguf-nv`](https://github.com/novolang/gguf-nv) — the model file, also
  split out of `orbit/ml`
- [Publishing a package to Orbit](https://novo-lang.org/publishing) —
  the layer rules this package is held to
