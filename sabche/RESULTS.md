# Sabche results

A predicted span is correct when its character IoU with a gold span is at least 0.5, matched
greedily one to one, micro-averaged over books. Scores come from the predictions in the
sabche-zeroshot-eval repo.

## Test split (29 books, 4,079 gold spans)

Reproduce without any API key (the predictions are in `results/sabche/`):

```
python src/score_spans.py --layer sabche --split test --per-book \
  --model claude=results/sabche/claude-sonnet-5/test/spans \
  --model gemini=results/sabche/gemini-3.1-flash-lite/test/spans
```

| | Claude Sonnet 5 | Gemini 3.1 Flash Lite | mmBERT (for reference) |
|---|---|---|---|
| All 29 books | **0.681** (P 0.613, R 0.766) | 0.668 (P 0.600, R 0.754) | 0.962 (P 0.954, R 0.970) |
| Old batch (11 books) | 0.643 (P 0.533, R 0.809) | 0.547 (P 0.458, R 0.678) | 0.958 (P 0.940, R 0.977) |
| New batch (18 books) | 0.712 (P 0.688, R 0.737) | 0.762 (P 0.724, R 0.804) | 0.964 (P 0.964, R 0.965) |
| Median book | 0.684 | 0.623 | 0.979 |
| Spans predicted (gold 4,079) | 5,096 | 5,129 | 4,144 |

## Validation

Gemini only, on 22 of the 33 validation books (2,645 gold spans): F1 0.549 (P 0.466, R 0.669).
The run stopped before the last 11 books. Claude was not run on validation.

## Per book (test F1)

| Book | Batch | Gold | Claude | Gemini | mmBERT |
|---|---|---|---|---|---|
| P000083 | old | 225 | 0.587 | 0.417 | 0.998 |
| P000067 | old | 382 | 0.696 | 0.610 | 0.992 |
| P000037 | old | 161 | 0.488 | 0.482 | 0.943 |
| P000164 | old | 313 | 0.785 | 0.621 | 0.989 |
| P000242 | old | 261 | 0.650 | 0.585 | 0.907 |
| IC05A6BE0 | new | 8 | 0.093 | 0.024 | 0.824 |
| IDB6093E9 | new | 154 | 0.665 | 0.596 | 0.945 |
| IE5895799 | new | 227 | 0.736 | 0.743 | 0.989 |
| I36A7A668 | new | 88 | 0.556 | 0.495 | 0.918 |
| P000144 | old | 107 | 0.645 | 0.618 | 0.986 |
| I3F4A91F5 | new | 367 | 0.591 | 0.875 | 0.991 |
| P000269 | old | 29 | 0.099 | 0.140 | 0.276 |
| I52248444 | new | 236 | 0.774 | 0.816 | 0.989 |
| I7D476A8B | new | 55 | 0.543 | 0.545 | 0.915 |
| IC6F06BCD | new | 122 | 0.725 | 0.691 | 0.955 |
| I0FCFA88F | new | 248 | 0.833 | 0.824 | 0.925 |
| P000247 | old | 8 | 0.314 | 0.280 | 1.000 |
| ICDC84458 | new | 181 | 0.819 | 0.834 | 0.962 |
| P000013 | old | 44 | 0.763 | 0.623 | 0.926 |
| I07240379 | new | 118 | 0.867 | 0.813 | 0.987 |
| IFA88A536 | new | 254 | 0.613 | 0.891 | 0.982 |
| P000118 | old | 42 | 0.735 | 0.660 | 0.988 |
| P000027 | old | 52 | 0.776 | 0.606 | 0.990 |
| IF3ACC3E1 | new | 45 | 0.684 | 0.815 | 1.000 |
| I9D9C7AC9 | new | 42 | 0.612 | 0.215 | 0.977 |
| I575514A8 | new | 77 | 0.637 | 0.632 | 0.775 |
| I9B6A4525 | new | 196 | 0.982 | 0.982 | 0.979 |
| I319DAFF7 | new | 15 | 0.875 | 0.933 | 1.000 |
| I9AEEF96A | new | 22 | 0.894 | 0.930 | 0.978 |

## Tokens and cost

| Run | Windows | Input tokens | Output tokens |
|---|---|---|---|
| Gemini, validation (22 books) | 431 | 5,499,223 | 205,913 |
| Gemini, test | 540 | 6,991,050 | 395,621 |
| Claude, test | 540 | 12,863,170 (1,858,680 of them read from the prompt cache) | 952,555 |

Claude ran through the Message Batches API, which is billed at half price.

| Run | Approximate cost |
|---|---|
| Claude Sonnet 5, test | about $15 |
| Gemini 3.1 Flash Lite, one run | about ₹500 |

These are rough figures from the account, not exact. As a cross-check, the Claude token counts
above at list prices with the batch discount come to about $16, and the last of the three
batches cost $5.34.
