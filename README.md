# layer-detection-llm-zeroshot

Zero-shot prompts for finding annotation layers in Tibetan commentaries with an LLM and no
training: what the prompt says and why, what it scored, and what it cost. One folder per layer.

| Layer | What it finds | Test F1, Claude Sonnet 5 | Test F1, Gemini 3.1 Flash Lite |
|---|---|---|---|
| [tsawa](tsawa/) | root text (རྩ་བ) the commentary quotes and explains | 0.603 | 0.556 |
| [sabche](sabche/) | outline headings (ས་བཅད) | 0.681 | 0.668 |
| [chapter](chapter/) | chapter and section headings (ལེའུ་) | 0.302 * | 0.540 * |

* Chapter: the two models used different prompts (Claude `chapter/prompt_claude.md`, Gemini `chapter/prompt.md`, which adds one paragraph written after reading Claude's test errors), so the two numbers are not comparable. See [chapter/RESULTS.md](chapter/RESULTS.md).

## How a prompt is run

The same scripts run every layer (`--layer tsawa`, `sabche` or `chapter`).

1. **Get the book texts.** They are not in this repo. `src/fetch_texts.py` clones them from
   OpenPecha using an ID list from `data/`:
   ```
   python src/fetch_texts.py --ids-file data/tsawa_ids.txt --raw-dir data/raw_opf \
       --manifest data/raw_opf/_manifest.csv
   ```
2. **Cut each book into windows and ask the model.** `src/zeroshot_run.py` cuts 16,000-character
   windows that overlap by 2,000 (`src/gemini_chunks.py`), sends the layer's `prompt.md` plus one
   window per call, and reads back anchors. `src/gemini_locate.py` maps the anchors to character
   offsets, removes duplicates from the overlap, and scores the book. Replies are cached one file
   per window, so a re-run costs nothing for windows already done.
   ```
   python src/zeroshot_run.py --layer tsawa --texts-dir data/raw_opf --books P000067 \
       --provider gemini --model gemini-3.1-flash-lite --out runs/tsawa/gemini
   ```
   Use `--provider anthropic --model claude-sonnet-5` for Claude. Keys are read from
   `GEMINI_API_KEY` or `ANTHROPIC_API_KEY`.
3. **Claude in bulk (optional).** `src/claude_batch.py` runs the test windows through the
   Message Batches API at half price, in groups: `plan`, `create --yes`, `poll`, `fetch`. The
   sabche Claude run used this. Its replies land in the same per-window cache.
4. **Score.** `src/score_spans.py` scores span files against the gold in `data/`, per book and by
   batch. It needs no texts and no keys, and the predictions from the reported runs are in
   `results/`.

## Layout

```
tsawa/   README.md (prompt story), prompt.md, RESULTS.md
sabche/  README.md (prompt story), prompt.md, RESULTS.md
chapter/ README.md (prompt story), prompt.md and prompt_claude.md, RESULTS.md
src/     zeroshot_run.py, claude_batch.py, score_spans.py, fetch_texts.py,
         gemini_chunks.py (windowing), gemini_locate.py (anchors to offsets)
data/    gold spans, per-book split and batch, book ID lists (offsets only, no text)
results/ predicted spans from the reported runs: results/<layer>/<model>/<split>/spans
```

Each layer folder holds the prompt story (`README.md`), the exact prompt that was run
(`prompt.md`) and the numbers, tokens and cost (`RESULTS.md`).

Scores use character IoU of at least 0.5, matched one to one, over the same held-out test
books as the trained mmBERT models. The trained models score higher on sabche (0.962) and lower
on tsawa (0.521); see
[sabche-layer-detection](https://github.com/tenzinyonten/sabche-layer-detection) and
[tsawa-layer-detection](https://github.com/tenzinyonten/tsawa-layer-detection). The original
evaluation repos, which also hold the mmBERT predictions, are
[tsawa-zeroshot-eval](https://github.com/tenzinyonten/tsawa-zeroshot-eval) and
[sabche-zeroshot-eval](https://github.com/tenzinyonten/sabche-zeroshot-eval).
