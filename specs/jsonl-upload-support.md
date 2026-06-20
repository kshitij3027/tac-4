# Feature: JSONL Upload Support

## Feature Description
Add support for uploading `.jsonl` (JSON Lines) files alongside the existing `.csv` and `.json` formats. Each JSONL file produces one new SQLite table. Because JSONL records are independent JSON objects (one per line), different rows may carry different keys — the implementation performs a full two-pass scan of the file to collect every possible field name before building the table. Nested objects and nested lists are flattened into column names using a configurable delimiter (`__`) stored in a shared constants file, with list items indexed positionally (e.g., `tags__0`, `tags__1`).

## User Story
As a data analyst  
I want to upload `.jsonl` files and query them with natural language  
So that I can explore log data, ML datasets, and other line-delimited JSON formats without converting them first

## Problem Statement
The application only accepts `.csv` and `.json` (array of objects) uploads. JSONL is a widely used format for logs, event streams, and ML training data. Users who have JSONL files must manually convert them before uploading, adding friction. Additionally, JSONL files often contain deeply nested structures that need to be flattened into a queryable relational schema.

## Solution Statement
Extend the file processor with a `convert_jsonl_to_sqlite` function that:
1. Reads every line of the file to discover all possible keys (two-pass: collect union of all keys, then build rows).
2. Flattens nested dicts and lists recursively using a `__` delimiter (stored in `core/constants.py`).
3. Writes the resulting flat records to SQLite via `pandas.DataFrame`, matching the existing CSV/JSON flow.
4. Registers `.jsonl` in the server upload endpoint routing.
5. Updates the HTML drop-zone and file input to accept `.jsonl`.
6. Provides test JSONL files and unit tests following the existing `test_file_processor.py` pattern.

## Relevant Files

- **`app/server/core/file_processor.py`** — add `convert_jsonl_to_sqlite` and the shared `flatten_record` helper; this is the primary implementation site.
- **`app/server/server.py`** — extend the `/api/upload` endpoint to accept `.jsonl` and route to the new converter.
- **`app/client/index.html`** — update drop-zone copy and `<input accept>` to include `.jsonl`.
- **`app/server/tests/core/test_file_processor.py`** — add JSONL test cases following existing patterns.
- **`app/server/tests/assets/`** — location for new JSONL fixture files.

### New Files
- **`app/server/core/constants.py`** — defines `NESTED_FIELD_DELIMITER = "__"` so every part of the codebase that references the separator imports from a single source.
- **`app/server/tests/assets/test_events.jsonl`** — flat JSONL fixture (simple key/value lines, no nesting) for basic upload tests.
- **`app/server/tests/assets/test_nested.jsonl`** — JSONL fixture with nested objects, nested lists, and sparse keys (some lines missing fields) to exercise the two-pass scan and flattener.

## Implementation Plan

### Phase 1: Foundation
Create `core/constants.py` with the delimiter constant. Implement the recursive `flatten_record` helper inside `file_processor.py` that handles:
- Nested dicts → `parent__child` column names
- Nested lists → `parent__0`, `parent__1` column names
- Mixed nesting → composed paths

Write the two-pass JSONL scanner (`convert_jsonl_to_sqlite`) that:
1. First pass: decode every line with `json.loads`, call `flatten_record`, collect the union of all resulting keys.
2. Second pass: build a list of dicts keyed by that full union (missing fields → `None`).
3. Hand the list to `pandas.DataFrame` and use the existing `to_sql` + schema-fetch pattern.

### Phase 2: Core Implementation
Wire `convert_jsonl_to_sqlite` into the server's `/api/upload` handler, alongside the existing CSV and JSON branches. Update the file-type validation to allow `.jsonl`.

### Phase 3: Integration
Update the client UI (`index.html`) drop-zone paragraph text and the hidden `<input type="file" accept>` attribute to include `.jsonl`. Create test JSONL fixture files. Write unit tests covering success, nested fields, sparse keys, invalid lines, and empty files.

## Step by Step Tasks

### Step 1: Create `core/constants.py`
- Create `app/server/core/constants.py`
- Define `NESTED_FIELD_DELIMITER = "__"` — the separator used when flattening nested JSON keys into column names

