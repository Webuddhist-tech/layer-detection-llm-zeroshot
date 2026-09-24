# Tsawa: zero-shot prompt

How the tsawa (རྩ་བ, root text) prompt for Gemini 3.1 Flash Lite and Claude Sonnet 5 was
built, what it was tested on, and what it does badly. The prompt itself is in
[prompt.md](prompt.md). Scores, token counts and cost are in [RESULTS.md](RESULTS.md).

## The task

Given a 16,000-character window of a Tibetan commentary, return every stretch of tsawa in it.
The models do not return offsets. They return anchors, and a locator maps the anchors back to
character offsets in the book.

## What we started from

The output format, the windowing and the locator come unchanged from the quotation benchmark
(Tibetan-quotation-detection), so the two tasks share one setup:

- Windows of 16,000 characters that overlap by 2,000, each cut just after the nearest shad.
- Each span comes back as `{label, frame, head, tail}`. `head` and `tail` are the first and
  last 20 characters or so of the span, extended to a syllable boundary. A span of 40
  characters or fewer goes whole in `head` with an empty `tail`. `frame` is the outline
  heading right before the span, only there to help locate it.
- The locator finds `head` with a forward cursor (one retry from the window start), then
  `tail` within 2,000 characters. Spans found twice in the overlap are merged at IoU 0.5.

## The prompt

It was written on 2026-09-22. Every number in it was measured on the **train and validation
books only** (100 books, 14,829 merged spans). The test books were not read while it was
written, and the worked examples are copied from train and validation books.

The measurements decided what the prompt says:

| Signal in train and validation | Value | What the prompt does with it |
|---|---|---|
| The prose after a span repeats and explains its words (the "gloss test") | 70% | This is the main test, and the only one that works when there is no heading or closer. |
| Span follows a named source (`ལས།`, `ཇི་སྐད་དུ།`, a person plus `ཞལ་ནས།`) | 3.3% | A named source before a passage means it is a quotation, so it is not marked. Marking quotations is the most expensive mistake, since most windows hold both. |
| `ནི` in the 30 characters before the span | 45% | The outline heading is a pointer, not proof. More than half of spans have none. |
| Span starts right after a closing `༽` | 16% | Same as above. |
| Followed by a closer such as `ཞེས་གསུངས` | 40% | The closer stays outside the span. `ཞེས་གསུངས` closes tsawa (12.7%) about as often as quotations (11.1%), so it says where a span ends and not what kind it is. |
| Contains a line break | 59% | Verse layout is weak evidence, since most verse quotations have it too. Never mark a passage just because it is verse. |
| Median length 96 characters, a third are 45 or shorter | | Do not pad a short span, and do not cut a long one (about 1% run past 850 characters). |
| Window has at least one tsawa span | 72% | An empty answer is right about one window in four, but it is not a safe default. |

The prompt also says: a new outline heading always starts a new span, `ཞེས་དང༌` between two
passages ends one span and starts another, and headings, closers, colophons and the
commentator's own glosses are never marked. It ends with four decision steps (named source,
then gloss test, then outline heading, then the commentator's own voice) and five worked
examples, one of them a verse with a named source that must not be marked.

## How it was tested

| Step | Model | Data | Result |
|---|---|---|---|
| Smoke test | Gemini | one window of I895C519A | 11 spans predicted, all correct (recall means nothing on one window) |
| Small check | Gemini | 3 books, 17 windows | F1 0.567 (P 0.769, R 0.449) |
| Validation | Gemini | 18 books, 258 windows | F1 0.568 (P 0.509, R 0.642) |
| Test, once | Gemini | 22 books, 361 windows | F1 0.556 (P 0.650, R 0.486) |
| Test, once | Claude Sonnet 5 | 22 books, 361 windows | F1 0.603 (P 0.705, R 0.526) |

The validation run is the only one used to develop the prompt. The test split was scored once
per model at the end and was not used for tuning. Claude was not run on validation.

## Model settings

| | Claude Sonnet 5 | Gemini 3.1 Flash Lite |
|---|---|---|
| Model id | `claude-sonnet-5` | `gemini-3.1-flash-lite` |
| Output | JSON schema (`label`, `head`, `tail` required, `frame` optional) | JSON MIME type |
| Thinking | adaptive, effort low | thinking level low |
| Sampling | none (Sonnet 5 accepts no temperature) | temperature 1.0 |
| Max tokens | 64,000 (streamed) | default |
| Caching | 1-hour prefix cache on the prompt | none |

The same prompt and the same windows were used for both.

## What it does badly

- Both models are cautious. Claude predicts 2,039 spans and Gemini 2,044 for 2,734 gold, so
  precision (0.705 and 0.650) is much higher than recall (0.526 and 0.486).
- New-batch books are harder than old-batch ones (Claude 0.531 against 0.672).
- One book, IF3ACC3E1 (an interlinear commentary with 747 gold spans, 27% of the test gold),
  scores 0.000 for Claude, Gemini and the trained mmBERT (a gold span every 48 characters or
  so). Without it Claude scores 0.714 and Gemini 0.661.
- Five books score under 0.31 for Claude (P000269 0.022, ICDC84458 0.194, P000242 0.229,
  P000144 0.249, IE5895799 0.309).

The trained mmBERT scores 0.521 on the same books, so both prompted models beat it here (see
the tsawa-layer-detection repo for the trained model).
