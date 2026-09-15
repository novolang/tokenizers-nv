# tokenizers-nv

A tokenizer turns text into the list of integers a language model reads,
and turns that list back into text. The three algorithms almost every model
uses are byte-level **BPE**, SentencePiece **unigram** and **WordPiece**.
This package brings all three to novo-lang behind one type, following
HuggingFace's [tokenizers](https://github.com/huggingface/tokenizers) for
the API shape and the `tokenizer.json` format, and
[sentencepiece](https://github.com/google/sentencepiece) for the unigram
model. It has no dependencies.
[prompt-nv](https://novo-lang.org/packages/prompt-nv) and
[llm-client-nv](https://novo-lang.org/packages/llm-client-nv) take the
token count this package gives.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

A **vocabulary** is the list of pieces a model knows, each with an
**id**, its position in the list. **Encoding** is turning text into ids and
**decoding** is turning ids back into text.

Before any of the three algorithms runs, the text is cut into words. That
step is called **pre-tokenization** for BPE and **normalization** for
WordPiece, and it is part of the model rather than a preliminary: the same
text through a different rule gives different ids.

**Byte-level BPE** starts from the bytes of the text and repeatedly merges
the adjacent pair with the lowest **rank** in a merges table, until no pair
in the word can be merged. It is byte-level because every byte is first
mapped to a printable character, so any input encodes and every encoding
decodes back exactly.

**SentencePiece unigram** holds a piece for each entry with a score, and
picks the segmentation of the text whose scores sum highest. Whitespace
becomes a visible character, so a segmentation can be turned back into text
without knowing where the spaces were.

**WordPiece** takes one word at a time and repeatedly takes the longest
piece in the vocabulary that the rest of the word starts with. Every piece
after the first carries a **continuation prefix**, `##` for BERT.

**Special tokens** are ids with a role rather than a meaning: unknown,
beginning of sequence, end of sequence, padding, separator and mask. A
**template** says which of them wrap a sequence.

A vocabulary here is a value the caller holds, not a path. `std.hf`
downloads the file, the caller reads it, and this package parses the text.
The same tokenizer therefore runs in a browser with the vocabulary fetched
over HTTP, in a server with it built into the binary, and beside a model on
a device with it in flash.

| Model | An example family | Vocabulary source |
| --- | --- | --- |
| `TokBpeModel` | GPT-2, Llama, Qwen | `vocab.json` and `merges.txt`, or `tokenizer.json` |
| `TokUnigramModel` | T5, ALBERT, Gemma | the pieces and scores in `tokenizer.json` |
| `TokWordPieceModel` | BERT and its descendants | `vocab.txt`, or `tokenizer.json` |

The pre-tokenizer rules, which differ between families that otherwise use
the same algorithm:

| Rule | Contractions | Longest digit run |
| --- | --- | --- |
| `TokPretokenGpt2` | case-sensitive | unbounded |
| `TokPretokenLlama3` | case-insensitive | 3 |
| `TokPretokenQwen2` | case-insensitive | 1 |
| `TokPretokenWhitespace` | — | unbounded |
| `TokPretokenNone` | — | unbounded |

The four sequence templates:

| Template | Ids added |
| --- | --- |
| `TokTemplateNone` | none |
| `TokTemplateSingle` | beginning at the front, end at the back |
| `TokTemplatePair` | beginning, the first sequence, separator, the second, end |
| `TokTemplateBosOnly` | beginning only |

Every function here is byte arithmetic and table lookup. Nothing opens a
file, reaches the network or reads a clock, and there is not one effect row
in the package.

## Install

```
novo pkg add tokenizers-nv
```

## Example

```novo
use tokmodel
use std.list

// Turn text into the ids a model reads. The tokenizer file's text is
// the caller's: this package opens nothing.
fn prompt_ids(tokenizer_json: Str, text: Str) -> Result<[Int], TokFault>
    // Which of the three models this is comes out of the file's own
    // `model.type` field, not from a choice made here.
    let m = tokmodel.of_tokenizer_json(tokenizer_json)!

    // The special tokens the same file declares: the unknown, the two
    // sequence markers, the padding and the rest.
    let s = tokmodel.specials_of_tokenizer_json(tokenizer_json)!

    // Encoding is total, so there is no failure to handle here. The
    // template puts a beginning-of-sequence id in front and nothing
    // after it, which is what a chat prompt wants.
    Ok(tokmodel.encode_with(m, text, s, TokTemplateBosOnly).ids)

fn main() [io]
    match prompt_ids("{}", "hello world")
        Ok(ids) => println("${list.len(ids)} token(s)")
        Err(f)  => println(f.message())
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: tokenizers-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `tokvocab` | The vocabulary: both lookup directions, added tokens, and the longest-prefix search the models share. |
| `tokbpe` | Byte-level BPE: the merges table, the five pre-tokenizer rules, the byte-to-character map, and encode and decode. |
| `tokunigram` | SentencePiece unigram: the pieces and their scores, the whitespace substitution, the best segmentation, and encode and decode. |
| `tokpiece` | WordPiece: the continuation prefix, the four normalisation rules, the whole-word piecing, and encode and decode. |
| `tokspecial` | The six special-token roles as named fields, the four templates, and the readers that find them in a file. |
| `tokmodel` | The three models behind one enum, the encoding value they all answer, and the batch, pad, truncate and streaming-decode calls. |
| `tokfault` | The seven ways loading or decoding fails, each naming the place in the input. |

## How to choose an entry point

**`tokmodel.of_tokenizer_json` is the call most programs make.** It reads
`model.type` out of the file and builds whichever of the three it names.
`specials_of_tokenizer_json` reads the special tokens from the same text.

**The per-model constructors take the parts.** `tokbpe.of_gpt2_files` takes
a `vocab.json` and a `merges.txt`. `tokpiece.of_vocab_text` takes a
`vocab.txt`. `tokunigram.of_json_pieces` takes the pieces array. Use them
for a checkpoint that has no `tokenizer.json`.

**`tokmodel.encode` adds nothing and `encode_with` applies a template.**
Take the first for a generation call, where the model is continuing text.
Take the second for a sentence encoder. `encode_pair` is the two-sequence
form and `encode_batch` is the list form.

**`tokmodel.decode` is the whole list and `decode_stream` is a generation
loop.** The second takes the tail of a multi-byte character left over from
the previous call and hands back the new one, so a loop printing tokens as
they arrive never prints half a character.

**The per-model modules are usable directly.** `tokbpe.tokenize`,
`tokunigram.best_path` and `tokpiece.piece_word` are the pieces rather than
the ids, for inspection and for a caller building something else.

## The rules a user needs

1. **BPE merges by rank, not by length.** The lowest-ranked adjacent pair
   in the word merges first, and then the next. Merging by length produces
   plausible tokens that are not the model's tokens, and the model reads
   them as different words. `tokbpe.rank_of` is the table lookup.
2. **The pre-tokenizer is part of the model.** GPT-2, Llama 3 and Qwen2
   split text differently, as the table above shows, and the same string
   through the wrong rule gives different ids. The rule travels with the
   merges in `TokBpe` for that reason.
3. **WordPiece is all or nothing on a word.** A word that matches for three
   pieces and then cannot be continued produces one unknown token, not
   those three pieces and an unknown. `tokpiece.piece_word` therefore takes
   a whole word, and an implementation that emitted as it went could not
   express it.
4. **The byte-level map must be a bijection over all 256 byte values.**
   That is what makes every input encodable and every encoding exactly
   reversible. `tokbpe.byte_map` is built from the rule rather than
   transcribed, and `TokBadByteMap` is what a broken one answers.
5. **Encoding never fails.** BPE falls back to single bytes, unigram to
   bytes or its unknown piece, and WordPiece to its unknown token, so
   `encode` answers a `TokEncoding` and not a `Result`. What fails is
   loading a file that is not what it claims, and decoding an id no
   vocabulary has.
6. **An id outside the vocabulary means the ids and the vocabulary came
   from different checkpoints.** `TokIdOutOfRange` carries the id and the
   size. `tokfault.is_load_error` is false for it, which is the
   distinction a caller acts on: a load failure means choose another file.
7. **A unigram piece with no score is refused.** Segmentation maximises
   over scores, so a missing one is not something to default.
8. **`byte_fallback` is what makes a unigram model lossless.**
   `tokmodel.is_lossless` answers whether text survives a round trip
   through this model, and a unigram model without byte fallback replaces
   anything it cannot represent with its unknown piece.
9. **Normalisation is four independent decisions.** Lower-casing,
   accent-stripping, punctuation splitting and CJK splitting are separate
   fields, because models mix them. `tokpiece.bert_uncased_rules` and
   `bert_cased_rules` are the two common combinations.
10. **The template is the caller's choice.** Nothing here wraps a sequence
    unless asked. A beginning-of-sequence id in the middle of a
    continuation is wrong, and a sentence encoder that did not get its
    wrapping produces embeddings that are subtly not the model's.
11. **`TokSpecials` fields are −1 when the model has none.** A byte-level
    BPE never needs an unknown token, and `tokspecial.none` is the honest
    default for a vocabulary that has not been read.
12. **Added tokens are matched before the model's own vocabulary.** That is
    what makes a control token added after training tokenize as one id
    rather than being split into pieces. `tokvocab.match_added` is the
    check.
13. **A `TokEncoding` carries six lists, built in one pass.** The ids, the
    token strings, the byte offset of each token in the original text, each
    token's length there, the sequence each came from, and the attention
    mask. A token the template added has a length of zero, which is how a
    caller tells one apart without comparing ids.
14. **`padded` and `truncated` are separate calls on the encoding.**
    Padding sets the attention mask to zero for what it added; truncation
    keeps the template's ids.

## What is not included

- **A file system.** Every constructor takes text or values. `std.hf` and
  `std.fs` are how a caller gets them.
- **A regular expression engine.** The pre-tokenizer patterns are written
  out as named rules, which keeps the dependency list empty and each
  pattern testable on its own.
- **A fourth algorithm.** The three here are what the reference
  implementation supports. A character tokenizer or a regex one would be a
  fourth variant of the enum and a branch in six functions.
- **Training.** Nothing here learns a vocabulary or a merges table.
- **A chat template.** Turning a conversation into one string is
  [prompt-nv](https://novo-lang.org/packages/prompt-nv)'s work.
- **A microcontroller build.** No module is declared to build for a device
  with no heap allocator, though the vocabulary-as-a-value shape is what
  such a build would need.

## Related packages

- [prompt-nv](https://novo-lang.org/packages/prompt-nv) takes a token
  counter as a parameter, and `tokmodel.token_count` is the function that
  fits it.
- [llm-client-nv](https://novo-lang.org/packages/llm-client-nv) uses that
  count to refuse an over-long request before the round trip.
- [gguf-nv](https://novo-lang.org/packages/gguf-nv) reads a model file's
  `tokenizer.ggml.*` metadata, which is where a GGUF keeps its vocabulary,
  its merges and its token types.
- [embeddings-nv](https://novo-lang.org/packages/embeddings-nv) takes the
  attention mask this package produces, and pools the matrix a model
  returns for the ids.
- `std.wordpiece` in the standard library is a BERT-uncased WordPiece
  tokenizer, and it coexists with this package. It is the shorter path for
  a demo with no dependency. This package supersedes it for anything new:
  `std.wordpiece` is ASCII-only, so an accented word tokenizes as unknown;
  it lower-cases unconditionally; its continuation prefix is fixed at `##`;
  its unknown id is whatever `[UNK]` has in the vocabulary, and is −1
  when the vocabulary has no `[UNK]` at all; its
  token-to-id lookup is a linear scan; and its encode always wraps in
  `[CLS]` and `[SEP]`. This package makes each of those a field or a
  choice, and its vocabulary lookup is a binary search.
- `std.hf` in the standard library fetches repository files from the
  HuggingFace Hub into a revision-pinned cache. That is where a
  `tokenizer.json` comes from.

## Tests

```bash
novo test tests/tokenizer_tests.nv    # 32 tests: the three models and the vocabulary
```

The expected values are the reference implementations' own. GPT-2's
`"Hello, world!"` is six tokens with the ids OpenAI's tokenizer gives, and
the byte-to-character map's fixed points are from the published table. The
SentencePiece cases are the reference's own round-trip contract, with the
best-path case worked by hand from three pieces and three scores so a
reader can check the arithmetic. BERT's `unaffable` becoming
`un ##aff ##able` is the example in the original paper.

The tests compile today and fail at run, each on the
`not implemented: tokenizers-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `tokvocab.of_tokens`, `.of_lines`, `.of_json_object`, `.with_added`, `.added` | no |
| `tokvocab.size`, `.id_of`, `.token_of`, `.has`, `.tokens_of`, `.is_dense` | no |
| `tokvocab.longest_prefix`, `.match_added` | no |
| `tokbpe.of_parts`, `.of_gpt2_files`, `.merge`, `.with_prefix_space`, `.merge_count`, `.rank_of` | no |
| `tokbpe.byte_map`, `.bytes_to_alphabet`, `.alphabet_to_bytes` | no |
| `tokbpe.pretokenize`, `.merge_word`, `.encode`, `.tokenize`, `.decode` | no |
| `tokunigram.of_pieces`, `.of_json_pieces`, `.piece`, `.with_byte_fallback`, `.with_dummy_prefix`, `.score_of` | no |
| `tokunigram.to_sentencepiece_space`, `.from_sentencepiece_space` | no |
| `tokunigram.best_path`, `.best_score`, `.encode`, `.tokenize`, `.decode` | no |
| `tokpiece.of_vocab`, `.of_vocab_text`, `.with_continuation`, `.with_normalizer`, `.with_max_word_len` | no |
| `tokpiece.bert_uncased_rules`, `.bert_cased_rules`, `.no_normalization` | no |
| `tokpiece.normalize`, `.split_words`, `.piece_word`, `.encode`, `.tokenize`, `.decode` | no |
| `tokspecial.none`, `.of_names`, `.bert_names`, `.gpt2_names`, `.with_eos`, `.is_special` | no |
| `tokspecial.of_added_tokens_json`, `.added_tokens_of_json` | no |
| `tokspecial.template_ids`, `.pair_separator_ids` | no |
| `tokmodel.of_tokenizer_json`, `.specials_of_tokenizer_json` | no |
| `tokmodel.model_name`, `.vocab_of`, `.vocab_size`, `.is_lossless`, `.empty_encoding` | no |
| `tokmodel.encode`, `.encode_with`, `.encode_pair`, `.encode_batch`, `.tokenize`, `.token_count` | no |
| `tokmodel.padded`, `.truncated` | no |
| `tokmodel.decode`, `.decode_skipping_specials`, `.decode_one`, `.decode_stream` | no |
| `tokfault.is_load_error`, `TokFault.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
