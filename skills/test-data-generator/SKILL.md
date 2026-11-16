---
name: test-data-generator
description: >
  Generates synthetic test/fixture data files in any format (JSON, CSV, JSONL, XML,
  YAML, TOML, Parquet, Excel, DOCX, TXT, SQLite, and more) with fully customizable
  content: shape, size, data types, distributions, validity conditions, edge cases,
  and schema constraints. Use whenever the user asks to generate test data, create
  sample files, make fixture data, produce synthetic datasets, generate mock records,
  create files for pipeline testing, stress test with edge cases, test Pydantic
  validators with invalid inputs, or says things like "give me 100 rows of", "create
  a CSV with", "make a JSON file that", "generate bad data for", "I need dummy data",
  "make a file I can load with pandas", or "build a dataset with X shape". Always use
  this skill even for casual requests like "make me a test file" — never freehand
  generate data files without it. Covers all file formats, ML datasets, API payloads,
  boundary testing, and multi-file batch generation.
---

# Test Data Generator Skill

This skill governs all synthetic data file generation. Read it fully before producing
any test/fixture/sample data file, regardless of format or complexity.

---

## Mental Model

Every data generation request has four axes. Identify all four before writing any code
or file content:

| Axis | Question | Examples |
|------|----------|---------|
| **Format** | What file type(s) are needed? | JSON, CSV, DOCX, Parquet... |
| **Shape** | What is the structure of records? | Flat, nested, multi-table, time-series... |
| **Population** | What values fill the fields? | Distributions, ranges, patterns, enums... |
| **Validity profile** | What mix of valid/invalid/edge data? | All valid, % corrupt, boundary-only... |

If the user hasn't specified all four, infer from context and state your assumptions
explicitly before generating. Do not silently guess.

---

## When NOT to Use This Skill

Skip this skill and generate directly if ALL of the following are true:
- Single file, single format
- ≤ 30 rows/records
- Schema is simple and fully stated by the user (no inference needed)
- Validity is implicitly all-valid (no mention of bad data, edge cases, or testing)
- No distribution, annotation, or reproducibility requirements

Examples that should bypass this skill:
- "Give me 20 rows of user data as CSV"
- "Make a small JSON with 5 product records"
- "Create a sample config.toml for my training run"

Examples that must use this skill:
- "Generate 500 rows with 20% invalid emails and null storms"
- "Make a mixed-validity dataset for testing my Pydantic model"
- "Give me adversarial strings in the name field"
- "I need reproducible fixtures with a fixed seed"
- Any request involving multiple files, profiles, distributions, or fault injection

---

## Workflow

### Step 1 — Clarify (if needed)
If the request is thin, ask **one targeted question** — the most blocking unknown.
Never ask multiple questions at once. If you can infer, infer and state the assumption.

Common inference rules:
- No format specified + "load with pandas/polars" → default to CSV
- No size specified → default to 20 rows / small file (comment that user can scale)
- No validity profile → default to all-valid unless "test edge cases" / "bad data" mentioned
- No schema specified → infer from field names or domain (e.g. "user records" → name, email, age)

### Step 2 — Declare the generation plan
Before writing file content or code, output a compact plan block:

```
FORMAT:   CSV
SHAPE:    flat, 5 columns (id, name, email, age, score)
SIZE:     50 rows
PROFILE:  80% valid | 10% missing values | 10% type violations
SEED:     42 (reproducible)
```

### Step 3 — Generate
Pick the right generation mode:

- **Direct output** (≤ ~30 rows, simple format): embed the file content in a code block,
  then save it to `/mnt/user-data/outputs/` via `create_file` or bash.
- **Script mode** (> ~30 rows, complex schema, Parquet/DOCX/Excel, or batch):
  write a Python generation script, run it with `bash_tool`, save outputs.

See `references/formats.md` for format-specific generation guidance and library choices.
See `references/profiles.md` for validity profile patterns and injection strategies.

### Step 4 — Present
Always present the output file(s) with `present_files`. Include a brief summary:
- What was generated (format, row count, schema)
- What validity conditions are present and how they're marked/distributed
- How to load the file (one-liner code snippet using pandas/polars/json/etc.)

