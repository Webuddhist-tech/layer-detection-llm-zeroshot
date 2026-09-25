# Chapter results

A predicted span is correct when its character IoU with a gold span is at least 0.5, matched
greedily one to one, micro-averaged over books. Scores come from the predictions in `results/chapter/`
and the shared scorer, `src/score_spans.py`.

**The two LLM columns use different prompts.** Claude ran with `prompt_claude.md`. Gemini ran with
`prompt.md`, which adds one paragraph (the density instruction). That paragraph was written after
reading Claude's test errors, so the test set has already informed the Gemini prompt and the two
columns are not a like-for-like comparison. Neither model was run on validation.

## Test split (36 books, 258 gold spans)

| | Claude Sonnet 5 (`prompt_claude.md`) | Gemini 3.1 Flash Lite (`prompt.md`) | mmBERT (for reference) |
|---|---|---|---|
| All 36 books | 0.302 (P 0.196, R 0.651) | **0.540** (P 0.400, R 0.829) | 0.829 (P 0.768, R 0.899) |
| Old batch (16 books) | 0.331 (P 0.219, R 0.683) | 0.642 (P 0.531, R 0.810) | 0.863 (P 0.853, R 0.873) |
| New batch (20 books) | 0.276 (P 0.177, R 0.621) | 0.472 (P 0.327, R 0.848) | 0.800 (P 0.705, R 0.924) |
| Median book | 0.387 | 0.500 | 0.928 |
| Spans predicted (gold 258) | 856 | 535 | 302 |

Reproduce without any API key:

```
python src/score_spans.py --layer chapter --split test --per-book \
  --model claude=results/chapter/claude-sonnet-5/test/spans \
  --model gemini=results/chapter/gemini-3.1-flash-lite/test/spans
```

### Where the false positives are

Counting predictions that match no gold span, and how many of those overlap a Sabche outline-heading
span in the same book:

| | Predictions | False positives | On Sabche spans |
|---|---|---|---|
| Claude | 856 | 688 | 605 (88%) |
| Gemini | 535 | 321 | 156 (49%) |

Both models mark outline headings as chapters, and Claude does it far more. The density instruction
in the Gemini prompt targets exactly this, so the lower share is consistent with it helping, but the
model and the prompt both changed, so the difference cannot be split between them. Claude's worst
books have hundreds of Sabche spans (P000067 alone had 207 false positives).

### Per book (test F1)

| Book | Batch | Gold | Claude | Gemini | mmBERT |
|---|---|---|---|---|---|
| I7989272E | new | 4 | 1.000 | 0.333 | 0.615 |
| IE9C4806D | new | 2 | 1.000 | 0.800 | 0.667 |
| P000245 | old | 24 | 0.958 | 0.980 | 0.980 |
| P000110 | old | 8 | 0.941 | 0.875 | 0.462 |
| IC05A6BE0 | new | 7 | 0.857 | 0.304 | 0.778 |
| I62D7430C | new | 1 | 0.667 | 0.667 | 1.000 |
| P000223 | old | 1 | 0.667 | 0.667 | 1.000 |
| P000079 | old | 13 | 0.593 | 0.609 | 0.818 |
| IAAADCAA0 | new | 9 | 0.588 | 0.500 | 0.778 |
| P000275 | old | 54 | 0.578 | 0.682 | 0.847 |
| P000118 | old | 4 | 0.571 | 0.800 | 0.889 |
| IE5895799 | new | 25 | 0.510 | 0.629 | 0.923 |
| I99DF06AA | new | 1 | 0.500 | 0.500 | 0.500 |
| I9AEEF96A | new | 1 | 0.500 | 0.000 | 1.000 |
| P000056 | old | 2 | 0.500 | 0.400 | 0.667 |
| I319DAFF7 | new | 1 | 0.500 | 0.000 | 1.000 |
| P000115 | old | 3 | 0.444 | 0.500 | 1.000 |
| I4DFE9067 | new | 2 | 0.400 | 0.500 | 0.800 |
| I52248444 | new | 15 | 0.375 | 0.788 | 0.966 |
| P000013 | old | 2 | 0.333 | 0.235 | 1.000 |
| IC6F06BCD | new | 10 | 0.327 | 0.213 | 0.486 |
| I0FCFA88F | new | 7 | 0.279 | 0.467 | 0.778 |
| I575514A8 | new | 7 | 0.250 | 0.341 | 0.933 |
| P000023 | old | 1 | 0.250 | 0.000 | 1.000 |
| I5F5D9F5A | new | 8 | 0.238 | 0.600 | 0.857 |
| ICDC84458 | new | 9 | 0.167 | 0.485 | 0.941 |
| I3F4A91F5 | new | 11 | 0.137 | 0.733 | 1.000 |
| I9B6A4525 | new | 10 | 0.122 | 0.392 | 0.621 |
| P000067 | old | 7 | 0.009 | 0.273 | 0.769 |
| P000027 | old | 2 | 0.000 | 0.571 | 1.000 |
| P000109 | old | 1 | 0.000 | 0.500 | 1.000 |
| P000172 | old | 1 | 0.000 | 0.000 | 1.000 |
| I45222122 | new | 1 | 0.000 | 0.000 | 1.000 |
| I9D9C7AC9 | new | 1 | 0.000 | 0.667 | 0.667 |
| P000134 | old | 1 | 0.000 | 0.667 | 1.000 |
| P000138 | old | 2 | 0.000 | 0.800 | 1.000 |

Most books have only a handful of gold spans, so single books move the score.

## Tokens and cost

| Run | Windows | Input tokens | Output tokens |
|---|---|---|---|
| Claude, test (Message Batches) | 536 | 10,808,402 (plus 2,762,740 read from the prompt cache) | 112,857 |
| Gemini, test | 536 | 7,505,359 | 101,090 |

Claude's answer for a window with no title is about 10 tokens, so output is a small part of the bill.

| Run | Approximate cost |
|---|---|
| Claude Sonnet 5, test, batch API | about $11.66 (computed from the token counts at list prices) |
| Gemini 3.1 Flash Lite, test | about ₹500 (the owner's figure, not recorded exactly) |
| Claude, one validation book (7 windows, normal API) | about $0.30 |

## Not run

- No Claude run on validation, and no Claude run with the density instruction.
- A Gemini validation run with the earlier prompt stopped at 120 of 550 windows on a billing error.
  It is not used anywhere.
