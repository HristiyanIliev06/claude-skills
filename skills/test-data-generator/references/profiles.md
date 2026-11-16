# Validity Profiles Reference

Patterns for injecting faults, edge cases, boundary values, and adversarial data
into generated files. Read the relevant section before applying a profile.

---

## Table of Contents
1. [Annotation Convention](#annotation-convention)
2. [Profile: all_valid](#all_valid)
3. [Profile: all_invalid](#all_invalid)
4. [Profile: boundary](#boundary)
5. [Profile: mixed](#mixed)
6. [Profile: adversarial](#adversarial)
7. [Profile: schema_drift](#schema_drift)
8. [Profile: type_chaos](#type_chaos)
9. [Profile: temporal](#temporal)
10. [Profile: null_storm](#null_storm)
11. [Profile: duplicate_heavy](#duplicate_heavy)
12. [Fault Taxonomy](#fault-taxonomy)
13. [Adversarial String Library](#adversarial-string-library)
14. [Distribution Reference](#distribution-reference)

---

## Annotation Convention

When generating `mixed` or multi-profile data, annotate faults in a sidecar column
`_fault_type` with values from the Fault Taxonomy below, unless the user explicitly
requests **unmarked data** (e.g. "I want to test if my parser catches these itself").

For formats that don't support extra columns (XML, TXT, DOCX):
- Append a `<!-- FAULT: ... -->` comment per record (XML)
- Write a companion `faults.jsonl` manifest: `{"line": N, "fault": "..."}`
- State the annotation method in the plan block

**Sidecar column values**:
```
_fault_type = ""              # valid record
_fault_type = "null_field"    # one or more required fields are null
_fault_type = "type_mismatch" # wrong type in a field
_fault_type = "out_of_range"  # value violates a range constraint
_fault_type = "boundary_min"  # value is exactly at minimum
_fault_type = "boundary_max"  # value is exactly at maximum
_fault_type = "boundary_zero" # value is exactly zero
_fault_type = "boundary_empty"# empty string / zero-length value
_fault_type = "duplicate"     # row is a duplicate of a previous row
_fault_type = "schema_extra"  # extra unexpected field present
_fault_type = "schema_missing"# expected field absent
_fault_type = "adversarial"   # injection / unicode / overflow string
_fault_type = "temporal_ooo"  # out-of-order timestamp
_fault_type = "temporal_future"# timestamp is in the future
_fault_type = "temporal_epoch"# timestamp is unix epoch (1970-01-01)
_fault_type = "mixed:<type>"  # multiple faults, list them: "mixed:null+type_mismatch"
```

---

## all_valid

Generate 100% conforming records. This is the default profile.

Rules:
- All required fields present and non-null
- All values within declared ranges/enums
- All types match schema
- No duplicate primary keys
- Timestamps in logical order and in the past (relative to generation time)

```python
def is_valid_record(r, schema):
    # Each field present, correct type, in range
    return all(
        r.get(f["name"]) is not None
        and isinstance(r[f["name"]], f["type"])
        and f.get("min", -inf) <= r[f["name"]] <= f.get("max", inf)
        for f in schema["fields"]
    )
```

---

## all_invalid

Every record has at least one fault. Use for testing rejection/error pipelines.

Strategy: cycle through fault types so the dataset covers the full fault taxonomy,
rather than randomly picking the same fault repeatedly.

```python
import itertools

FAULT_CYCLE = itertools.cycle([
    "null_field", "type_mismatch", "out_of_range",
    "schema_extra", "adversarial", "duplicate"
])

def inject_fault(record, fault_type, schema):
    r = record.copy()
    if fault_type == "null_field":
        field = random.choice(schema["required_fields"])
        r[field] = None
    elif fault_type == "type_mismatch":
        field = random.choice(schema["numeric_fields"])
        r[field] = "not_a_number"
    elif fault_type == "out_of_range":
        field = random.choice(schema["bounded_fields"])
        r[field] = schema[field]["max"] * 100  # wildly out of range
    elif fault_type == "schema_extra":
        r["__unexpected_field__"] = "surprise"
    elif fault_type == "adversarial":
        field = random.choice(schema["string_fields"])
        r[field] = random.choice(ADVERSARIAL_STRINGS)
    elif fault_type == "duplicate":
        pass  # caller handles by repeating a previous record
    r["_fault_type"] = fault_type
    return r
```

---

## boundary

Records specifically test the exact edges of constraints.
Produce one record per boundary condition per constrained field.

Boundary types to cover per field:
- Numeric: `min`, `max`, `min-1` (just below), `max+1` (just above), `0`, `-1`
- String: `""` (empty), `" "` (whitespace-only), `max_length`, `max_length+1` char
- Enum: each valid value + one invalid value
- Nullable: `None` for optional fields; presence of `None` for required fields
- Array: `[]` (empty), `[single_item]`, `[max_items]`, `[max_items+1]`

Format:
```
id, field, value, expected_outcome, _fault_type
1, age, 0, reject, boundary_zero
2, age, 18, accept, boundary_min
3, age, 120, accept, boundary_max
4, age, 121, reject, out_of_range
5, name, "", reject, boundary_empty
6, name, "A", accept, boundary_min
```

---

## mixed

Configurable split across valid and fault categories.

Default split (if not specified by user):
- 70% valid
- 10% null_field
- 8% type_mismatch
- 7% out_of_range
- 5% adversarial

```python
def mixed_profile(n, split=None):
    if split is None:
        split = {
            "valid": 0.70,
            "null_field": 0.10,
            "type_mismatch": 0.08,
            "out_of_range": 0.07,
            "adversarial": 0.05,
        }
    # Validate split sums to 1.0
    assert abs(sum(split.values()) - 1.0) < 1e-9, "Split must sum to 1.0"
    
    labels = random.choices(
        list(split.keys()),
        weights=list(split.values()),
        k=n
    )
    records = []
    for label in labels:
        r = make_valid_record()
        if label != "valid":
            r = inject_fault(r, label, schema)
        records.append(r)
    random.shuffle(records)  # don't cluster faults at the end
    return records
```

---

## adversarial

Strings designed to break parsers, trigger injections, cause encoding errors, or
overflow buffers. Apply to string fields (names, emails, notes, search queries).

See **Adversarial String Library** below for the full list.

Strategy:
- Replace 1–3 string fields per record with adversarial strings
- Cycle through all adversarial categories across records
- Always set `_fault_type = "adversarial"`

---

## schema_drift

Simulates producer/consumer schema mismatch — common in real data pipelines.

Fault variants:
- **renamed field**: `user_id` → `userId` or `uid`
- **extra field**: add an unexpected column not in the schema
- **missing field**: omit a field entirely (even a required one)
- **reordered columns**: shuffle column order (for CSV, TSV)
- **type changed**: field that was int is now string (e.g. `"42"` instead of `42`)
- **nested flattened**: `address.city` becomes `address_city` (snake-case flattened)

```python
DRIFT_TYPES = ["rename", "extra", "missing", "reorder", "type_change", "flatten"]

def apply_drift(record, drift_type, schema):
    r = record.copy()
    if drift_type == "rename":
        old = random.choice(list(r.keys()))
        r[old + "_v2"] = r.pop(old)
    elif drift_type == "extra":
        r["legacy_field"] = "unexpected_value"
    elif drift_type == "missing":
        del r[random.choice(schema["fields"])]
    elif drift_type == "type_change":
        field = random.choice(schema["numeric_fields"])
        r[field] = str(r[field])  # int → string
    elif drift_type == "flatten":
        if "address" in r and isinstance(r["address"], dict):
            for k, v in r["address"].items():
                r[f"address_{k}"] = v
            del r["address"]
    r["_fault_type"] = f"schema_drift:{drift_type}"
    return r
```

---

## type_chaos

Every record has a type violation in at least one field. Useful for testing
type coercion logic, Pydantic validators, pandas `dtype` handling.

Type violation matrix:

| Expected type | Injected as |
|--------------|-------------|
| int | `"42"`, `42.9`, `True`, `None`, `"NaN"` |
| float | `"3.14"`, `"inf"`, `"NaN"`, `None`, `True` |
| str | `42`, `[]`, `{}`, `None`, `True` |
| bool | `"true"`, `1`, `0`, `"yes"`, `None` |
| datetime | `"not-a-date"`, `"2024-13-45"`, `1234567890` (epoch int), `None` |
| list | `"item1,item2"`, `{}`, `None`, `"[]"` (string) |
| dict | `"key:value"`, `None`, `[]`, `"null"` |

---

## temporal

Tests timestamp handling edge cases.

Variants:
- `epoch_zero`: `1970-01-01T00:00:00Z` — often signals uninitialized datetime
- `far_future`: `2099-12-31T23:59:59Z` — test upper bound handling
- `out_of_order`: timestamps that decrease in a sequence that should be ascending
- `timezone_mix`: mix of UTC, naive, and local-timezone timestamps in same column
- `dst_ambiguous`: timestamps in the "fall back" hour (e.g. `2024-11-03T01:30:00`)
- `leap_second`: `2016-12-31T23:59:60Z` (not valid in Python's datetime; tests parsers)
- `negative_unix`: negative POSIX timestamp (before 1970)
- `microsecond_precision`: timestamps that differ only in microseconds

```python
from datetime import datetime, timezone, timedelta

TEMPORAL_VARIANTS = {
    "epoch_zero":    datetime(1970, 1, 1, tzinfo=timezone.utc),
    "far_future":    datetime(2099, 12, 31, 23, 59, 59, tzinfo=timezone.utc),
    "negative_unix": datetime(1969, 12, 31, tzinfo=timezone.utc),
}
```

---

## null_storm

High density of `None`/`NaN`/empty values. Use for testing:
- `pd.DataFrame.dropna()` / `fillna()` logic
- Pydantic `Optional` field handling
- SQL `NULL` propagation
- Feature imputation in ML pipelines

```python
NULL_DENSITY = 0.4  # 40% of values are null

def null_storm_record(record, density=NULL_DENSITY):
    r = record.copy()
    for key in r:
        if key.startswith("_"):
            continue
        if random.random() < density:
            r[key] = None  # or float("nan") for numeric fields
    r["_fault_type"] = "null_storm"
    return r
```

---

## duplicate_heavy

Dataset where many rows are exact or near-duplicates. Tests deduplication logic.

Variants:
- **exact duplicate**: identical row repeated N times
- **near-duplicate**: same record with one field slightly changed (e.g. `"John"` vs `"john"`)
- **same PK, different data**: same `id`, different `value` (PK collision)

```python
def inject_duplicates(records, exact_rate=0.15, near_rate=0.10):
    result = list(records)
    n = len(records)
    
    # Exact duplicates
    for _ in range(int(n * exact_rate)):
        orig = random.choice(records).copy()
        orig["_fault_type"] = "duplicate_exact"
        result.append(orig)
    
    # Near duplicates (one field mutated)
    for _ in range(int(n * near_rate)):
        orig = random.choice(records).copy()
        field = random.choice([k for k in orig if not k.startswith("_") and isinstance(orig[k], str)])
        orig[field] = orig[field].lower()  # simple near-dup: case change
        orig["_fault_type"] = "duplicate_near"
        result.append(orig)
    
    random.shuffle(result)
    return result
```

---

## Fault Taxonomy

| Code | Category | Description |
|------|----------|-------------|
| `null_field` | Completeness | Required field is None/null/NaN |
| `type_mismatch` | Type | Field has wrong Python/SQL type |
| `out_of_range` | Range | Value outside declared min/max |
| `boundary_min` | Boundary | Value is exactly at minimum (valid edge) |
| `boundary_max` | Boundary | Value is exactly at maximum (valid edge) |
| `boundary_zero` | Boundary | Value is exactly zero |
| `boundary_empty` | Boundary | Empty string / zero-length collection |
| `duplicate_exact` | Uniqueness | Identical copy of an earlier row |
| `duplicate_near` | Uniqueness | Near-copy with minor mutation |
| `pk_collision` | Uniqueness | Same primary key, different data |
| `schema_extra` | Schema | Unexpected field present |
| `schema_missing` | Schema | Expected field absent |
| `schema_rename` | Schema | Field renamed (producer/consumer mismatch) |
| `type_chaos_int` | Type | Int field contains non-int |
| `type_chaos_str` | Type | String field contains non-string |
| `adversarial` | Security | Injection or overflow string |
| `temporal_ooo` | Temporal | Out-of-order timestamp |
| `temporal_future` | Temporal | Timestamp in the future |
| `temporal_epoch` | Temporal | Unix epoch timestamp |
| `temporal_dst` | Temporal | DST-ambiguous timestamp |
| `null_storm` | Completeness | Multiple fields null in one record |
| `encoding_utf8` | Encoding | Non-ASCII / UTF-8 special characters |
| `encoding_bom` | Encoding | BOM character at start of string |

---

## Adversarial String Library

Use these in any string field when `profile = adversarial`. Assign via `random.choice()`.

```python
ADVERSARIAL_STRINGS = [
    # SQL injection
    "'; DROP TABLE users; --",
    "1 OR 1=1",
    "' UNION SELECT null, null, null --",
    
    # XSS / HTML injection
    "<script>alert('xss')</script>",
    '"><img src=x onerror=alert(1)>',
    "javascript:alert(document.cookie)",
    
    # Path traversal
    "../../etc/passwd",
    "../../../windows/system32/config/sam",
    
    # Format string
    "%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s",
    "%d %d %d %d %d",
    
    # Null byte / control chars
    "normal\x00hidden",
    "line1\rline2",          # carriage return overwrite
    "field1\tfield2",        # tab (breaks TSV)
    
    # Unicode edge cases
    "\u202e" + "reverse",    # right-to-left override
    "café",                  # NFC vs NFD normalization
    "\ufeff" + "bom_text",   # BOM
    "𝕳𝖊𝖑𝖑𝖔",              # Mathematical bold fraktur (confusables)
    "ñoño",                  # Spanish special chars
    "日本語",                 # CJK
    "مرحبا",                 # Arabic (RTL)
    "🚀💥🔥",               # Emoji (4-byte UTF-8)
    
    # Very long string (DoS / buffer overflow simulation)
    "A" * 10_000,
    "B" * 65_535,            # 16-bit max
    
    # Numeric strings (ambiguous coercion)
    "1e308",                  # float overflow
    "999999999999999999999",  # big int
    "0x1A",                   # hex
    "3.14.15",                # invalid float
    "1,234,567",              # comma-formatted number
    
    # JSON / YAML break attempts
    '{"key": "value"}',       # nested JSON in a string field
    "null",
    "true",
    "[]",
    "---\nkey: value",        # YAML break in string
    
    # Date-like but invalid
    "2024-02-30",             # Feb 30 doesn't exist
    "2024-13-01",             # Month 13
    "99:99:99",               # Invalid time
    
    # Whitespace variations
    "",                       # empty
    " ",                      # single space
    "   ",                    # spaces only
    "\t",                     # tab only
    "\n",                     # newline only
    "\r\n",                   # CRLF
]
```

---

## Distribution Reference

When generating numeric fields, pick the appropriate distribution:

| Distribution | Use case | Python |
|-------------|----------|--------|
| Uniform | Random values across a range | `random.uniform(lo, hi)` |
| Normal/Gaussian | Age, height, test scores | `random.gauss(mu, sigma)` |
| Log-normal | Income, price, duration | `numpy.random.lognormal(mean, sigma)` |
| Exponential | Time between events | `numpy.random.exponential(scale)` |
| Zipf/Power-law | Word frequencies, city sizes | `numpy.random.zipf(a)` |
| Bernoulli | Binary outcomes (0/1) | `random.random() < p` |
| Binomial | Count of successes in N trials | `numpy.random.binomial(n, p)` |
| Poisson | Event counts per interval | `numpy.random.poisson(lam)` |
| Beta | Proportions (0–1 bounded) | `numpy.random.beta(a, b)` |
| Constant | All same value (degenerate) | `[value] * n` |
| Step | Block of A, then block of B | `[A]*n//2 + [B]*n//2` |
| Increasing | Monotonically growing | `range(n)` or `cumsum` |

For categorical fields:
- **Balanced**: `random.choices(categories, k=n)` — equal probability
- **Skewed**: `random.choices(categories, weights=[0.7, 0.2, 0.1], k=n)` — long tail
- **Single dominant**: 90% one value, rest distributed across others
