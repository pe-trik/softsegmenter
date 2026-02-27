# SoftSegmenter

A tool for re-segmenting long-form simultaneous speech translation outputs to match reference segmentation, enabling segment-level quality (BLEU) and latency (YAAL) evaluation.

Implements the SoftSegmenter alignment algorithm from [*Better Late Than Never: Evaluation of Latency Metrics for Simultaneous Speech-to-Text Translation*](https://arxiv.org/abs/2509.17349).


## How It Works

Simultaneous speech translation systems (e.g., those evaluated via [SimulEval](https://github.com/facebookresearch/SimulEval)) produce a single long-form output per audio recording. However, most evaluation metrics (like BLEU and YAAL) are designed for segmented inputs. SoftSegmenter takes the reference speech segmentation and aligns the hypothesis words (including its emission timestamps) to it, effectively re-segmenting the hypothesis according to the reference segments.

The pipeline consists of the following steps:

1. **Tokenization** — Reference and hypothesis words are optionally tokenized using the [Moses tokenizer](https://github.com/luismsgomes/mosestokenizer). For Chinese/Japanese (or when `--lang` is not set), tokenization is skipped and character-level units are used instead.
2. **Alignment** — A dynamic programming algorithm (similar to Needleman-Wunsch / DTW, but without gap penalties) aligns hypothesis words to reference words. The alignment maximizes a Jaccard-based character-set similarity score at word level, or exact match at character level. Punctuation is prevented from aligning with non-punctuation tokens, which helps to mitigate segmentation errors around sentence boundaries.
3. **Re-segmentation** — Aligned hypothesis words are grouped by their assigned reference segment IDs, producing one hypothesis segment per reference segment.
4. **Evaluation** — Each re-segmented instance is scored with:
   - **LongYAAL** (Yet Another Average Lagging) — both computation-aware and computation-unaware variants
   - **BLEU** via [SacreBLEU](https://github.com/mjpost/sacrebleu)


## Installation

```bash
# Clone the repository
git clone https://github.com/pe-trik/softsegmenter.git
cd softsegmenter

# Install dependencies
pip install -r requirements.txt
```

### Requirements

- Python 3.8+
- `mosestokenizer>=1.2.1`
- `PyYAML>=6.0.3`
- `sacrebleu>=2.5.1`


## Usage

```bash
python softsegmenter.py \
  --speech_segmentation ref_segments.yaml \
  --ref_sentences_file reference_sentences.txt \
  --hypothesis_file simuleval_instance_file.log \
  --lang en \
  --bleu_tokenizer 13a \
  --output_folder segmentation_output
```

### Arguments

| Argument | Required | Default | Description |
|---|---|---|---|
| `--speech_segmentation` | Yes | — | Path to a YAML or JSON file defining the reference speech segmentation. Each line must contain `wav`, `offset`, and `duration` fields. |
| `--ref_sentences_file` | Yes | — | Path to a plain-text file with one reference sentence per line (aligned to the segments in the speech segmentation file). |
| `--hypothesis_file` | Yes | — | Path to a JSONL file with SimulEval instance outputs. Each line is a JSON object with `source`, `prediction`, `delays`, and optionally `elapsed` fields. |
| `--lang` | No | `None` | Language code for the Moses tokenizer (e.g., `en`, `de`). Set to `None`, `zh`, or `ja` to skip tokenization. |
| `--char_level` | No | `False` | Use character-level alignment and scoring instead of word-level. |
| `--bleu_tokenizer` | No | `13a` | Tokenizer for SacreBLEU (e.g., `13a`, `intl`, `ja-mecab`, `zh`). |
| `--output_folder` | Yes | — | Directory where output files will be written. |
| `--offset_delays` | No | `False` | Offset delays relative to the first segment of each recording. Useful when hypothesis delays are relative to the start of the first segment rather than the full recording. |
| `--fix_elapsed` | No | `False` | Fix elapsed times for computation-aware YAAL. SimulEval computes elapsed as `ELAPSED_i = DELAY_i + TOTAL_RUNTIME_i`; this flag corrects it to be incremental (`NEW_ELAPSED_i = ELAPSED_i - ELAPSED_{i-1} + DELAY_{i-1}`). |


## Input Formats

### Speech Segmentation (YAML/JSON)

A list of segments, each with the following fields:

```yaml
- {wav: recording.wav, offset: 2.433, duration: 9.05, speaker_id: spk1}
- {wav: recording.wav, offset: 15.003, duration: 9.675, speaker_id: spk1}
```

- `wav` — audio filename (used to group segments by recording)
- `offset` — segment start time in seconds
- `duration` — segment duration in seconds
- `speaker_id` — (optional) speaker identifier

### Reference Sentences

One sentence per line, aligned 1:1 with the segmentation entries:

```
Hello, this is Elena and I will present our work.
We will discuss what lexical borrowing is.
```

### Hypothesis File (JSONL)

One JSON object per line (SimulEval output format):

```json
{"source": ["recording.wav"], "prediction": "Hello this is Elena ...", "delays": [4067.0, 4067.0, ...], "elapsed": [4100.0, 4200.0, ...], "source_length": 220000}
```

- `source` — list with the audio filename
- `prediction` — the full hypothesis text
- `delays` — per-token ideal emission delays (in ms) the length of this list should match the number of words (characters if `--char_level` is set) in `prediction`
- `elapsed` — (optional) per-token computation-aware delays (in ms)
- `source_length` — (optional, but highly recommended) total recording length in ms


## Output

The tool creates the following files in `--output_folder`:

### `instances.resegmented.json`

A JSON array of re-segmented instances, one per reference segment:

```json
[
  {
    "index": 0,
    "prediction": "Hello , this is Elena and I will present our work .",
    "reference": "Hello, this is Elena and I will present our work.",
    "source_length": 9050.0,
    "delays": [4067.0, 4067.0, ...],
    "elapsed": [4100.0, 4200.0, ...],
    "recording_end": 220000.0
  },
  ...
]
```

Each entry contains the hypothesis words assigned to that segment, with delays offset relative to the segment start.

### `scores.resegmented.csv`

A tab-separated file with aggregate scores:

```
ca_unaware_yaal	ca_aware_yaal	bleu
2968.2952	5905.0729	22.9429
```

- `ca_unaware_yaal` — YAAL computed with ideal delays
- `ca_aware_yaal` — YAAL computed with computation-aware (elapsed) delays
- `bleu` — corpus-level SacreBLEU score


## Example

See the [example/](example/) directory for sample input files and expected output. Run the example with:

```bash
cd example
bash resegment.sh
```


## Citation

If you use this tool in your research, please cite it as follows:

```bibtex
@article{polak2025better,
  title={Better Late Than Never: Evaluation of Latency Metrics for Simultaneous Speech-to-Text Translation},
  author={Pol{\'a}k, Peter and Papi, Sara and Bentivogli, Luisa and Bojar, Ond{\v{r}}ej},
  journal={arXiv preprint arXiv:2509.17349},
  year={2025}
}
```

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.