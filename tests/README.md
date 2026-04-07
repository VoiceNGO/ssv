# SSV Test Fixtures

Shared test fixtures for all SSV parser implementations.

## File conventions

Each test is a named pair of files in a category subdirectory:

| File          | Meaning                        |
| ------------- | ------------------------------ |
| `{name}.ssv`  | Input to parse                 |
| `{name}.json` | Expected output (valid parse)  |
| `{name}.err`  | Expected error (invalid input) |

A test has either a `.json` or an `.err` companion — never both.

## JSON output format

All `.json` files contain a top-level array of tables, even when only one table is present:

```json
[[row, row, ...], [row, row, ...]]
```

Each row is a JSON object keyed by field name. Values map as follows:

| SSV type                                            | JSON type |
| --------------------------------------------------- | --------- |
| `string`, `string(N)`, `string(..N)`, `string[...]` | string    |
| `bool`                                              | boolean   |
| all number formats                                  | number    |
| `T[]`                                               | array     |
| `[T1, T2]` (unnamed tuple)                          | array     |
| `[name1:T1, name2:T2]` (named tuple)                | object    |
| null (via `#! NULL`)                                | `null`    |

## Error format

`.err` files contain a short human-readable description of the expected error. Implementations are not required to match the exact text; the description exists to document intent. Match on error class (type error, range error, parse error) if your test runner categorizes errors.

## Running the fixtures

Typical test loop (pseudocode):

```
for each {name}.ssv in tests/**/:
    result = parse(read("{name}.ssv"))

    if {name}.json exists in same dir:
        expect result to equal parse_json(read("{name}.json"))

    if {name}.err exists in same dir:
        expect result to be an error
```

Implementations may choose to verify the error message or only assert that parsing failed.
