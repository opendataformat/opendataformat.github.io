# ODF Data and Serialization Specification

**Status:** Draft for ODF 1.1.x  
**Metadata profile:** OpenDF DDI-Codebook 2.5 Profile 1.0.1  
**Scope:** ZIP container, `data.csv`, and their relationship to `metadata.xml`.

## 1. Conventions and scope

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

This specification is normative for the data and packaging layer. The [ODF DDI profile](profile.xml) is normative for the metadata layer. In case of conflict, the profile governs the permitted DDI metadata structure and this document governs the ZIP and CSV serialization.

This document intentionally does not define a universal semantic model for missing values. It defines how implementations mark and report loss of missing-value information instead of silently inventing a cross-software mapping.

## 2. Conformance

An implementation is a conforming **Writer** if it creates an ODF package that meets every applicable MUST and MUST NOT requirement in Sections 3–8.

An implementation is a conforming **Reader** if it accepts every package meeting Sections 3–8, validates the required cross-file relationships, and applies the warning behavior in Section 8.

A **round trip** means Writer A → ODF package → Reader A. A Reader MUST NOT claim a lossless round trip for a property that is not represented in the ODF package.

## 3. ODF package

An ODF package MUST be a ZIP archive with the extension `.odf.zip`. It MUST contain these two root-level members:

| Member | Requirement | Purpose |
|---|---|---|
| `data.csv` | REQUIRED | Tabular data serialized according to Section 4. |
| `metadata.xml` | REQUIRED | DDI-Codebook 2.5 metadata conforming to the ODF profile. |

The archive MAY contain additional files. An additional file MUST NOT replace, shadow, or change the meaning of a required member.

ZIP member names MUST be UTF-8, use `/` as a separator, and MUST NOT be absolute paths or contain `..` path segments. Writers SHOULD use the ZIP DEFLATE method or store members uncompressed. Readers MUST accept stored and DEFLATE-compressed members.

Writers SHOULD place the two required members first, in the order shown above, to support streaming readers. Member order is not semantically significant.

## 4. CSV data file

