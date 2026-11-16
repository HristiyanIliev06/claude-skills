# Format Reference

Per-format generation guidance, library choices, gotchas, and minimal code snippets.
Read only the section(s) relevant to the current request.

---

## Table of Contents
1. [JSON](#json)
2. [JSONL (JSON Lines)](#jsonl)
3. [CSV](#csv)
4. [TSV](#tsv)
5. [XML](#xml)
6. [YAML](#yaml)
7. [TOML](#toml)
8. [Parquet](#parquet)
9. [Excel (.xlsx)](#excel)
10. [TXT (plain text / unstructured)](#txt)
11. [Markdown (.md)](#markdown)
12. [DOCX](#docx)
13. [SQLite (.db)](#sqlite)
14. [HDF5 (.h5)](#hdf5)
15. [Pickle (.pkl)](#pickle)

---

## JSON

**Library**: `json` (stdlib) for simple; `orjson` for speed/binary.

**Gotchas**:
- `json.dumps` doesn't serialize `datetime` by default — use `default=str` or a custom encoder.
- NaN / Infinity are not valid JSON — replace with `null` or a sentinel string.
- Large files: prefer JSONL over a single giant JSON array.

**Patterns**:
```python
import json, random
from datetime import datetime, timezone

records = [
    {
        "id": i,
        "name": f"User_{i}",
        "score": round(random.uniform(0, 100), 2),
        "created_at": datetime.now(timezone.utc).isoformat(),
        "tags": random.choices(["ml", "data", "api", "web"], k=random.randint(0, 3)),
    }
    for i in range(1, n + 1)
]

with open("output.json", "w") as f:
    json.dump(records, f, indent=2, default=str)
```

**Nesting depth**: for deeply nested structures, build helper functions per sub-object
rather than one huge comprehension. Annotate structure with comments above the output.

**Schema annotation**: optionally write a companion `schema.json` using JSON Schema
format when the user needs to validate against it later.

---

## JSONL

**Library**: `json` (stdlib) — write one JSON object per line, no trailing comma.

**When to use**: streaming pipelines, Hugging Face datasets, LLM fine-tuning corpora,
Spark/Flink ingestion, large files that shouldn't be loaded fully into RAM.

```python
with open("output.jsonl", "w") as f:
    for record in records:
        f.write(json.dumps(record, default=str) + "\n")
```

**Reading one-liner**: `pd.read_json("output.jsonl", lines=True)`

---

## CSV

**Library**: `csv` (stdlib) or `pandas` for complex cases.

**Gotchas**:
- Always specify `newline=""` when opening file for `csv.writer` on Windows targets.
- Quote style: default is `QUOTE_MINIMAL`; use `QUOTE_ALL` if fields contain commas.
- Encoding: default UTF-8; specify `encoding="utf-8-sig"` if output is for Excel on Windows.
- First row: always include a header row unless the user says otherwise.
- Floats: control precision with `round()` before writing; CSV stores as text.

```python
import csv, random

headers = ["id", "name", "age", "score", "active"]
with open("output.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=headers)
    writer.writeheader()
    for i in range(1, n + 1):
        writer.writerow({
            "id": i,
            "name": f"Record_{i}",
            "age": random.randint(18, 80),
            "score": round(random.gauss(70, 15), 2),
            "active": random.choice([True, False]),
        })
```

**Reading one-liner**: `pd.read_csv("output.csv")`

**Delimiter variants**: TSV → `delimiter="\t"`, pipe → `delimiter="|"`.

---

## TSV

Same as CSV with `delimiter="\t"`. Extension `.tsv`. Common in bioinformatics.
Reading: `pd.read_csv("output.tsv", sep="\t")`

---

## XML

**Library**: `xml.etree.ElementTree` (stdlib) or `lxml` for namespaces/XPath.

**Gotchas**:
- Attribute values must be strings — cast all non-string values explicitly.
- Special chars (`<`, `>`, `&`) are auto-escaped by ElementTree; don't double-escape.
- No standard for arrays — wrap repeated elements in a container element.
- Namespace declarations can be verbose; omit unless user specifically requests them.

```python
import xml.etree.ElementTree as ET

root = ET.Element("records")
for i in range(1, n + 1):
    record = ET.SubElement(root, "record")
    ET.SubElement(record, "id").text = str(i)
    ET.SubElement(record, "name").text = f"Item_{i}"
    ET.SubElement(record, "value").text = str(round(random.uniform(0, 100), 2))

tree = ET.ElementTree(root)
ET.indent(tree, space="  ")  # Python 3.9+
tree.write("output.xml", encoding="unicode", xml_declaration=True)
```

---

## YAML

**Library**: `pyyaml` (`pip install pyyaml`).

**Gotchas**:
- `yaml.dump` uses block style by default; pass `default_flow_style=False` for readability.
- Booleans: YAML parses bare `yes`/`no`/`on`/`off` as booleans — quote them as strings if intended as strings.
- Dates: YAML auto-parses ISO dates; wrap in quotes if you want them as raw strings.
- Multiline strings: use `|` (literal block) or `>` (folded block) for long text fields.

```python
import yaml, random

data = [{"id": i, "name": f"User_{i}", "score": round(random.random() * 100, 2)}
        for i in range(1, n + 1)]

with open("output.yaml", "w") as f:
    yaml.dump(data, f, default_flow_style=False, allow_unicode=True)
```

---

## TOML

**Library**: `tomllib` (read, stdlib Python 3.11+) / `tomli-w` (write: `pip install tomli-w`).

**When to use**: config files, not large datasets. TOML is not suited for arrays
of records with >~50 entries — prefer YAML or JSON for those.

```python
import tomli_w

config = {
    "model": {"learning_rate": 0.001, "epochs": 50, "device": "cuda"},
    "data": {"train_path": "data/train.csv", "batch_size": 32},
}
with open("config.toml", "wb") as f:
    tomli_w.dump(config, f)
```

---

## Parquet

**Library**: `pyarrow` or `pandas` with pyarrow backend.
Install: `pip install pyarrow pandas --break-system-packages`

**When to use**: large datasets (>10k rows), typed columnar data, ML pipelines.

**Gotchas**:
- Column types must be consistent — mixed types in a column will error or coerce.
- Nested structs are supported; use `pyarrow.struct` for complex schemas.
- Partitioned writes: use `pyarrow.parquet.write_to_dataset` for partitioned output.
- Snappy compression is default and widely supported; use `compression="gzip"` for
  better compression ratio or `"none"` for debugging.

```python
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq

df = pd.DataFrame({
    "id": range(1, n + 1),
    "value": [round(random.gauss(50, 10), 4) for _ in range(n)],
    "category": random.choices(["A", "B", "C"], k=n),
    "timestamp": pd.date_range("2024-01-01", periods=n, freq="1min"),
})
df.to_parquet("output.parquet", index=False, compression="snappy")
```

**Reading one-liner**: `pd.read_parquet("output.parquet")`

---

## Excel

**Library**: `openpyxl` (`.xlsx`) — `pip install openpyxl --break-system-packages`
For reading XLS (legacy): `xlrd`. For writing with formatting: `openpyxl` directly.

**Gotchas**:
- `pd.to_excel` requires `openpyxl` engine for `.xlsx`.
- Sheet names max 31 chars, no `[]:*?/\` chars.
- Excel stores dates as floats internally — always write Python `datetime` objects,
  not ISO strings, for proper date cell type.
- For multi-sheet files, use `pd.ExcelWriter` as context manager.

```python
import pandas as pd

with pd.ExcelWriter("output.xlsx", engine="openpyxl") as writer:
    df_users.to_excel(writer, sheet_name="Users", index=False)
    df_orders.to_excel(writer, sheet_name="Orders", index=False)
```

**Adding formatting** (column widths, header bold): use `openpyxl` directly after
writing. Read `openpyxl` docs or ask the user if they need it.

---

## TXT

**When to use**: unstructured text corpora, NLP test data, log files, prose datasets.

**Patterns**:

*Log file*:
```python
import random
from datetime import datetime, timedelta

levels = ["INFO", "WARNING", "ERROR", "DEBUG"]
base_time = datetime(2024, 1, 1, 0, 0, 0)
with open("output.log", "w") as f:
    for i in range(n):
        ts = (base_time + timedelta(seconds=i * random.randint(1, 30))).isoformat()
        level = random.choices(levels, weights=[60, 20, 10, 10])[0]
        f.write(f"[{ts}] {level}: Event message {i} from source_module_{random.randint(1,5)}\n")
```

*Prose corpus* (for NLP/tokenizer testing): generate variable-length paragraphs using
lorem ipsum or domain-specific word banks. Include intentional misspellings,
Unicode characters, emoji, and empty lines for robustness testing.

*Edge case strings for adversarial TXT*: See `references/profiles.md` → Adversarial Strings.

---

## Markdown

**When to use**: documentation test corpora, RAG pipeline testing, markdown parser testing.

Generate with varying:
- Heading depths (H1–H6)
- Table sizes and alignment
- Code block languages
- Nested lists (ordered + unordered)
- Link and image syntax (real and broken)
- Frontmatter (YAML block at top) for static site generator testing

```python
def make_markdown_doc(title, n_sections=5, n_rows=10):
    lines = [f"# {title}\n"]
    for s in range(1, n_sections + 1):
        lines.append(f"\n## Section {s}\n")
        lines.append("Some paragraph text with **bold** and _italic_ content.\n")
        # Table
        lines.append("| Column A | Column B | Column C |\n|---|---|---|\n")
        for r in range(n_rows):
            lines.append(f"| Val_{s}_{r}_A | {random.randint(0,100)} | {random.choice(['x','y','z'])} |\n")
        # Code block
        lines.append(f"\n```python\nresult = compute({s})\nprint(result)\n```\n")
    return "".join(lines)
```

---

## DOCX

**Library**: `python-docx` — `pip install python-docx --break-system-packages`

**Also**: read the `docx` skill (`/mnt/skills/public/docx/SKILL.md`) for full
formatting guidance if the user wants styled output (fonts, tables, headings, etc.).

**Quick generation**:
```python
from docx import Document

doc = Document()
doc.add_heading("Test Document", level=1)
for i in range(n):
    doc.add_heading(f"Section {i+1}", level=2)
    doc.add_paragraph(f"This is paragraph {i+1} with some generated content.")
doc.save("output.docx")
```

For structured data in DOCX (tables), inject rows programmatically:
```python
table = doc.add_table(rows=1, cols=3)
table.style = "Table Grid"
hdr = table.rows[0].cells
hdr[0].text, hdr[1].text, hdr[2].text = "ID", "Name", "Value"
for i in range(n):
    row = table.add_row().cells
    row[0].text, row[1].text, row[2].text = str(i), f"Item_{i}", str(round(random.random()*100,2))
doc.save("output.docx")
```

---

## SQLite

**Library**: `sqlite3` (stdlib).

**When to use**: multi-table relational test data with FK constraints, SQL query testing,
ORM testing (SQLAlchemy, Django ORM).

```python
import sqlite3, random

conn = sqlite3.connect("output.db")
cur = conn.cursor()

cur.execute("""CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE
)""")
cur.execute("""CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    amount REAL,
    status TEXT
)""")

for i in range(1, n_users + 1):
    cur.execute("INSERT INTO users VALUES (?, ?, ?)", (i, f"User_{i}", f"user{i}@example.com"))
    for _ in range(random.randint(0, 5)):
        cur.execute("INSERT INTO orders (user_id, amount, status) VALUES (?, ?, ?)",
                    (i, round(random.uniform(5, 500), 2), random.choice(["pending", "shipped", "cancelled"])))

conn.commit()
conn.close()
```

---

## HDF5

**Library**: `h5py` — `pip install h5py --break-system-packages`

**When to use**: large numerical arrays, ML weight checkpoints, scientific data.

```python
import h5py, numpy as np

with h5py.File("output.h5", "w") as f:
    f.create_dataset("features", data=np.random.randn(n_samples, n_features).astype(np.float32))
    f.create_dataset("labels", data=np.random.randint(0, n_classes, size=n_samples))
    f.attrs["description"] = "Synthetic classification dataset"
    f.attrs["seed"] = 42
```

---

## Pickle

**Library**: `pickle` (stdlib). Use `joblib` for numpy arrays (more efficient).

**When to use**: Python-specific test fixtures, caching ML model outputs, serialized
sklearn pipelines. **Warning**: pickle is not portable across Python versions or
architectures. Never use for long-term storage or cross-language exchange.

```python
import pickle

data = {"X_train": ..., "y_train": ..., "X_test": ..., "y_test": ...}
with open("fixtures.pkl", "wb") as f:
    pickle.dump(data, f, protocol=pickle.HIGHEST_PROTOCOL)
```