### Step 2: Implement `flatten_record` in `file_processor.py`
- Import `NESTED_FIELD_DELIMITER` from `core.constants` at the top of `file_processor.py`
- Add `flatten_record(record: dict, prefix: str = "", delimiter: str = NESTED_FIELD_DELIMITER) -> dict` function:
  - Iterate over `record.items()`
  - If value is a `dict`, recurse with `prefix + key + delimiter`
  - If value is a `list`, iterate with index: recurse or store `prefix + key + delimiter + str(i)` for each element
  - For list elements that are dicts, recurse with `prefix + key + delimiter + str(i) + delimiter`
  - Otherwise store flat value at `prefix + key`
- Keep function pure — no side effects, no I/O

### Step 3: Implement `convert_jsonl_to_sqlite`
- Add `convert_jsonl_to_sqlite(jsonl_content: bytes, table_name: str) -> Dict[str, Any]` to `file_processor.py`
- Sanitize `table_name` via existing `sanitize_table_name`
- Decode bytes to UTF-8 string, split on newlines, filter blank lines
- First pass: for each line `json.loads(line)` → `flatten_record(row)`, add all keys to a `set`
- If no valid lines found raise `ValueError("JSONL file is empty or contains no valid records")`
- Second pass: rebuild each flattened row as a full dict with all collected keys (missing → `None`)
- Clean column names: `.lower().replace(' ', '_').replace('-', '_')` (matching CSV/JSON convention)
- Build `pd.DataFrame(rows)` and write via `df.to_sql(table_name, conn, if_exists='replace', index=False)`
- Fetch schema, sample data, row count — return `{'table_name', 'schema', 'row_count', 'sample_data'}` matching existing shape
- Wrap in `try/except`, raise `Exception(f"Error converting JSONL to SQLite: {str(e)}")`

### Step 4: Update `server.py` upload endpoint
- In `server.py`, add `convert_jsonl_to_sqlite` to the import from `core.file_processor`
- Change the file-type validation line from:
  ```python
  if not file.filename.endswith(('.csv', '.json')):
      raise HTTPException(400, "Only .csv and .json files are supported")
  ```
  to:
  ```python
  if not file.filename.endswith(('.csv', '.json', '.jsonl')):
      raise HTTPException(400, "Only .csv, .json, and .jsonl files are supported")
  ```
- Add a new `elif` branch in the routing block:
  ```python
  elif file.filename.endswith('.jsonl'):
      result = convert_jsonl_to_sqlite(content, table_name)
  ```

### Step 5: Update client HTML
- In `app/client/index.html`, update the drop-zone paragraph:
  - Change `<p>Drag and drop .csv or .json files here</p>` to `<p>Drag and drop .csv, .json, or .jsonl files here</p>`
- Update the file input accept attribute:
  - Change `accept=".csv,.json"` to `accept=".csv,.json,.jsonl"`

### Step 6: Create test JSONL fixture files
- Create `app/server/tests/assets/test_events.jsonl`:
  - 4 lines, each a flat JSON object with fields: `id`, `event`, `user_id`, `timestamp`
  - Example:
    ```
    {"id": 1, "event": "login", "user_id": 101, "timestamp": "2024-01-01T10:00:00"}
    {"id": 2, "event": "purchase", "user_id": 102, "timestamp": "2024-01-01T11:00:00"}
    {"id": 3, "event": "logout", "user_id": 101, "timestamp": "2024-01-01T12:00:00"}
    {"id": 4, "event": "signup", "user_id": 103, "timestamp": "2024-01-02T09:00:00"}
    ```
- Create `app/server/tests/assets/test_nested.jsonl`:
  - 3 lines with nested objects, nested lists, and one line missing an optional field
  - Example:
    ```
    {"id": 1, "name": "Alice", "address": {"city": "New York", "zip": "10001"}, "tags": ["python", "ml"], "score": 95.5}
    {"id": 2, "name": "Bob", "address": {"city": "Boston", "zip": "02101"}, "tags": ["java", "backend", "devops"], "score": 88.0}
    {"id": 3, "name": "Carol", "address": {"city": "Chicago", "zip": "60601"}, "tags": ["design"], "score": null}
    ```

