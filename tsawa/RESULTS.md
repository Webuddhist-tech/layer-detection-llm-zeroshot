# Tsawa results

A predicted span is correct when its character IoU with a gold span is at least 0.5, matched
greedily one to one, micro-averaged over books. Scores come from the predictions in the
tsawa-zeroshot-eval repo.

## Test split (22 books, 2,734 gold spans)

| | Claude Sonnet 5 | Gemini 3.1 Flash Lite | mmBERT (for reference) |
|---|---|---|---|
| All 22 books | **0.603** (P 0.705, R 0.526) | 0.556 (P 0.650, R 0.486) | 0.521 (P 0.485, R 0.562) |
| Without IF3ACC3E1 | **0.714** (P 0.705, R 0.724) | 0.661 (P 0.653, R 0.669) | 0.596 (P 0.485, R 0.774) |
| Old batch (10 books) | **0.672** (P 0.672, R 0.673) | 0.649 (P 0.658, R 0.639) | 0.573 (P 0.476, R 0.720) |
| New batch (12 books) | **0.531** (P 0.754, R 0.410) | 0.465 (P 0.639, R 0.366) | 0.466 (P 0.497, R 0.438) |
| Median book | **0.767** | 0.703 | 0.693 |
| Spans predicted (gold 2,734) | 2,039 | 2,044 | 3,169 |

Claude beats Gemini on 14 books, loses on 4 and is within 0.005 on 4. It beats mmBERT on 16
of 22.

## Validation (18 books, 1,700 gold spans)

Gemini only: F1 0.568 (P 0.509, R 0.642). Claude was not run on validation.

## Per book (test F1)

| Book | Batch | Gold | Claude | Gemini | mmBERT |
|---|---|---|---|---|---|
| P000118 | old | 52 | 0.990 | 0.917 | 0.945 |
| P000056 | old | 108 | 0.986 | 0.991 | 0.977 |
| I9B6A4525 | new | 200 | 0.978 | 0.940 | 0.786 |
| I575514A8 | new | 33 | 0.969 | 0.952 | 0.815 |
| P000067 | old | 207 | 0.967 | 0.933 | 0.950 |
| I9D9C7AC9 | new | 43 | 0.965 | 0.941 | 0.911 |
| IC05A6BE0 | new | 166 | 0.963 | 0.748 | 0.894 |
| P000164 | old | 183 | 0.950 | 0.956 | 0.917 |
| P000013 | old | 89 | 0.903 | 0.798 | 0.892 |
| I319DAFF7 | new | 17 | 0.828 | 0.828 | 0.591 |
| I9AEEF96A | new | 19 | 0.800 | 0.919 | 0.667 |
| I0FCFA88F | new | 144 | 0.734 | 0.659 | 0.749 |
| I3F4A91F5 | new | 58 | 0.685 | 0.628 | 0.487 |
| P000027 | old | 51 | 0.566 | 0.641 | 0.718 |
| P000083 | old | 190 | 0.512 | 0.503 | 0.434 |
| IC6F06BCD | new | 14 | 0.488 | 0.333 | 0.324 |
| IE5895799 | new | 29 | 0.309 | 0.216 | 0.257 |
| P000144 | old | 113 | 0.249 | 0.230 | 0.245 |
| P000242 | old | 171 | 0.229 | 0.201 | 0.146 |
| ICDC84458 | new | 61 | 0.194 | 0.191 | 0.217 |
| P000269 | old | 39 | 0.022 | 0.136 | 0.169 |
| IF3ACC3E1 | new | 747 | 0.000 | 0.000 | 0.000 |

## Tokens and cost

| Run | Windows | Input tokens | Output tokens |
|---|---|---|---|
| Gemini, validation | 258 | 3,901,128 | 211,421 |
| Gemini, test | 361 | 5,462,201 | 231,173 |
| Claude, test | 361 | 7,312,436 (plus 2,494,149 read from the prompt cache) | 1,206,126 |

Claude ran through the normal API with streaming, one window at a time.

| Run | Approximate cost |
|---|---|
| Claude Sonnet 5, test | about $25 |
| Gemini 3.1 Flash Lite, one run | about ₹500 |

These are rough figures from the account, not exact. As a cross-check, the Claude token counts
above at list prices ($2 per million input, $0.20 cached, $10 output) come to about $27, and
the evaluation repo README says about $22.
