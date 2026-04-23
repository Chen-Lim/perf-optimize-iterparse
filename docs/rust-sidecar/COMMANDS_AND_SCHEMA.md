# Commands and Schema

## Binary

```bash
healthpro-engine <COMMAND> [OPTIONS]
```

## Command: scan-sources

Scans an Apple Health export and returns source names plus summary counts.

```bash
healthpro-engine scan-sources /path/export.zip
```

Options:

```text
--progress-every <N>      Emit progress every N parsed entities. Default: 500000
--json                    Emit only JSONL. Default: true for UI usage
--error-log <PATH>        Optional JSONL error log path
```

### scan-sources stdout JSONL

Started:

```json
{
  "event": "started",
  "command": "scan-sources",
  "input": "/path/export.zip"
}
```

Progress:

```json
{
  "event": "progress",
  "processed": 500000,
  "records": 498100,
  "workouts": 1900,
  "unique_sources": 4
}
```

Done:

```json
{
  "event": "done",
  "command": "scan-sources",
  "records": 12000000,
  "workouts": 3200,
  "sources": [
    {
      "name": "Apple Watch",
      "records": 8000000,
      "workouts": 3000,
      "total": 8003000
    },
    {
      "name": "iPhone",
      "records": 4000000,
      "workouts": 200,
      "total": 4000200
    }
  ],
  "elapsed_ms": 65000
}
```

Error:

```json
{
  "event": "error",
  "message": "zip archive does not contain a supported Apple Health export xml file",
  "code": "input_not_found"
}
```

## Command: export

Exports selected sources into category CSV files.

```bash
healthpro-engine export /path/export.zip \
  --sources /tmp/selected_sources.json \
  --categories categories/health_categories.json \
  --out /path/output
```

Options:

```text
--sources <PATH>          Required JSON file containing selected source names
--categories <PATH>       Category rule JSON. If omitted, use embedded defaults
--out <DIR>               Output directory. Default: input file directory
--format <csv|parquet>    Output format. MVP: csv only
--chunk-size <N>          Max CSV rows per part. Default: 880000
--bom / --no-bom          Add UTF-8 BOM for Excel. Default: --bom
--progress-every <N>      Emit progress every N parsed entities. Default: 500000
--error-log <PATH>        Optional JSONL error log path
```

### export stdout JSONL

Started:

```json
{
  "event": "started",
  "command": "export",
  "input": "/path/export.zip",
  "output_dir": "/path/output",
  "selected_sources": 2
}
```

Progress:

```json
{
  "event": "progress",
  "processed": 1000000,
  "matched_sources": 820000,
  "written": 810000,
  "skipped": 10000,
  "errors": 0
}
```

File saved:

```json
{
  "event": "file_saved",
  "category": "1_Heart_Cardio",
  "path": "/path/output/1_Heart_Cardio.csv",
  "rows": 250000
}
```

Split notice:

```json
{
  "event": "split",
  "category": "1_Heart_Cardio",
  "part": 2,
  "path": "/path/output/1_Heart_Cardio_Part2.csv"
}
```

Skipped category:

```json
{
  "event": "category_skipped",
  "category": "5_Mobility_Gait",
  "reason": "no_data"
}
```

Done:

```json
{
  "event": "done",
  "command": "export",
  "processed": 12003200,
  "matched_sources": 8003000,
  "written": 7900000,
  "errors": 0,
  "files": [
    {
      "category": "1_Heart_Cardio",
      "path": "/path/output/1_Heart_Cardio.csv",
      "rows": 250000
    }
  ],
  "elapsed_ms": 90000
}
```

## Selected Sources Schema

File written by Python before calling `export`.

```json
{
  "sources": [
    "Apple Watch",
    "iPhone"
  ]
}
```

Rust validation:

- `sources` must exist.
- `sources` must be a non-empty array.
- Every source must be a non-empty string.
- Matching must be exact after XML decoding.

## Category Rule Schema

MVP file:

```json
{
  "version": 1,
  "categories": [
    {
      "id": "1_Heart_Cardio",
      "label": "Heart & Cardio",
      "keywords": [
        "heartrate",
        "restingheartrate",
        "heartratevariability",
        "walkingheartrateaverage"
      ]
    }
  ]
}
```

Fields:

```text
version       Integer schema version.
categories    Ordered array. Order controls export/log order.
id            File base name. Must be filesystem-safe.
label         Human-readable label.
keywords      Case-insensitive substring keywords matched against normalized type.
```

Validation:

- Category IDs must be unique.
- Keywords must be lowercase ASCII in the committed config.
- Empty keyword lists are invalid.
- Unknown fields should be ignored for forward compatibility.

## Default Category Rules