### Step 7: Add JSONL tests to `test_file_processor.py`
- Add these test methods to the existing `TestFileProcessor` class:
  - `test_convert_jsonl_to_sqlite_flat_success(test_db, test_assets_dir)`:
    - Load `test_events.jsonl`; call `convert_jsonl_to_sqlite`
    - Assert `row_count == 4`, schema contains `id`, `event`, `user_id`, `timestamp`
    - Assert a specific row value (e.g., event `"login"` has `user_id == 101`)
  - `test_convert_jsonl_to_sqlite_nested_fields(test_db, test_assets_dir)`:
    - Load `test_nested.jsonl`; call `convert_jsonl_to_sqlite`
    - Assert flattened columns exist: `address__city`, `address__zip`, `tags__0`, `tags__1`, `tags__2`
    - Assert row 1 has `address__city == "New York"`, `tags__0 == "python"`
    - Assert row 3 has `tags__2 == "devops"` (Bob's third tag)
    - Assert `tags__2` is None for Carol (only 1 tag)
  - `test_convert_jsonl_to_sqlite_sparse_keys(test_db)`:
    - Use inline bytes with two records where one is missing a key
    - Assert both rows appear and the missing key is `None` in the sparse row
  - `test_convert_jsonl_to_sqlite_empty(test_db)`:
    - Pass `b""` or a bytes with only blank lines
    - Assert raises `Exception` with `"Error converting JSONL to SQLite"`
  - `test_convert_jsonl_to_sqlite_invalid_line(test_db)`:
    - Pass bytes with one valid line and one invalid JSON line
    - Assert raises `Exception` with `"Error converting JSONL to SQLite"`

### Step 8: Run validation commands
- Run `cd app/server && uv run pytest` — all tests must pass
- Manually verify `.jsonl` appears in the file input dialog (accept attribute)
- Manually verify the drop-zone text includes `.jsonl`

## Testing Strategy

### Unit Tests
- `test_convert_jsonl_to_sqlite_flat_success` — happy-path flat JSONL
- `test_convert_jsonl_to_sqlite_nested_fields` — nested objects and lists flatten correctly with `__` delimiter
- `test_convert_jsonl_to_sqlite_sparse_keys` — union of keys; missing fields become `None`
- `test_convert_jsonl_to_sqlite_empty` — empty file raises descriptive exception
- `test_convert_jsonl_to_sqlite_invalid_line` — malformed JSON line raises exception

### Integration Tests
- Existing CSV and JSON tests must continue to pass (no regressions)
- The server validation change allows `.jsonl` and rejects unknown extensions with an updated message

### Edge Cases
- JSONL with only blank lines → empty error
- Single-record JSONL → works (row_count == 1)
- Deeply nested dicts (3+ levels) → flattened correctly
- Lists containing dicts (e.g., `"items": [{"name": "a"}, {"name": "b"}]`) → `items__0__name`, `items__1__name`
- Mixed flat and deeply nested rows in the same file → two-pass union correctly captures all keys
- JSONL with numeric or boolean leaf values → stored correctly in SQLite column

## Acceptance Criteria
- Uploading a `.jsonl` file via the UI or API creates a new SQLite table, identical in behavior to uploading `.csv` or `.json`
- Nested object fields are stored as columns named `parent__child` (using `__` from `constants.py`)
- Nested list items are stored as columns named `parent__0`, `parent__1`, etc.
- Rows missing a key present in other rows have `None` (NULL) for that column
- The drop-zone text and file input accept attribute include `.jsonl`
- All existing tests pass (zero regressions)
- The five new JSONL unit tests all pass
- `NESTED_FIELD_DELIMITER` is defined in `core/constants.py` and imported wherever used

## Validation Commands
Execute every command to validate the feature works correctly with zero regressions.

- `cd app/server && uv run pytest` - Run all server tests (existing + new) with zero failures
- `cd app/server && uv run pytest tests/core/test_file_processor.py -v` - Run file processor tests verbosely to confirm all JSONL cases pass
- `grep -n "jsonl" app/server/server.py` - Confirm `.jsonl` is in the upload validation and routing
- `grep -n "jsonl" app/client/index.html` - Confirm `.jsonl` appears in drop-zone text and file input accept
- `grep -n "NESTED_FIELD_DELIMITER" app/server/core/constants.py app/server/core/file_processor.py` - Confirm constant is defined and used

## Notes
- No new dependencies required — `json` (stdlib), `pandas`, and `sqlite3` already handle everything.
- `flatten_record` should be importable from `file_processor` in case future formats (e.g., nested CSV) need it.
- The two-pass approach is intentional: a single pass would miss keys that appear only in later rows, producing silently incomplete schemas. For large files this doubles I/O but correctness takes priority; future optimization could use a streaming union.
- List index notation (`_0`, `_1`) was chosen to mirror the requirement; the delimiter before the index (`__0`) uses the same `NESTED_FIELD_DELIMITER` so the separator is truly single-source.
- If a list element is itself a dict, the columns become `list_field__0__nested_key`, consistent with the flat separator rule.
- The `sanitize_table_name` function already strips the extension, so `upload.jsonl` → table `upload` without extra handling.
