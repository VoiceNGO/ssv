# SSV — Super Separated Values... or Spaghetti Syntax Voodoo

A sane replacement for CSV files. Human readable, machine parsable, built-in schema and validation, fully customizable

While its intent is a more powerful CSV replacement, you _could_ use it for just about anything. It would just get really, really busy.

## Language Specific Implementations

This repository only outlines the SSV spec. Implementations are currently available for [Rust](https://github.com/VoiceNGO/ssv-rust), [TypeScript](https://github.com/VoiceNGO/ssv-typescript), and [Python](https://github.com/VoiceNGO/ssv-python)

## Format overview

A basic SSV document looks like this:

```
# This is a comment

name:string | age:int | score:float | tags:string[]
Alice       | 30      | 9.5         | rust;pl;systems
Bob         | 25      | 7.0         | java
```

- **First non-comment line** is the header: `fieldname:type` columns separated by the primary delimiter.
- **Subsequent lines** are data rows, one row per line.
- **`#` lines** are comments and are ignored anywhere in the document.

Or, if you prefer a more CSV-style look:

```
#! DELIMITERS , ;

name,age:int,score:float,tags
Alice,30,9.5,rust;pl;systems
Bob,25,7.0,java
```

## Strict Parsers

The parsers are designed to be strict. If the file parses, the data is valid. Any error will result in a failure to parse the entire file and as such validating SSV files should be part of your workflow, preferably an automated one, e.g. in a git pre-commit hook

## Basic Types

| SSV type          | Notes                          | Example value                 |
| ----------------- | ------------------------------ | ----------------------------- |
| `string`          | Default type if none specified | `hello world`                 |
| `string(N)`       | Exactly N characters           | `EUR` for `string(3)`         |
| `string(..N)`     | Up to N characters             | `Dinosaur` for `string(..10)` |
| `string[A, B, C]` | An enum                        | `string[Red, Green, Blue]`    |
| `bool`            |                                | `true` / `false` / `0` / `1`  |
| `float`           | 32 bit                         | `3.14`                        |
| `int`             | 32 bit                         | `42`                          |
| `T[]`             | list of type T                 | `1;2;3`                       |
| `[T1, T2]`        | tuple                          | `10;hello`                    |

## More Numeric Types

The basic numeric types are all 32 bit and signed. If you don't know what that means, it means a valid int value is roughly ± 2 billion. If you want other types, they exist:

| SSV type  | Min             | Max            |
| --------- | --------------- | -------------- |
| `float`   | ≈1.18 x 10^-38  | ≈3.40 x 10^38  |
| `float64` | ≈2.23 x 10^-308 | ≈1.79 x 10^308 |
| `int`     | -2,147,483,648  | 2,147,483,647  |
| `int8`    | -128            | 127            |
| `int16`   | -32,768         | 32,767         |
| `int64`   | -9.22 x 10^18   | 9.22 x 10^18   |
| `int128`  | -1.70 x 10^38   | 1.70 x 10^38   |
| `uint`    | 0               | 4,294,967,295  |
| `uint8`   | 0               | 255            |
| `uint16`  | 0               | 65,535         |
| `uint64`  | 0               | 1.84 x 10^19   |
| `uint128` | 0               | 3.40 x 10^38   |

- Don't know which one to choose? Pick the one with the smallest range that you are 100% sure will fit all of your data. e.g. a `uint8`, which caps at 255, is a good choice for `age`, unless we're talking about trees.
- Don't care? just use `int` everywhere, which is fine for _most_ data. Large parsed SSV documents will use a bit more memory.
- Implementations of various int & float sizes is language dependant. Some languages may use strings for values over 32 or 64 bits. Floats are always 64 bits in JS, for instance, but the parser will do a range check to validate data

## Numeric Formats

Various numeric formats are parsed by default. Note that if you disable any of them, and a number is presented in that format, the parser will entirely fail

### Radix formats

Binary, octal, and hex formats are parsed by default, e.g. `0b101010101`, `0o123456`, or `0x1234abcd` (case insensitive)

You may disable these via these [parser comments](#parser-comments):

```
#! DISABLE_BINARY_NUMBERS
#! DISABLE_HEX_NUMBERS
#! DISABLE_OCTAL_NUMBERS
#! DISABLE_RADIX_NUMBERS
```

`DISABLE_RADIX_NUMBERS` disables all of these types

### Exponentials

Exponential numbers are also supported by default, e.g. `1e3`, which evaluates to `1000`

Disable this with `#! DISABLE_EXPONENTIAL_NUMBERS` if desired

### Numeric Symbols

- By default, `.` is the decimal separator for floats, e.g. `3.14`, but you can change this with `#! DECIMAL_SEPARATOR ,`, for instance
- A decimal separator used in an int field is invalid
- You may use `#! NUMERIC_SEPARATOR _` (with whatever single character you like) as a separator in large numbers, e.g. `1_000_000`
  - Must not conflict with other delimiters
  - Is completely ignored when parsing numeric fields
  - Feel free to use `,` or `.` if you aren't already using those tokens as delimiters
- Negative numbers begin with `-` by default, but you can use `#! PARENTHETICAL_NEGATIVES` to use parenthetical notation, e.g. `(500)`, instead

## Tuples and Lists

Tuples are lists with known lengths and represented as `[T1, T2, ...TN]`
Lists are of unknown length and represented as `T[]`

Tuples are limited to a maximum of **20 elements**. Parsers **will** reject tuples exceeding this limit.

## Whitespace and escaping

Structural whitespace (unescaped spaces and tabs) is trimmed from both ends of each field and element. The following escape sequences are recognised:

| Sequence | Meaning                                                                                     |
| -------- | ------------------------------------------------------------------------------------------- |
| `\\`     | Literal backslash                                                                           |
| `\n`     | Newline                                                                                     |
| `\ `     | Space (preserved at field boundaries)                                                       |
| `\t`     | Tab (preserved at field boundaries)                                                         |
| `\#`     | Literal `#` character                                                                       |
| `\X`     | Literal delimiter character `X`, e.g. `\|` or `\;`, depending on what delimiters are in use |

- Any other `\X` is an error.
- A primitive field that contains an **unescaped** delimiter character is an error; You _must_ escape it with `\`.
- Empty lines are always ignored
- Any line containing **only** the first delimiter, whitespace, and `-` characters is ignored by default. Why? See [Markdown Tables](#markdown-tables)
- If you have some reason to, you may change the escape character with `#! ESCAPE_CHARACTER ^` (using whatever character you prefer)

## Comments

Comments are entire lines starting with a `#` character. A `#` anywhere else in a line is treated as data and can only be parsed into a string field.

There is a possible edge case where a string that should start with `#` is treated as a comment. In this case, you can escape the `#` character or put an empty first column in your SSV:

```
| Comments     |
| # My Comment |
```

Note: If you want to format SSV config files using a Markdown parser, you may want to use `#!` for all comments as most Markdown parsers will add line breaks between what it sees as `# Header` lines. Just don't accidentally write `#! DELIMITERS` or similar!

An actual SSV formatter in VSCode is planned

## Parser Comments

Parser comments are instructions that change the parsing behavior and start with `#!`.

**IMPORTANT**: _Any_ parser comments found after a table header is parsed automatically create a _new_ table. See [Multiple Tables](#multiple-tables)

Note: For backwards compatibility, unknown parser instructions are silently ignored

## Delimiters

Delimiters are a ranked list of single characters. The first separates columns; The second separates elements within a list or tuple field, and so on.

The default delimiter set is `| ;`.

Declare custom delimiters with a parser comment:

```
#! DELIMITERS | ; : ,
```

### Delimiter Rules

- Must not be: alphanumeric, whitespace, the escape character, or `#`
- By default, must not be `.`, but you can change `NUMERIC_SEPARATOR` to allow this if desired
- Must not be `-`, unless `PARENTHETICAL_NEGATIVES` is set, then must not be `(` or `)`
- The first delimiter (the column separator) must not be `:`, `,`, `[`, or `]`
- No duplicates
- If there are multiple `#! DELIMITERS` comments, the last one _before each table_ is applied. This allows SSV files with different delimiters to be concatenated together and work.

### Delimiter Application

Each tuple or list at nesting depth _N_ splits on the Nth delimiter.

Whenever we have a list of tuples, e.g. `[string, string][]`, the list uses the first delimiter, and the tuple uses the next. Example:

```
#! DELIMITERS | ; :

friends:[string, string][]
Bob:Hope;Tom:Jones;Frank:Sinatra
```

If two tuples are at the same depth, they use the same delimiter:

```
#! DELIMITERS | ; :

parents:[[string, string], [string, string]]
Rob:Petrie ; Laura:Petrie
```

## Null / empty fields

SSV is null-free by default. An empty or missing field produces the **zero value** for its type by default:

| Type              | Zero value             |
| ----------------- | ---------------------- |
| string            | `""`                   |
| all numeric types | `0`                    |
| bool              | `false`                |
| lists             | `[]`                   |
| tuple             | zero value per element |

**Important Note**: A numeric type with a range condition, e.g. `age:uint8(18..)`, might not allow empty values at all and empty values can cause parsing failures.

Do you really want nulls in your data? I suggest adding a boolean "set" field instead.

### Do you really, _really_ want nulls in your data?

Hey, it's your data. Who am I to bring up the "billion dollar mistake" argument?

Add this as a parser comment:

```
#! NULL _
```

- The null character must follow the same rules as delimiter characters
- The null character must not be an existing delimiter
- Any field with that character and _only_ that character is considered null by the parser
- The character is still valid elsewhere, e.g. `red_apples` is still valid in this example.
- Only nullable fields can accept null data. Add a `?` after each field type, e.g. `name: string?`
- Nested fields can also be nullable: `name: [string?, string?]`

## Default values

SSV also allows default values in any field with the `=value` syntax:

```
player:string | wins:int=0 | losses:int=0
bob
alice
```

If a field is both nullable and has a default, the default will _not_ apply to fields with null set

**Note**: The original value is _lost_ in the parsing step by all parsers. So `serialize(deserialize(ssv))` will write out the default values to all previously empty fields. In practice however, it does not make sense to have both defaults and machine-generated SSV files.

## Numeric Ranges

All numeric types may be constrained via the syntax: `(min..max)`. `min` and `max` are both optional. e.g. to only allow numbers from 0-100, you may use `uint8(0..100)`, Ranges are inclusive, so the values `0` and `100` are both valid in this example.

Ranges come before defaults: `karma:int8(-100..100)=10`

## Field Count Mismatches, and Empty Fields

SSV only validates _data_, and ignores empty columns, either added or missing. So the following is valid:

```
| name: string |||
| bob |||||||||
```

As long as all the mis-matches contain no data, the parsers do not care what you do. This allows you to add new columns to an SSV file without needing to update every row:

```
name: string | age: uint8 | eye_color: string
bob
alice
```

As soon as we introduce a data mismatch between headers and data, however, the parser _will_ throw

**Not valid**:

```
   | name: string |
24 | bob          |
```

Validation requirements can also cause parser errors here:

**Not valid**:

```
name: string | age: int(18..)
bob
```

## Field Aliases

Nested fields may optionally be named. This allows them to be mapped to objects in whatever parser you are using:

```
parents: [father: [string, string], mother: [string, string]]
```

## Type Aliases

Types may be aliased using the `TYPE` comment:

```
#! TYPE name = [string, string]

parents: [father:name, mother:name]
Rob:Petrie ; Laura:Petrie
```

Type aliases may reference other aliases as long as they are already defined earlier in the SSV:

```
#! TYPE name = [string, string]
#! TYPE parents = [name, name]
```

## Custom Types

A `TYPE` command can also create custom types using regular expressions. Regular expressions follow JavaScript syntax with a minimum feature set:

```
#! TYPE email = /^.+@.+\..+$/

name: string | email: email
Bob          | bob@bob.com
```

Regex Features:

| Feature | Description                            | Example      |
| ------- | -------------------------------------- | ------------ |
| `.`     | Any single character                   | `a.c`        |
| `[]`    | Any one character inside               | `[aeiou]`    |
| `[^]`   | Aany one char NOT inside               | `[^aeiou]`   |
| `^`     | Start of a line                        | `^Start`     |
| `$`     | End of a line                          | `End$`       |
| `*`     | 0 or more                              | `a*`         |
| `+`     | 1 or more                              | `a+`         |
| `?`     | 0 or 1                                 | `a?`         |
| `\`     | Escapes a metacharacter                | `\.`         |
| `[-]`   | Character range                        | `[a-z]`      |
| `()`    | Grouping                               | `(ab)+`      |
| `\|`    | Alternation / OR                       | `(cat\|dog)` |
| `{n}`   | Exactly n repetitions                  | `[0-9]{3}`   |
| `{n,m}` | n to m repetitions                     | `[0-9]{3,5}` |
| `\d`    | Any digit, `[0-9]`                     |              |
| `\w`    | Any word character, `[a-zA-Z_]`        |              |
| `\W`    | Any non-word character, `[^a-zA-Z_]`   |              |
| `\s`    | Any whitespace character, `[ \t]`      |              |
| `\S`    | Any non-whitespace character, `[^ \t]` |              |

## Multiple Tables

SSV supports multiple tables in a single file. To add a new table, add any recognized [Parser Comment](#parser-comments) followed by your table. To name a table, use `#! TABLE`:

```
#! TABLE friends

name: string
Bob
Sue
Richard

#! TABLE colors

color: string
red
green
blue
```

Note: `#! TABLE` can also be used without a name to just mark a new table

### Isolating Tables

By default, parser comments apply to the entire file, and `#! TABLE` comments are the only comments that apply to each individual table. You can change this behavior with `#! ISOLATED_TABLES`:

- Any parser comment found after a table header resets _all_ settings to their defaults (except `ISOLATED_TABLES` itself). This includes resetting the delimiters, removing added types, etc.
- You can use this to concatenate as many SSV files together as you'd like, and they will all parse correctly regardless of their individual config as long as the first file has `ISOLATED_TABLES` set

## Markdown Tables

Because rows containing only the first delimiter, whitespace, and `-` characters are ignored by default, and because empty cells are ignored, markdown tables are valid in SSV files (assuming you keep `|` as the first delimiter) and any markdown formatter should be able to automatically format your data for you (more useful on config files than 100,000 line logs):

```
#! TYPE difficulty = uint8(0..3)
#! TYPE vector3 = [float, float, float]

| level    | diff:difficulty | spawn:vector3   |
| -------- | --------------- | --------------- |
| Forest   | 1               | 0.0;  1.2;  5.5 |
| Dungeon  | 3               | 10.0; 0.0; -2.0 |
```

To disable this (if you need lines like `| --- | --- |` to read as data for some reason), add:

```
#! DISABLE-MARKDOWN-SUPPORT
```

## Cheat Sheet of Valid Parser Comments

```
#! DECIMAL_SEPARATOR .
#! DELIMITERS | ;
#! DISABLE_BINARY_NUMBERS
#! DISABLE_EXPONENTIAL_NUMBERS
#! DISABLE_HEX_NUMBERS
#! DISABLE_OCTAL_NUMBERS
#! DISABLE_RADIX_NUMBERS
#! DISABLE_REGEX_CHECK
#! DISABLE-MARKDOWN-SUPPORT
#! ESCAPE_CHARACTER \
#! ISOLATED_TABLES
#! NUMERIC_SEPARATOR _
#! PARENTHETICAL_NEGATIVES
#! TABLE name
#! TYPE alias = [type1, type2]
#! TYPE alias = /regex/
```

## License

This project is licensed under your choice of any of the following licenses:

- 0BSD License (LICENSE-0BSD or https://opensource.org/license/0bsd)
- Apache License, Version 2.0 (LICENSE-APACHE or https://www.apache.org/licenses/LICENSE-2.0)
- MIT License (LICENSE-MIT or https://opensource.org/licenses/MIT)

Personally I suggest 0BSD as it doesn't even require keeping the license file. Do whatever you want. I don't care. If your legal department cares, the other two are available.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in this work by you, as defined in the Apache-2.0 license, shall be licensed as above, without any additional terms or conditions.