---

## Shape Vocabulary

Use these terms consistently when interpreting user requests:

| Shape | Meaning | Example |
|-------|---------|---------|
| **flat** | Single table, no nesting | CSV with scalar columns |
| **nested** | Records contain sub-objects/arrays | JSON with address object inside user |
| **wide** | Many columns (50+), few rows | Feature matrix |
| **tall** | Many rows (10k+), few columns | Event log |
| **sparse** | Many nulls/zeros, intentionally | Recommendation matrix |
| **ragged** | Variable-length records | JSON arrays of unequal length |
| **multi-table** | Relational: multiple linked files | users.csv + orders.csv with FK |
| **time-series** | Timestamped sequential records | sensor readings, price history |
| **graph** | Nodes + edges | edges.csv + nodes.csv |

---

## Validity Profiles — Quick Reference

Full injection patterns are in `references/profiles.md`. Summary:

| Profile name | Description |
|-------------|-------------|
| `all_valid` | 100% conforming records — the "happy path" |
| `all_invalid` | 100% broken records — stress tests rejection logic |
| `boundary` | Values at exact min/max/zero/empty edges |
| `mixed` | Configurable % valid + % of each fault type |
| `adversarial` | SQL injection, XSS strings, Unicode bombs, huge values |
| `schema_drift` | Wrong column names, extra/missing fields |
| `type_chaos` | Strings in numeric fields, booleans as ints, etc. |
| `temporal` | Out-of-order timestamps, future dates, epoch=0 |
| `null_storm` | High null density; tests None/NaN handling |
| `duplicate_heavy` | Many exact and near-duplicate rows |

When generating `mixed`, annotate injected faults in a sidecar column (e.g. `_fault_type`)
or in a companion `faults.json` manifest — unless the user explicitly wants unmarked data.

---

## Size Reference

| Label | Rows | Use case |
|-------|------|----------|
| tiny | 5–20 | Quick visual inspection, REPL paste |
| small | 50–200 | Unit tests, schema validation |
| medium | 1k–10k | Integration tests, pandas benchmarks |
| large | 100k–1M | Load tests, Parquet/chunked reading |
| XL | 1M+ | Streaming pipeline tests — always use script mode |

For large/XL, always generate via script with chunked writing. Never try to embed
large file content inline.

---

## Inferring Schema from Domain

If no schema is provided, map domain keywords to standard field sets:

| Domain keyword | Inferred fields |
|---------------|----------------|
| user / account | id, name, email, age, created_at, is_active |
| product / item | id, name, category, price, stock, sku |
| order / transaction | order_id, user_id, product_id, quantity, total, status, created_at |
| event / log | timestamp, level, message, source, trace_id |
| sensor / IoT | device_id, timestamp, value, unit, battery_level |
| ML dataset | feature_1..N, label, split (train/val/test) |
| employee / HR | employee_id, name, department, salary, hire_date, manager_id |
| address / geo | street, city, state, country, postal_code, lat, lon |

Always show the inferred schema in the plan block and invite correction.

---

## Multi-file / Relational Generation

When the user requests multi-table data:
1. Identify the entity hierarchy (parent → child)
2. Generate parent table first, extract PKs
3. Inject PKs as FKs into child table with configurable fan-out (e.g. 1–5 orders per user)
4. Output all files in a named subfolder: `outputs/dataset_name/`
5. Include a `README.txt` in the folder describing the schema and relationships

---

## Reproducibility

Always accept and propagate a `seed` parameter. Default seed = 42.
For script mode, set `random.seed(seed)` and `numpy.random.seed(seed)` at the top.
Document the seed in the plan block and in the output file's header comment (where format allows).

---

## Reference Files

Read these before generating when applicable:

- `references/formats.md` — Format-specific libs, gotchas, and code snippets for each
  file type (JSON, CSV, JSONL, XML, YAML, TOML, Parquet, Excel, DOCX, TXT, Markdown)
- `references/profiles.md` — Validity profile injection patterns, fault taxonomy,
  sidecar annotation conventions, and adversarial string library
