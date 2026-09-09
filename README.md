# Indic-Token-izer

A balanced **Byte-Pair Encoding (BPE)** tokenizer trained for **Indic + English** Wikipedia text with faithful markdown preservation. Designed for equitable fertility across 4 languages while maintaining a compact 10k vocabulary.

> **Correction:** This repository contains **4 languages**, not 5 — verified from `metrics.json:3-8` and `tokenizer.json:42-315`:
> `en` (English), `hi` (Hindi/Devanagari), `ta` (Tamil), `te` (Telugu).

---

## 📁 Repository Structure

```
Indic-Token-izer/
├── tokenizer.json   # Hugging Face `tokenizers` BPE model (vocab + merges, 10k)
├── metrics.json     # Training / evaluation metrics (faithful units, ratios, scores)
├── .gitignore
└── README.md
```

---

## 🔤 Tokenizer — `tokenizer.json`

### Architecture

| Field | Value | Location |
|-------|-------|----------|
| **Model** | `BPE` | `tokenizer.json:33` |
| **Version** | `1.0` | `tokenizer.json:2` |
| **Vocab size** | `10,000` | `metrics.json:15` / `tokenizer.json:42-` (ids 0–9999) |
| **Merges** | `9,658` | `tokenizer.json:merges` |
| **Normalizer** | `NFKC` | `tokenizer.json:16-18` |
| **Pre-tokenizer** | `Metaspace` (replacement `▁`, `prepend_scheme: never`, `split: true`) | `tokenizer.json:19-25` |
| **Decoder** | `Metaspace` (same as pre-tokenizer) | `tokenizer.json:26-31` |
| **Post-processor** | `null` | `tokenizer.json:25` |
| **Truncation / Padding** | `null` (disabled) | `tokenizer.json:3-4` |
| **Special tokens** | `[UNK]` id `0` | `tokenizer.json:5-14` |
| **byte_fallback** | `false` | `tokenizer.json:39` |
| **fuse_unk / dropout** | `false` / `null` | `tokenizer.json:38,34` |

**What this means:**
- `NFKC` normalizes compatibility characters before tokenization.
- `Metaspace` (`▁` U+2581) marks word boundaries like SentencePiece — `▁` at token start indicates a new word (e.g., `▁भारत`).
- No byte fallback — out-of-vocab characters map to `[UNK]` rather than UTF-8 bytes.
- No truncation/padding by default — handle at model input time.

### Vocabulary Composition (10,000 tokens)

Derived by script analysis of `tokenizer.json:42-`:

| Language | Script | Tokens | Share | Example tokens (from vocab) |
|----------|--------|--------|-------|-----------------------------|
| **en** — English | Latin | **4,212** | 42.1% | `▁the` (439), `▁and` (492), `▁India` (502), `the` (417) |
| **ta** — Tamil | Tamil | **1,998** | 20.0% | `▁இந்தியா` (789), `ந்த` (383), `ம்` (396), `த்து` (896) |
| **hi** — Hindi | Devanagari | **1,429** | 14.3% | `▁भारत` (852), `भारत` (451), `▁के` (588), `ार` (415) |
| **te** — Telugu | Telugu | **1,356** | 13.6% | `భారత` (521), `▁ప్ర` (998), `ార` (449), `దేశ` (530) |
| **other/symbol** | Digits / Punct / Wiki markup | **1,005** | 10.1% | `\n` (1), `https://` (355), `](https://en.wikipedia.org/wiki/` (402), `▁[↑` (518) |

Merges distribution mirrors vocab: `en 42.8% (4129)`, `ta 20.2% (1952)`, `hi 14.0% (1353)`, `te 13.4% (1297)`, `other 9.6% (927)`.

**Base alphabet (first ~342 ids):** includes `\n`, ASCII `!`–`~`, `▁`, `₹`, `—`, `‘’“”`, and all atomic Indic characters:
- Devanagari `ँ`–`॰` (135–210)
- Tamil `ஃ`–`்` (211–256)
- Telugu `ం`–`్` (257–315)

Remaining ~9,658 entries are learned BPE merges — heavily weighted toward Wikipedia markdown patterns like `a.org/wiki/`, `Category:`, `FOOTNOTE`, `ISBN`, `archive.org/web/`, citation markers, etc.

---

## 📊 Metrics — `metrics.json`

### Overview

```json
// metrics.json:1-37
{
  "variant": "wiki_faithful_markdown",
  "languages": { "en": "English", "hi": "Hindi", "te": "Telugu", "ta": "Tamil" },
  "weights": { "en": 2, "hi": 3, "te": 6, "ta": 3 },
  "vocab_size": 10000,
  ...
}
```

| Field | Description |
|-------|-------------|
| `variant` | `wiki_faithful_markdown` — training corpus was Wikipedia dumps rendered as faithful markdown (links, refs, tables preserved, not stripped). |
| `languages` | 4 language codes with full names. |
| `weights` | Sampling weights used during BPE training to balance low-resource scripts. `te` up-weighted 6×, `hi/ta` 3×, `en` 2×. |
| `vocab_size` | `10000` — total vocab entries including specials. |
| `faithful_units` | Count of **ground-truth units** per language under `unit_policy`. |
| `token_counts` | Tokens produced by the tokenizer on the same evaluation slice. |
| `ratios` | `token_counts / faithful_units` — **fertility** (lower = more efficient compression). |
| `unit_policy` | `"Counts each contiguous Unicode letter/mark/number run as one unit and each visible non-space punctuation/symbol character as one unit."` |
| `spread` | `max(ratios) - min(ratios)` = `0.0200` — fairness gap. Lower is more equitable. |
| `score` | `49830.81` — aggregated weighted score reported by training harness (lower spread + lower weighted fertility is better). |

