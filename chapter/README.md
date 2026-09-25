# Chapter: zero-shot prompt

How the chapter (ལེའུ་, chapter and section heading) prompt for Claude Sonnet 5 and Gemini 3.1 Flash
Lite was built, what it was tested on, and what it does badly. The prompts are in
[prompt_claude.md](prompt_claude.md) (Claude) and [prompt.md](prompt.md) (Gemini), and they differ by
one paragraph. Scores, token counts and cost are in [RESULTS.md](RESULTS.md).

## The task

Given a 16,000-character window of a Tibetan text, return every chapter title in it. A chapter span is
the heading line that opens a division of a book. It names the division and is not the text under it.
The setup is the same as for tsawa and sabche (see those folders): 16,000-character windows overlapping
by 2,000, cut at a shad, spans returned as `{label, frame, head, tail}` and mapped back to offsets by a
locator.

## The prompt

It was written on 2026-09-25. Every number in it was measured on the **train and validation books
only** (342 books, 2,784 spans, 5,706 windows). The test books were not read when it was first
written, and the worked examples are copied from train and validation books.

| Signal in train and validation | Value | What the prompt does with it |
|---|---|---|
| Median length 43 characters (p10 15, p90 104, none over 212) | | Titles are one short line. 46% are 40 characters or fewer and go whole in `head`. |
| Starts at the beginning of a line | 98.6% | Only line-initial candidates count. |
| Followed straight away by a shad and a line break | 89% | The shad is not part of the span. |
| Comes right after a line that closes the previous unit | 56% | Look at the line before the candidate. |
| Opens with `༄༅། །` | 34% | The opener is part of the span. |
| Ends `ཞེས་བྱ་བ་བཞུགས་སོ` or similar | 24% | Typical of a work title in a collected volume. |
| Starts with `ལེའུ་` and a number | 5.6% | Only a small part of the layer is numbered chapters. |
| Window has at least one title | 23% (two or more: 11%) | The answer is empty for about three windows in four. The prompt says so first. |
| More than half of the books have one or two titles | 58% | Do not invent titles to fill a window. |
| Overlap with the Sabche layer | none | Outline headings such as `དང་པོ་ནི།` are never marked here. |

The prompt describes four kinds of chapter span: the title of a work in a collected volume, a numbered
chapter line, a plain section title, and front matter such as `དཀར་ཆག` (table of contents) or
`དཔེ་སྐྲུན་གསལ་བཤད` (publisher's note). One rule came from looking at the data: a short bare
chapter-number line that ends `པའོ` (`ལེའུ་གཉིས་པའོ།`) is a title and is marked, while a long closing
colophon is not. All 27 annotated lines of that kind are under 26 characters and come from two books, and
none of 829 longer lines that contain both `ལེའུ` and `པའོ` is annotated. It ends with six deciding
steps and six worked examples, one of them an outline heading with body text that must return an empty
list.

## How it was tested

| Step | Model | Data | Result |
|---|---|---|---|
| One-book check | Claude | validation book I8698A1DF, 7 windows | F1 0.667, 4 of 6 gold found |
| Test, once | Claude Sonnet 5 | 36 books, 536 windows | F1 0.302 (P 0.196, R 0.651) |
| Test, once | Gemini 3.1 Flash Lite | 36 books, 536 windows | F1 0.540 (P 0.400, R 0.829) |

The one-book check cost about $0.30 and showed the setup works: it found four of the six gold spans (the
front-matter heading, two section titles and the last work title), missed two work titles (one because it
mistyped the end anchor, one because it skipped it), and added two extra headings that were not
annotated.

Claude's test score was low, and 88% of its false positives overlapped Sabche outline headings, which the
prompt says not to mark. After seeing that, a paragraph was added to the prompt: a window with ten or
more short heading-like lines is almost certainly Sabche, so return an empty list unless a work title or
numbered chapter line is clearly among them. Gemini then ran once with the new prompt.

**This means the two test scores are not comparable, and the Gemini prompt is not independent of the
test set.** The density paragraph was written after reading Claude's test errors. Neither model was run on
validation, and Claude was not rerun with the new prompt. A Gemini validation run with the earlier prompt
started but stopped at 120 of 550 windows on a billing error, and it is not used.

## Model settings

The same as for tsawa and sabche: Claude Sonnet 5 through the Message Batches API with adaptive thinking at
low effort, JSON-schema output, a 1-hour prompt cache, no temperature and 64,000 max tokens. Gemini 3.1
Flash Lite at temperature 1.0, thinking level low, JSON output. The Gemini run stalled once on a hung
call at 426 windows and was resumed, keeping the saved windows.

## What it does badly

- **Both models mistake Sabche outline headings for chapters.** Claude made 856 predictions for 258 gold
  spans, and 605 of its 688 false positives overlap Sabche spans. Gemini with the density paragraph made
  535 predictions, and 156 of its 321 false positives do.
- Precision is the weak side for both (0.196 and 0.400), while recall is fair (0.651 and 0.829).
- The new batch is harder for both than the old batch (Gemini 0.472 against 0.642).
- The annotation is sparse, so some false positives are probably real headings that were never labelled.
  This was not checked one by one.

The trained mmBERT scores 0.829 on the same books (see the chapter-layer-detection repo), far above both.
