 #  IMDB Data Cleaning Project

A complete data cleaning pipeline for a messy IMDB movies dataset using Python and Pandas. This project demonstrates real-world data quality issues and how to fix them systematically — a strong portfolio piece for any data analyst or data engineer role.

---

##  Project Structure

```
imdb-data-cleaning/
│
├── messy_IMDB_dataset.csv     # Raw input dataset (101 rows, 12 columns)
├── clean_imdb_dataset.py      # Main cleaning script
├── imdb_cleaned.csv           # Output — cleaned dataset
└── README.md                  # This file
```

---

## Dataset Overview
| Property | Raw | Cleaned |
|---|---|---|
| Rows | 101 | 100 |
| Columns | 12 | 11 |
| Missing values | 35+ | 0 |
| Usable numeric columns | 0 | 4 |

---

##  Issues Found & Fixed

### 1. Encoding & Column Names
- File used `latin1` encoding — caused `UnicodeDecodeError` with default UTF-8
- Two column names were garbled due to encoding: `Original titlÊ`, `Genrë¨`
- One column had leading/trailing whitespace: `" Votes "`
- **Fix:** Load with `encoding='latin1'`, strip and rename all columns to clean snake_case

### 2. Ghost Column
- `Unnamed: 8` was 100% null — a leftover from an extra delimiter
- **Fix:** Dropped immediately after loading

### 3. Inconsistent Date Formats (`release_date`)
Multiple formats found in the same column:
```
1995-02-10        ← ISO format
09 21 1972        ← US format, space-separated
23rd December of 1966  ← natural language
18/11/1976        ← European format
10-29-99          ← 2-digit year
```
- **Fix:** Regex to extract 4-digit year as fast path; `dateutil.parser` with `fuzzy=True` as fallback for natural language dates

### 4. Corrupted Score Values
```
"8,9f"   "8..8"   "8:8"   "++8.7"   "8.7."   "9,."   "08.9"
```
- **Fix:** Strip all non-numeric characters, collapse duplicate dots, remove trailing dots, validate range 1.0–10.0

### 5. Income Noise
```
"$ 4o8,035,783"   ← letter 'o' instead of digit '0' (OCR error)
"$ 576"           ← suspiciously low value
```
- **Fix:** Remove `$` and spaces, replace OCR `o→0`, strip commas, parse to float

### 6. Duration Sentinel Strings
```
"Inf"   "Nan"   "-"   " "   "Not Applicable"   "178c"
```
- **Fix:** Explicit blocklist for sentinel strings → `NaN`; strip trailing non-digit characters; validate range 60–300 mins

### 7. Country Inconsistencies
```
"USA"  "US"  "US."  "New Zesland"  "New Zeland"  "Italy1"  "West Germany"
```
- **Fix:** Lookup map to standardise to consistent country names

### 8. European Votes Format
```
"2.278.845"  →  2278845
"1.572.674"  →  1572674
```
- **Fix:** Remove dots used as thousands separators before parsing to integer

### 9. Missing Values
| Column | Missing | Strategy |
|---|---|---|
| `content_rating` | 24 | Fill with `"Unknown"` |
| `duration_min` | 4 | Fill with median |
| `score` | 1 | Fill with median |
| `income_usd` | 1 | Fill with median |
| `votes` | 1 | Fill with median |
| `release_year` | 1 | Fill with median year |

---

##  Key Bug Fixed — fillna Before astype

A critical ordering issue was resolved during development. Casting a column to `Int64` *before* filling NaN values raises:

```
TypeError: Invalid value '129.5' for dtype 'Int64'
```

The correct order is always:

```python
# WRONG
df[col] = df[col].astype("Int64")
df[col] = df[col].fillna(median_val)   # ← crashes

# CORRECT
df[col] = df[col].fillna(median_val)   # 1. fill while float64
df[col] = df[col].round(0).astype("Int64")  # 2. then cast
```

---

##  Tech Stack

- **Python 3.10+**
- **Pandas** — data loading, transformation, type casting
- **NumPy** — NaN handling
- **re** — regex-based string cleaning
- **python-dateutil** — fuzzy date parsing

Install dependencies:

```bash
pip install pandas numpy python-dateutil
```

---

##  How to Run

1. Clone the repo and place `messy_IMDB_dataset.csv` in the project folder
2. Run the cleaning script:

```bash
python clean_imdb_dataset.py
```

3. Output is saved as `imdb_cleaned.csv` in the same directory

---

##  Final Schema

| Column | Type | Description |
|---|---|---|
| `title_id` | string | IMDB title ID (e.g. tt0111161) |
| `title` | string | Movie title |
| `release_year` | Int64 | Year of release |
| `genre` | string | Genre(s), comma-separated |
| `duration_min` | Int64 | Runtime in minutes |
| `country` | string | Country of origin |
| `content_rating` | string | Rating (R, PG-13, etc.) |
| `director` | string | Director name |
| `income_usd` | Int64 | Box office income in USD |
| `votes` | Int64 | Number of IMDB votes |
| `score` | float64 | IMDB score (1.0 – 10.0) |

---

##  Skills Demonstrated

- Handling real-world encoding issues (`latin1` vs UTF-8)
- Multi-format date parsing with regex + `dateutil`
- String cleaning with regex (`re.sub`)
- OCR error correction
- Pandas dtype management (`float64` → `Int64` pipeline)
- Systematic null value strategy (drop vs impute)
- Data validation with range checks

---

##  Sample — Before vs After

**Score column:**
```
Before: "8,9f"  "8..8"  "8:8"  "++8.7"  "9,."
After:   8.9     8.8     8.8     8.7      9.0
```

**Country column:**
```
Before: "US"  "US."  "New Zesland"  "Italy1"
After:  "USA" "USA"  "New Zealand"  "Italy"
```

**Votes column:**
```
Before: "2.278.845"  "1.572.674"
After:   2278845      1572674
```