### Detailed Table

| Lang | Code | Weight | Faithful Units | Token Count | Ratio `tokens/units` | Tokens per 1k units |
|------|------|--------|----------------|-------------|----------------------|---------------------|
| English | `en` | 2 | 186,426 | 122,437 | **0.6567** | 657 |
| Hindi | `hi` | 3 | 88,359 | 59,203 | **0.6700** | 670 |
| Telugu | `te` | 6 | 36,293 | 23,589 | **0.6499** | 650 |
| Tamil | `ta` | 3 | 185,869 | 122,822 | **0.6607** | 661 |

**Verification** (`metrics.json:16-34`):
```
ratio = token_counts[lang] / faithful_units[lang]
spread = max(ratios) - min(ratios) = 0.6700 - 0.6499 = 0.02006
```

### Interpretation

- **Efficiency:** All ratios ~0.65–0.67 means ~65 tokens per 100 faithful units — good compression for a 10k BPE. `te` is most efficient (0.649), `hi` least (0.670) despite highest weight — reflects Telugu's agglutinative morphology vs. Hindi's shorter words.
- **Fairness:** `spread = 0.02` is **very low** (<2% absolute difference) — tokenizer is well-balanced; no language is penalized by >2% extra tokens per unit.
- **Weighting effect:** Despite `te` having only 36k faithful units (smallest corpus slice), its 6× weight ensures 13.6% vocab share, preventing English dominance.
- **Score:** `49830.80` — useful for comparing runs with same variant/weights; lower score indicates better overall weighted fertility.

> **Why not 5 languages?** Both `metrics.json:3-8` and the vocab script check show exactly 4 scripts. If you expected Kannada (`kn`), Malayalam (`ml`), or Bengali (`bn`), they are **not** in this `wiki_faithful_markdown` variant — consider training a new variant with those languages added.

---

## 🚀 Usage

### Install

```bash
pip install tokenizers transformers
```

### Load with `tokenizers` (fast, recommended)

```python
from tokenizers import Tokenizer

tok = Tokenizer.from_file("tokenizer.json")

# Encode
enc = tok.encode("भारत एक महान देश है। India is great. இந்தியா")
print(enc.tokens)
# e.g. ['▁भारत', '▁एक', '▁महान', '▁देश', '▁है', '।', '▁India', '▁is', '▁great', '.', '▁இந்தியா']
print(enc.ids)
print(f"tokens: {len(enc.tokens)}")

# Decode
print(tok.decode(enc.ids))
```

### Load with `transformers`

```python
from transformers import PreTrainedTokenizerFast

tok = PreTrainedTokenizerFast(tokenizer_file="tokenizer.json", unk_token="[UNK]")
tok.encode("తెలుగు భాష భారతదేశం")
tok.tokenize("தமிழ் மொழி")
```

### Compute fertility check (reproduce metrics)

```python
import json, re

# faithful units as per metrics.json:22 policy
unit_pat = re.compile(r'[\p{L}\p{M}\p{N}]+|[^\s\p{L}\p{M}\p{N}]', re.UNICODE)  # use `regex` lib for \p{}
# fallback: contiguous letter/mark/number OR single visible punct/symbol

def faithful_units(text: str) -> int:
    # simplified: counts letter/mark/number runs + each punct/symbol
    import unicodedata
    units = 0
    i = 0
    while i < len(text):
        c = text[i]
        cat = unicodedata.category(c)
        if cat[0] in ("L","M","N"):
            units += 1
            while i < len(text) and unicodedata.category(text[i])[0] in ("L","M","N"):
                i += 1
        elif not c.isspace():
            units += 1
            i += 1
        else:
            i += 1
    return units
```

---

## ⚙️ Training Configuration (inferred)

- **Corpus:** Wikipedia faithful markdown (`variant: wiki_faithful_markdown`)
- **Languages:** `en`, `hi`, `te`, `ta` with weights `2:3:6:3` to counter corpus imbalance (en/ta ~186k units vs te 36k units)
- **Vocab:** 10k BPE, 9,658 merges, NFKC + Metaspace
- **Evaluation slice:** Same faithful units counted for ratio calculation — ensures apples-to-apples fertility measurement

---

## 📌 Notes & Limitations

- Small vocab (10k) favors compactness over rare-word coverage — expect more splits for named entities, highly inflected forms, and code-mixed text.
- No byte fallback — unseen scripts (e.g., Bengali, Kannada) will produce `[UNK]` (id 0). Extend vocab or enable `byte_fallback` if you add languages.
- Wiki markup tokens (`https://`, `Category:`, `FOOTNOTE`, `ISBN_(identifier)`) inflate vocab — intentional for faithful markdown; strip markup pre-tokenization if you want pure linguistic tokens.
- Metrics reflect a specific eval slice (sums ~496k faithful units total) — re-evaluate on your domain (news, social, transliterated) before production.

---

## 📄 License & Citation

If you use this tokenizer, please cite:

```
Indic-Token-izer — BPE 10k, wiki_faithful_markdown variant
Languages: en, hi, te, ta | Vocab: 10000 | Weights en:2 hi:3 te:6 ta:3
https://github.com/TharunSivamani/Indic-Token-izer
```

---

## 🙏 Acknowledgments

- Built with [Hugging Face `tokenizers`](https://github.com/huggingface/tokenizers)
- Corpus: Wikipedia dumps (faithful markdown rendering)
- Metrics policy: contiguous `Letter/Mark/Number` run = 1 unit, each visible punctuation/symbol = 1 unit