`data.csv` MUST conform to [RFC 4180](https://www.rfc-editor.org/rfc/rfc4180), with the ODF clarifications in this section.

1. The file MUST be UTF-8 encoded and MUST NOT include a byte-order mark.
2. Records MUST use CRLF (`\r\n`) line endings. A Writer SHOULD terminate the final record with CRLF; a Reader MUST accept either form permitted by RFC 4180.
3. The first record MUST be a header record. It MUST contain one field for each variable, in the same order as `dataDscr/var` in `metadata.xml`.
4. Each header field MUST equal the corresponding `var/@name` exactly, including case. Variable names MUST be unique.
5. Every data record MUST contain exactly the same number of fields as the header.
6. Fields containing a comma, quotation mark, CR, or LF MUST be double-quoted. A quotation mark in a quoted field MUST be written as two quotation marks.
7. A comma is the only delimiter. Tab, semicolon, locale-dependent decimal separators, and whitespace trimming MUST NOT be used as dialect features.
8. A Reader MUST preserve leading and trailing whitespace in an unquoted field. A Writer SHOULD quote character values whose leading or trailing whitespace is significant.

The package MIME type, when one is supplied, SHOULD be `application/zip`; the embedded data file MIME type is `text/csv; charset=utf-8; header=present`.

## 5. Metadata-to-data relationship

`metadata.xml` MUST conform to the ODF profile. In particular:

- `codeBook/stdyDscr`, its citation, title statement, and study title are REQUIRED.
- `codeBook/fileDscr/fileTxt/fileName` is REQUIRED.
- `codeBook/dataDscr` and each `dataDscr/var` are REQUIRED.
- Each `var/@name` is REQUIRED when `var` is present.
- `varFormat` is OPTIONAL, but if a `varFormat` element is present, `varFormat/@type` is REQUIRED. Its permitted values are `character` and `numeric`; its DDI default is `numeric`.
- Textual labels and descriptions MAY omit `@xml:lang`; an omitted language tag means default or unspecified language. When supplied, a Writer SHOULD use an ISO 639-1 language code.
- Dataset notes, if used by the profile, belong at `fileDscr/notes`.

The number and order of `dataDscr/var` elements MUST equal the number and order of CSV header fields. A Reader MUST treat a mismatch as an error, not guess a mapping by display label.

`varFormat/@type` MUST be `numeric` for numeric lexical values and `character` for character values. A Writer MUST NOT declare a numeric column as character, or the reverse, merely to avoid a conversion problem.

## 6. Canonical value serialization

### 6.1 Character values

Character values MUST be Unicode text encoded as UTF-8. A Writer MUST NOT apply locale-dependent transliteration or silently normalize a value. A Reader MUST return the exact sequence of Unicode code points represented by the CSV field, except for RFC 4180 quote decoding.

An empty character field is a zero-length string at the CSV level. Because the current ODF profile has no interoperable marker that distinguishes an empty string from a missing character value, a Writer MUST issue the warning in Section 8 when that distinction is not preserved.

### 6.2 Integers

An integer MUST be serialized in base 10, with an optional leading minus sign and no grouping separators. For example: `0`, `42`, and `-9`.

A numeric sentinel such as `-9` is an ordinary numeric value. A Writer MUST serialize it literally and MUST NOT convert it to a blank field solely because it is conventionally used as a missing-value code. Its human meaning MAY be recorded in labels or notes in `metadata.xml`.

### 6.3 Floating-point values

Numeric fields MUST use `.` as the decimal separator. `NaN`, `Inf`, `Infinity`, and locale-specific strings MUST NOT be used as numeric values.

For IEEE 754 values, a Writer MUST use a decimal representation that round-trips to the same binary value in a correctly rounded parser of the corresponding precision:

| Source storage | Minimum significant decimal digits | Rationale |
|---|---:|---|
| IEEE 754 binary32 (`float`) | 9 | Sufficient for a binary32 value to round trip. |
| IEEE 754 binary64 (`double`) | 17 | Sufficient for a binary64 value to round trip. |

A Writer MAY emit more digits only when they still describe the source value exactly. A Reader MUST NOT parse numeric data through a locale-sensitive conversion. It SHOULD use a parser that preserves all supplied significant digits before choosing its in-memory storage type.

The portable DDI core records only `numeric` or `character` in `varFormat/@type`; it does **not** record whether a numeric source column was an integer, binary32 float, or binary64 double. The software-specific behavior for those types is documented in Section 6.5. Cross-implementation conformance covers the serialized numeric value, not recovery of a source application's storage type.

### 6.4 Dates and times

Dates and times MUST NOT be serialized as software-specific numeric epochs unless their interpretation is fully represented by interoperable metadata.

For portable textual serialization, Writers SHOULD use ISO 8601 lexical forms:

| Logical value | Canonical form | Example |
|---|---|---|
| Date | `YYYY-MM-DD` | `2026-11-02` |
| Time | `hh:mm:ss[.fraction]` | `14:05:09.125` |
| Date-time | `YYYY-MM-DDThh:mm:ss[.fraction][Z or ±hh:mm]` | `2026-11-02T14:05:09Z` |

The current profile does not provide a mandatory, interoperable logical-date declaration. Therefore a Reader MUST NOT infer that a numeric or character column is a date solely from its appearance. Implementations MAY retain a native display format as implementation metadata, but MUST warn if that format is needed to reconstruct date semantics and is not represented in the ODF package.

### 6.5 Software-specific conversion notes (non-normative)

This section describes package behavior. It does not add metadata elements or attributes to the ODF profile, and it does not create a cross-software type guarantee.

#### Stata

The Stata package has two distinct conversion paths.

- In its direct four-CSV workflow, `variables.csv` records `stata_storage_type` and `stata_format`. The importer uses those columns to restore Stata `byte`, `int`, `long`, `float`, `double`, string storage, and display formats, including Stata date and date-time formats. This is a Stata-package CSV convention, not an ODF ZIP metadata requirement.
- For direct CSV export, Stata writes floats with 9 significant decimal digits, doubles with 17, and integers without decimal rounding. Thus the serialized lexical value is sufficient to recover the same `float` or `double` when the Stata-specific CSV metadata is available.
- In the ODF ZIP path, a Stata Reader imports numeric data as `double`. It MUST NOT claim that the original Stata storage type or display format has been recovered from the ODF profile alone.
- Stata `.a` through `.z` extended missing values are collapsed to ordinary missing for ODF ZIP export with a warning. Ordinary numeric sentinel values, including `-9`, remain numeric values and are written literally.

#### Python/Pandas

- A profile-conformant ODF interchange uses only `varFormat/@type` and CSV lexical values. Pandas then applies its normal import conversion; the original pandas dtype, nullable dtype, and logical date-time dtype are not guaranteed to be reconstructed from the profile.
- The current Python package writes numeric output with `%.17g`, uses CRLF line endings in its round-trip mode, and can restore pandas dtypes only by writing custom namespaced attributes and a missing-value marker in `metadata.xml`.
- Those custom pandas attributes are **not** part of the current ODF profile. Therefore they are outside this specification and a package containing them MUST NOT be described as profile-conformant solely on that basis. A profile-conformant Python writer MUST disable or remove them; its behavior is then limited to the portable CSV and DDI rules in this document.
- Pandas date-time values SHOULD be serialized as ISO 8601 text when portable exchange is intended. Without an interoperable logical-date declaration, another Reader cannot safely infer that a text column is a date-time column.

## 7. Labels and language

Variable labels, variable descriptions, category labels, study titles, and dataset labels MUST be written in `metadata.xml`, not in CSV headers except for the required `var/@name` mapping.

`@xml:lang` is OPTIONAL for these text elements. A Writer MAY omit it for the default language; a Reader MUST accept both tagged and untagged text. A Reader MUST NOT manufacture a language tag for untagged text.

## 8. Missing values and warnings

At the portable CSV data layer, an empty field represents a missing or absent serialized value. This is a transport marker, not a complete semantic missing-value model.

The following rules apply:

1. A Reader MUST interpret an empty numeric field as a missing numeric value.
2. A Writer encountering a source-system extended missing value that has no ODF representation—such as Stata `.a` through `.z`—MUST write an empty field and MUST issue a warning identifying the column and the count of values collapsed to generic missing.
3. A Writer MUST NOT map an extended missing value to an arbitrary numeric sentinel such as `-9` unless the user explicitly requests that transformation.
4. A Reader MUST NOT infer a specific extended missing category from an empty field.
5. A Writer MUST warn when it cannot distinguish a missing character value from a valid zero-length string after serialization.
6. A Writer SHOULD preserve any source-system missing-value labels in the available metadata labels or notes, but such labels MUST NOT be presented as proof that the original missing category is recoverable.

Warnings MUST be visible to the calling user or calling program. A warning SHOULD name the variable, the number of affected values, the source representation when known, and the resulting ODF representation. An implementation MAY offer a strict mode that turns these warnings into errors.

## 9. Validation and error handling

Before finalizing a package, a Writer MUST validate:

1. the required archive members;
2. CSV header names, field counts, UTF-8 encoding, and RFC 4180 quoting;
3. the one-to-one ordered mapping between CSV headers and `var/@name` values; and
4. profile requirements, including `varFormat/@type` when `varFormat` exists and its permitted values.

A Reader MUST fail with an error for malformed ZIP data, invalid UTF-8, invalid CSV quoting, duplicate header names, header/metadata mismatches, or missing required members. It MUST use warnings, rather than silent coercion, for representational loss described in Section 8.

## 10. Interoperability boundary and future extensions

This version defines a portable core: CSV field values, DDI profile metadata, UTF-8 text, and explicit missing-value loss warnings. The following are deliberately outside the current interoperable core because the profile does not yet encode them:

- original numeric storage width (`byte`, `int`, `long`, `float`, `double`);
- native display formats and date epochs;
- multiple distinct system-missing categories; and
- a portable checksum or persistent identifier scheme.

An implementation MAY retain such information privately, but MUST NOT rely on private information for cross-implementation conformance. Any interoperable extension MUST first define its metadata location and validation rule in the ODF profile, then update this specification.

## 11. References

- Bradner, S. (1997). [RFC 2119: Key words for use in RFCs to indicate requirement levels](https://www.rfc-editor.org/rfc/rfc2119).
- Shafranovich, Y. (2005). [RFC 4180: Common format and MIME type for comma-separated values (CSV) files](https://www.rfc-editor.org/rfc/rfc4180).
- Open Data Format. [DDI-Codebook 2.5 profile](profile.xml).
