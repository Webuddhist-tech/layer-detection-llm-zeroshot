# layer-detection-llm-zeroshot

Zero-shot prompts for finding annotation layers in Tibetan commentaries with an LLM and no
training: what the prompt says and why, what it scored, and what it cost. One folder per layer.

| Layer | What it finds | Test F1, Claude Sonnet 5 | Test F1, Gemini 3.1 Flash Lite |
|---|---|---|---|
| [tsawa](tsawa/) | root text (རྩ་བ) the commentary quotes and explains | 0.603 | 0.556 |
| [sabche](sabche/) | outline headings (ས་བཅད) | 0.681 | 0.668 |

Each folder has:

- `README.md`: how the prompt was built, what it was tested on, what it does badly
- `prompt.md`: the exact prompt that was run
- `RESULTS.md`: F1, precision and recall on validation and test, per book, tokens and cost

Scores use character IoU of at least 0.5, matched one to one, over the same held-out test
books as the trained mmBERT models. The trained models score higher on sabche (0.962) and lower
on tsawa (0.521); see
[sabche-layer-detection](https://github.com/tenzinyonten/sabche-layer-detection) and
[tsawa-layer-detection](https://github.com/tenzinyonten/tsawa-layer-detection). The
predictions and the code that runs the LLMs and scores them are in
[tsawa-zeroshot-eval](https://github.com/tenzinyonten/tsawa-zeroshot-eval) and
[sabche-zeroshot-eval](https://github.com/tenzinyonten/sabche-zeroshot-eval).
