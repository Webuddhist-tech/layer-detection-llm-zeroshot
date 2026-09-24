# Sabche: zero-shot prompt

How the sabche (ས་བཅད, outline headings) prompt for Gemini 3.1 Flash Lite and Claude Sonnet 5
was built, what it was tested on, and what it does badly. The prompt itself is in
[prompt.md](prompt.md). Scores, token counts and cost are in [RESULTS.md](RESULTS.md).

## The task

Given a 16,000-character window of a Tibetan commentary, return every outline heading in it,
for example `གཉིས་པ་༼ཚུལ་ཁྲིམས་ཕར་ཕྱིན་གྱི་རབ་དབྱེ་༽ནི`. Windowing, anchors and the locator are
the same as for tsawa (see the tsawa folder): 16,000-character windows overlapping by 2,000,
cut at a shad, spans returned as `{label, frame, head, tail}` and mapped back to offsets by a
locator.

## The prompt

It was written on 2026-09-24. Every number in it was measured on the **train and validation
books only** (294 books, 38,213 spans). The test books were not read while it was written, and
the worked examples are copied from train and validation books.

| Signal in train and validation | Value | What the prompt does with it |
|---|---|---|
| Opens with a number word (དང་པོ, གཉིས་པ, གསུམ་པ ...) | 70% | Tells the model what a heading looks like. |
| Ends in `ནི`, usually with the topic in `༼ ༽` before it | 74% | Same. |
| Begins at the start of a line | 98% | Same. |
| Numbered with brackets instead, one per line, often several in a row: `(1)`, `{2}`, `[3]` | about 9% | Each line is its own span, and a run is never merged. |
| Ends at `ནི` or the last syllable of the topic, with the closing shad outside | 95% | The span is the heading only and stops before the shad. |
| Median length 40 characters (p25 27, p75 93, p90 183) | | Most headings go whole in `head` with an empty `tail`. |

The prompt also lists what not to mark: root text under a heading (the heading is the span,
the root text is not), quotations after a named source, the commentator's own exposition, a
number word inside a running sentence, and colophons. It has four worked examples: a heading
with a parenthesised topic, bracket-numbered headings in a run, a long heading that lists its
sub-items, and a heading followed by a quotation where only the heading is marked.

## How it was tested

| Step | Model | Data | Result |
|---|---|---|---|
| Validation | Gemini | 22 of 33 books, 431 windows | F1 0.549 (P 0.466, R 0.669) |
| Test | Gemini | 29 books, 540 windows | F1 0.668 (P 0.600, R 0.754) |
| Test | Claude Sonnet 5 | 29 books, 540 windows | F1 0.681 (P 0.613, R 0.766) |

The Gemini validation run stopped after 22 of the 33 books, so the validation number covers
only those. It is not a random sample of the split. Claude was not run on validation. The
test split was scored once per model.

For Claude, the test was run with the Message Batches API (half price, asynchronous): a
2-request trial batch first, then three batches of 179, 180 and 179 windows. Gemini ran
window by window under a watchdog that re-ran until all 540 windows were saved.

## Model settings

The same as for tsawa: Claude Sonnet 5 with adaptive thinking at low effort, JSON-schema
output and a 1-hour prompt cache, no temperature, 64,000 max tokens. Gemini 3.1 Flash Lite at
temperature 1.0, thinking level low, JSON output. Same prompt and windows for both.

## What it does badly

- Both models predict far too many spans: Gemini 5,129 and Claude 5,096 for 4,079 gold, so
  precision is about 0.60 while recall is about 0.75.
- The worst cases are books with few gold headings. P000083 gets 344 extra predictions from
  Gemini and 232 from Claude, and IC05A6BE0 has 8 gold spans against 159 and 136 extras.
- The gold does not mark every heading. Bare `Nth-པ་ནི།` headings with no title are annotated
  only about 29% of the time, so many of the extra predictions are probably real headings that
  were never annotated. This was not checked one by one.
- Old-batch books are hard for both (Gemini 0.547, Claude 0.643). On new-batch books Gemini
  does better than Claude (0.762 against 0.712), with a few large gaps such as I3F4A91F5
  (Gemini 0.875, Claude 0.591) and IFA88A536 (0.891 against 0.613). Why was not looked into.

The trained mmBERT scores 0.962 on the same books, far above both (see the
sabche-layer-detection repo for the trained model).
