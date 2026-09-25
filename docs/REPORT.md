# Tibetan layer detection: data preparation, training and evaluation

Draft report. It summarises the three layer repositories (tsawa, sabche, chapter) and the zero-shot
LLM comparison in this repository, so one document shows how the pieces fit. It is meant to be edited
by the team. Every number below comes from the repositories or the Hugging Face model and dataset
cards listed in section 9.

## 1. What this project does

Digitised Tibetan Buddhist books from OpenPecha carry annotation layers. We look at three of them, and
for each one we train a separate model that marks the layer in a whole book:

| Layer | What it marks |
|---|---|
| **Tsawa** (རྩ་བ) | root text that a commentary quotes piece by piece and then explains |
| **Sabche** (ས་བཅད) | the outline headings a commentary gives to each section it is about to explain |
| **Chapter** (ལེའུ་) | the heading line that opens a division of a book: a chapter, a work in a collected volume, a section, or front matter |

Each model is `jhu-clsp/mmBERT-base` fine-tuned for BIO token classification. We also test how far a
zero-shot LLM prompt (Claude Sonnet 5 and Gemini 3.1 Flash Lite) gets on the same books with no training.

## 2. Data

Source: 539 OpenPecha `.opf` books in two annotation batches (266 old, 273 new). The raw annotations are
never edited. Every fix is written to a sidecar CSV, so the cleaning can be reviewed or undone.

| | Tsawa | Sabche | Chapter |
|---|---|---|---|
| Books with the layer | 212 (89 old, 123 new) | 333 (138 old, 195 new) | 394 (159 old, 235 new) |
| Raw spans | 21,155 | 43,223 | 3,252 |
| Books kept | 124 | 323 | 378 |
| Spans kept | 17,999 in the 124 kept books (18,289 across all 212 after merging), 17,563 after masking | 42,292 | 3,042 |
| Median span length | 96 characters | 40 characters | 43 characters |
| Share of tokens | about 4 to 5.5% | about 4% | about 0.18% |

## 3. Data preparation

The steps are the same in outline for every layer: audit the raw annotation, fix what can be fixed
without changing meaning, drop books that cannot be trusted, then split by book. The decisions differ
because the layers behave differently.

| Step | Tsawa | Sabche | Chapter |
|---|---|---|---|
| Edge snapping | 8,513 of 21,155 spans moved to a syllable boundary (at most 12 characters) | 1,359 old and 352 new spans (at most 3 characters) | 316 spans (at most 3 characters) |
| Overlaps inside a book | 17 fixed, 6 stubs dropped | 1 span trimmed after snapping | none reported |
| Merging neighbouring spans | yes: 806 merges (2,860 spans absorbed), following the team decision on split root text | **no**: the new batch has one heading per line, so neighbours are separate headings | only on the same line: 5 merges. A first version merged across lines and fused headings, so it was dropped |
| Drifted offsets | none reported | 30 old-batch books re-aligned (4,173 spans shifted), 227 unverified spans dropped | 15 old-batch books whose offsets drift could not be recovered, so they were left out |
| Books excluded | 88 with fewer than 10 spans | 10 (3 unrecoverable, 2 without headings, 5 with only the outline annotated) | 16 (1 unfixable edge, 15 drifting) |
| Masked from the loss | 436 quotation-like spans | none | none |
| Short spans | kept | kept | kept (306 raw spans under 15 characters) |

**Splits.** All splits are by book, and the test split is frozen: it is scored once at the end and never
used for tuning.

| | Tsawa | Sabche | Chapter |
|---|---|---|---|
| Method | greedy, seed 123, 76/12/12 by windows, books that share a title or root text kept together | 83/8.5/8.5 by windows, built on the tsawa split with shared-text books grouped | 83/8.5/8.5 by windows, stratified on window share, spans per window, short-span share and old-batch share |
| Books train / val / test | 84 / 18 / 22 | 261 / 33 / 29 | 308 / 34 / 36 |
| Gold spans train / val / test | 13,129 / 1,700 / 2,734 | 35,122 / 3,091 / 4,079 | 2,509 / 275 / 258 |

