# Advanced Script Detector – Technical Guide

A complete reference for \`\` (v 2.0). Install, import, or run the detector from the command line and extend it with new scripts.

---

## 1. Module overview

| Feature | Details |
| ------- | ------- |
|         |         |
| **Supported scripts** | Devanagari `hi`, Gujarati `gu`, Gurmukhi `pa`, Bengali `bn`, Odia `or`, Tamil `tam`, Telugu `te`, Kannada `kn`, Malayalam `ml`, Urdu/Arabic `ur`, ASCII/Latin `en` |
| **Algorithm**         | Unicode‑range lookup on up to *N* significant code‑points (default = 6) → deterministic, no ML                                                                     |
| **Speed**             | \~10–15 M detections/sec; column‑wise unique‑value scanning to avoid repeats                                                                                       |
| **Confidence score**  | proportion ∈ [0, 1] of inspected characters that match winning script                                                                                              |
| **Cache**             | word → lang (JSON). Reads once, writes once per DataFrame. O(1) lookup.                                                                                            |
| **I/O helpers**       | Strings, pandas DataFrames, CSV/XLS(X) (with chunking), folders, dict/JSON                                                                                         |
| **CLI driver**        | `python script_detector.py file.csv --cols Name …`                                                                                                                 |
| **Extensibility**     | Edit `UNICODE_RANGES` or subclass `ScriptDetector`                                                                                                                 |

---

## 2. Installation

```bash
# clone repo
git clone https://github.com/amitportal/language_detector
cd script‑detector

# optional: create venv
python -m venv .venv && source .venv/bin/activate

# install runtime deps (only pandas is optional but recommended)
pip install pandas openpyxl  # openpyxl needed for .xlsx
```

> *No compiled wheels, pure Python 3.8 +.*

---

## 3. Quick start

### 3.1 Single string

```python
from script_detector import ScriptDetector

det = ScriptDetector()
print(det.detect("कुमार"))      # ('hi', 1.0)
print(det("سعید"))              # alias → ('ur', 1.0)
```

### 3.2 DataFrame columns (vectorised)

```python
import pandas as pd
from script_detector import ScriptDetector

df = pd.read_csv("1_Abdasa_elector_data.csv")
cols = ["Name", "Relative_Name", "lastname", "rel_lastname"]

ScriptDetector(cache_file="lang_cache.json").annotate_frame(df, cols, auto_cache=True)
```

### 3.3 Annotate file – creates sibling `<input>_lang.csv`

```python
from script_detector import ScriptDetector

det = ScriptDetector(sample_chars=4)  # faster, still accurate for names

det.annotate_file(
    "1_Abdasa_elector_data.csv",
    columns=["Name", "Relative_Name", "lastname", "rel_lastname"],
    auto_cache=True,
    min_cache_score=0.95,
)
```

### 3.4 Folder recursion

```python
ScriptDetector().annotate_folder("./rolls", columns=["Name"], patterns=["*.csv"])
```

### 3.5 Streaming a 2 GB CSV

```python
ScriptDetector().annotate_file(
    "mega_roll.csv",
    ["Name"],
    chunksize=100_000,  # pandas chunk streaming
)
```

### 3.6 JSON / dict

```python
record = {"Name": "Ravi", "Comment": "नमस्ते"}
print(ScriptDetector().annotate_json(record, ["Comment"]))
```

### 3.7 CLI

```bash
python script_detector.py 1_Abdasa_elector_data.csv --cols Name Relative_Name \
       --cache cache.json --sample 5 --threshold 0.9
```

---

## 4. API reference

### 4.1 `ScriptDetector` constructor

| param          | type  | default | meaning                                |        |                 |
| -------------- | ----- | ------- | -------------------------------------- | ------ | --------------- |
| `sample_chars` | `int` | `6`     | max significant code‑points per string |        |                 |
| `default_code` | `str` | `'en'`  | returned when nothing recognised       |        |                 |
| `cache_file`   | \`str | Path    | None\`                                 | `None` | JSON cache path |

### 4.2 Core methods

| method                                          | returns         | notes                    |
| ----------------------------------------------- | --------------- | ------------------------ |
| `detect(text)`                                  | `(lang, score)` | fast single‑string check |
| `annotate_frame(df, cols, **kw)`                | `DataFrame`     | adds `<col>_lang`        |
| `annotate_file(path, cols, **kw)`               | `Path`          | writes annotated copy    |
| `annotate_folder(folder, cols, patterns, **kw)` | `list[Path]`    | recursive                |
| `annotate_json(record, keys, **kw)`             | `dict`          | non‑destructive          |

> **Kwargs** `auto_cache`, `min_cache_score`, `chunksize` propagate through helpers.

---

## 5. Adding a new script

```python
from script_detector import UNICODE_RANGES
# add Sinhala (ISO "si") block U+0D80–0DFF
UNICODE_RANGES["si"] = [range(0x0D80, 0x0E00)]
```

No code reload needed if done before instantiation.

---

## 6. Performance tuning

| lever               | effect                                                            |
| ------------------- | ----------------------------------------------------------------- |
| `sample_chars` ↓    | faster, but leading numerals/emojis may mislead                   |
| column‑wise uniques | built‑in; ensure strings not categorical ← pandas handles         |
| cache warm‑up       | run once with `auto_cache=True`; subsequent runs ≈ 50‑70 % faster |
| `chunksize`         | stream huge CSVs without RAM blow‑up                              |

---

## 7. Error handling & edge‑cases

- Unknown/mixed scripts → returns `default_code` with score 0.
- Corrupt cache JSON → silently ignored, new cache started.
- Excel chunking not supported (pandas limitation).
- Strings containing only whitespace/control chars → `default_code`.

---

## 8. Tests (pytest stubs)

```python
import pytest, pandas as pd
from script_detector import ScriptDetector

det = ScriptDetector(sample_chars=4)

@pytest.mark.parametrize("word, expected", [
    ("राम", "hi"), ("سعید", "ur"), ("Ravi", "en"),
])
def test_detect(word, expected):
    lang, score = det.detect(word)
    assert lang == expected and score > 0.9

def test_dataframe():
    df = pd.DataFrame({"Name": ["राम", "Ravi"]})
    out = det.annotate_frame(df, ["Name"])
    assert set(out["Name_lang"]) == {"hi", "en"}
```

---

## 9. Changelog

- **v 2.0** – vectorised unique‑value detection, batched cache flush, CLI rewrite.
- **v 1.x** – initial release with hi/ur/gu/tam/te/kn/ml/en.

---

## 10. License & attribution

© 2025 Amit Kumar. MIT License. Unicode block data sourced from the Unicode Consortium (© Unicode).