Initial Rust config must preserve current Python grouping:

```json
{
  "version": 1,
  "categories": [
    {
      "id": "1_Heart_Cardio",
      "label": "Heart & Cardio",
      "keywords": ["heartrate", "restingheartrate", "heartratevariability", "walkingheartrateaverage"]
    },
    {
      "id": "2_Body_Metrics",
      "label": "Body Metrics",
      "keywords": ["bodymass", "bmi", "bodyfat", "leanbodymass", "bodywatermass"]
    },
    {
      "id": "3_Daily_Activity",
      "label": "Daily Activity",
      "keywords": ["stepcount", "activeenergy", "basalenergy", "distance", "flights"]
    },
    {
      "id": "4_Sleep_Recovery",
      "label": "Sleep Recovery",
      "keywords": ["sleepanalysis"]
    },
    {
      "id": "5_Mobility_Gait",
      "label": "Mobility & Gait",
      "keywords": ["walkingspeed", "steplength", "asymmetry", "support", "steadiness"]
    },
    {
      "id": "6_Reproductive",
      "label": "Reproductive",
      "keywords": ["menstrual", "ovulation", "cervical"]
    },
    {
      "id": "7_Vitals_Respiratory",
      "label": "Vitals & Respiratory",
      "keywords": ["oxygensaturation", "respiratoryrate", "bodytemperature", "bloodpressure"]
    },
    {
      "id": "8_Running_Dynamics",
      "label": "Running Dynamics",
      "keywords": ["runningpower", "verticaloscillation", "groundcontact", "runningstridelength", "runningspeed"]
    },
    {
      "id": "9_Cycling_Stats",
      "label": "Cycling Stats",
      "keywords": ["cyclingpower", "cadence", "cyclingspeed", "functionalthreshold"]
    },
    {
      "id": "10_Swimming_Water",
      "label": "Swimming & Water",
      "keywords": ["swimming", "strokecount", "underwater", "watertemperature"]
    },
    {
      "id": "11_Workouts_Training",
      "label": "Workouts & Training",
      "keywords": ["workout", "hkworkout", "running", "walking", "cycling", "strength"]
    },
    {
      "id": "12_Environment_Senses",
      "label": "Environment & Senses",
      "keywords": ["timeindaylight", "environmentalaudio", "headphoneaudio"]
    },
    {
      "id": "13_Nutrition_Hydration",
      "label": "Nutrition & Hydration",
      "keywords": ["dietary", "water", "caffeine"]
    },
    {
      "id": "14_Mindfulness_Mental",
      "label": "Mindfulness & Mental",
      "keywords": ["mindful", "stateofmind"]
    },
    {
      "id": "15_Symptoms_Illness",
      "label": "Symptoms & Illness",
      "keywords": ["symptom"]
    }
  ]
}
```

## CSV Row Schema

MVP output columns match current Python output:

```csv
type,value,unit,startdate,sourcename
```

`Record` mapping:

```text
type        <- Record@type
value       <- Record@value
unit        <- Record@unit
startdate   <- Record@startDate
sourcename  <- Record@sourceName
```

`Workout` mapping:

```text
type        <- Workout@workoutActivityType or "Workout"
value       <- Workout@duration
unit        <- Workout@durationUnit or "min"
startdate   <- Workout@startDate
sourcename  <- Workout@sourceName
```

## Future SQLite Schema

The sidecar can later support `--format sqlite` or a separate `ingest-db`
command based on the Apple-Health-Resonator-CLI schema.

Minimum future tables:

```sql
CREATE TABLE records (
  id INTEGER PRIMARY KEY,
  record_type TEXT NOT NULL,
  value_text TEXT,
  value_num REAL,
  unit TEXT,
  source_name TEXT,
  source_version TEXT,
  device TEXT,
  creation_date TEXT,
  start_date TEXT NOT NULL,
  end_date TEXT,
  dedupe_key TEXT UNIQUE
);

CREATE TABLE workouts (
  id INTEGER PRIMARY KEY,
  workout_type TEXT NOT NULL,
  duration REAL,
  duration_unit TEXT,
  total_distance REAL,
  total_energy_burned REAL,
  source_name TEXT,
  creation_date TEXT,
  start_date TEXT NOT NULL,
  end_date TEXT,
  dedupe_key TEXT UNIQUE
);

CREATE TABLE ingest_runs (
  id INTEGER PRIMARY KEY,
  started_at TEXT NOT NULL,
  finished_at TEXT,
  input_path TEXT NOT NULL,
  records_inserted INTEGER,
  workouts_inserted INTEGER,
  records_skipped INTEGER,
  errors_count INTEGER,
  schema_version TEXT NOT NULL
);
```