The tsawa split is the base for the other two. The sabche split keeps its books' tsawa assignments except
three moved by conflict resolution, and the chapter split keeps 97 books from it, so most books keep the
same split across layers.

**Leakage** (long spans of 40 or more characters that also appear word for word in a train book). Tsawa
validation 3.3%, test 1.1%. Sabche validation 1.6%, test 2.5% (about 6.8% of test spans overlap one or two
train books at document level, concentrated in one lamrim group). Chapter validation 8.7%, test 13.8%,
concentrated in a few books. Cross-book shared text is measured and reported, not removed.

## 4. Datasets

Windows of 8,192 tokens (8,190 content tokens plus CLS and SEP) start every 5,120 tokens, so consecutive
windows overlap by 3,070 tokens. Labels are `O`=0, `B-<layer>`=1, `I-<layer>`=2, and a token is labelled by
the token-start rule. Span offsets in the CSVs and dataset columns use an exclusive end, and the scoring
files use an inclusive end. The datasets are on Hugging Face: `formatting-tsawa-v6`,
`formatting-sabche-v1` and `formatting-chapter-v1` (private, under the `Yontenn` account). Each card gives
the format, splits, how the labels were made, known limits, a citation and acknowledgements.

## 5. Training

One script (`train.py`, identical in the three repositories) trains all layers. The layer is chosen by
`--label-name`. The settings that differ are chosen from the data, not by preference.

| | Tsawa | Sabche | Chapter |
|---|---|---|---|
| Base model | mmBERT-base | mmBERT-base | mmBERT-base |
| Learning rate, batch size | 1e-5, 8 | 1e-5, 8 | 1e-5, 8 |
| Epochs | up to 15, best at 6.9 | 8 | 8 |
| Class weights | inverse frequency: O 1, B 1190.7, I 13.5 | inverse frequency: O 1, B 1298, I 25 | **square-root** inverse frequency: O 1, B 142.5, I 24.1 |
| Early stopping | patience 3 epochs, 4 evaluations per epoch | same | same |
| Other | weight decay 0.01, clip 0.3, warmup 6%, bf16, seed 42 | same | same |
| Gradient checkpointing | off | off | on (memory and speed only) |

Chapter uses square-root weights because plain inverse frequency would give `B` about 20,300, since chapter
spans are very sparse. It was never compared with plain inverse frequency. Tsawa also has a side experiment
with a hand-made features column, which did not help and is not part of the pipeline.

Not recorded: the GPU model and training time for some runs, and the best epoch for the sabche and chapter
runs. The training arguments are stored with each model on Hugging Face (`training_args.bin`).

## 6. Evaluation

The same converter and scorer are used for every layer. A prediction is decoded with Viterbi (only legal
BIO transitions, a penalty of 4.0 for every span exit), each book is decoded once, duplicates from the
overlapping windows are removed, and a predicted span counts as correct when its character IoU with a gold
span is at least 0.5, matched greedily one to one and micro-averaged over books. Everything reported here
is whole-book. The trainer's own validation scores are per window, where a span in the overlap counts
twice, so they are not compared with test scores.

## 7. Results

### Trained models (mmBERT), test split

| Layer | Books | F1 | Precision | Recall | Old / new batch F1 |
|---|---|---|---|---|---|
| Tsawa, headline | 21 | **0.596** | 0.485 | 0.774 | 0.573 / 0.629 |
| Tsawa, with the interlinear book | 22 | 0.521 | 0.485 | 0.562 | 0.573 / 0.466 |
| Sabche | 29 | **0.962** | 0.954 | 0.970 | 0.958 / 0.964 |
| Chapter | 36 | **0.829** | 0.768 | 0.899 | 0.863 / 0.800 |

### Zero-shot LLMs, test split, same books and scorer

| Layer | Claude Sonnet 5 | Gemini 3.1 Flash Lite | mmBERT |
|---|---|---|---|
| Tsawa (22 books; 21 books) | 0.603 (0.714) | 0.556 (0.661) | 0.521 (0.596) |
| Sabche | 0.681 | 0.668 | 0.962 |
| Chapter | 0.302 (older prompt) | 0.540 (newer prompt) | 0.829 |

On tsawa the zero-shot models beat the trained model. On sabche and chapter the trained models are far ahead.
The two chapter LLM numbers use different prompts: a paragraph telling the model that a window full of short
heading-like lines is probably Sabche was added after reading Claude's test errors. So the chapter LLM
numbers are not comparable, and the newer prompt is not independent of the test set. Claude was not rerun
with it.

Cost of one test run: Claude about $25 (tsawa), $15 (sabche) and $12 (chapter), Gemini about 500 rupees
per run.

## 8. Findings and limits

- **One tsawa test book is outside the task.** `IF3ACC3E1` is an interlinear commentary annotated a word at
  a time (a median of 2 syllables, against 29 in the other test books). No system matches any of its 747
  gold spans, which is 27% of the tsawa test spans. The tsawa headline is the 21-book score, with the 22-book
  score shown beside it. Four similar books are in train.
- **Sabche annotation is partly incomplete.** Bare `Nth-པ་ནི།` headings are annotated only about 29% of the
  time, so some correct predictions score as false positives. Its test set is large enough to be stable.
- **Chapter is sparse and the test set is small.** 258 spans in 36 books, and more than half of the books
  have only one or two spans, so scores are noisy. One validation book holds 47% of the validation spans and
  the model finds almost none of them, so the validation score (0.620) says little.
- **The layers overlap in look.** The chapter LLM runs mark Sabche outline headings as chapters (88% of
  Claude's false positives). Titles of works inside compilations are labelled Chapter in some books and Book
  title in others.
- **Frozen test splits were scored once per system,** with the break penalty fixed beforehand. The chapter
  test split was touched twice by model inference with the same final model.

## 9. Reproducing and where things are

| Piece | Where |
|---|---|
| Tsawa pipeline, results, model card | `tsawa-layer-detection` (private, `tenzinyonten`) |
| Sabche pipeline, results | `sabche-layer-detection` (private, `tenzinyonten`) |
| Chapter pipeline, results | `chapter-layer-detection` (private, `tenzinyonten`) |
| Zero-shot prompts, LLM results, scripts | this repository (`chapter/`, `sabche/`, `tsawa/`, `src/`) |
| Datasets | Hugging Face `Yontenn/formatting-tsawa-v6`, `formatting-sabche-v1`, `formatting-chapter-v1` |
| Models | Hugging Face `Yontenn/mmbert-tsawa-v6-nofeat`, `mmbert-sabche-v1`, `mmbert-chapter-v1` |

Each layer repository has a `README.md` with the six steps in order, `docs/PIPELINE.md` (why each decision was
made) and `docs/RESULTS.md` (all numbers, per book). Test scores can be checked without a GPU with
`src/score_spans.py` and the saved predictions. Book texts are not in any repository and come from
OpenPecha through `src/fetch_texts.py`. `data/tsawa_audit.csv` records where each text was on the original
machine, so point it at your own copy first. Data-rights questions on the OpenPecha texts are still open, so
check the terms before sharing text or anything derived from it.

## 10. Open items

- Rerun Claude on the chapter layer with the newer prompt (validation first, then test) so the two LLM numbers
  are like for like.
- Decide whether the four interlinear tsawa training books should be dropped and the model retrained.
- Fill in the missing training facts (GPU, time, best epoch) if the run records can be recovered.
- Confirm the visibility of these repositories against the data-rights position.
