# Fork notes (kkls-mike/serde_php-rs)

This is a fork of [`serde_php`](https://crates.io/crates/serde_php) `0.5.0`, maintained
for our PHP-serialized invoice data (10+ years of legacy records). It exists to make
deserialization **debuggable** and to let us decide how PHP strings are surfaced to
serde. This file records *why* the changes exist so the reasoning isn't lost.

---

## TL;DR — branch model

| Branch       | PHP string is delivered to serde as | Use when |
|--------------|-------------------------------------|----------|
| `master`     | a **byte sequence** (`visit_seq`) — upstream behavior | You hit genuinely binary / non-UTF-8 serialized data and need byte-faithful parsing. |
| `visit-str`  | a **string** (`visit_str`, falling back to `visit_bytes` for non-UTF-8) | Default. Our data is UTF-8 text + numbers; this gives readable errors and unlocks the standard serde ecosystem. |

Both branches share every other fix (see below). `visit-str` = `master` + the single
"Fix B" commit.

**Switching is one word** in the consuming project's `Cargo.toml`:

```toml
[patch.crates-io]
serde_php = { git = "https://github.com/kkls-mike/serde_php-rs", branch = "visit-str" }
#                                                                         ^^^^^^^^^^
# change to "master" to fall back to byte-faithful behavior
```

---

## Background: why PHP strings are awkward in serde

PHP's `serialize()` writes strings as `s:<BYTE_LENGTH>:"<raw bytes>";`. PHP strings are
**arbitrary byte buffers, not Unicode** — people serialize images, gzip blobs, encrypted
payloads, Latin-1 text, etc. Rust's `String`/`&str` *must* be valid UTF-8, so a
format-faithful deserializer cannot blindly hand every PHP string to `visit_str` (it
would error on the first non-UTF-8 byte).

### Why upstream chose `Vec<u8>` / `visit_seq` (the "oddball" choice)

It's deliberate, not careless:

1. PHP strings are binary → represent them as bytes, not text.
2. The author wanted the ergonomic owned container `Vec<u8>` to work out-of-the-box with
   no `serde_bytes` dependency.
3. **serde does not specialize `Vec<u8>`** — a plain `Vec<u8>` deserializes element-by-
   element through `visit_seq`, *not* `visit_bytes`. So to make `Vec<u8>` "just work" as
   the string type, the deserializer must emit `visit_seq`.
4. Uniform `visit_seq` is also **deterministic**: every row takes the same code path,
   regardless of whether its bytes happen to be valid UTF-8.

Upstream still offers `String` as a convenience via an explicit `deserialize_string`
(UTF-8 conversion); only the generic `deserialize_any` path uses the byte-sequence form.

### The cost (our friction)

`deserialize_any` is what forwarded scalars (`u64`, `bool`, ...), `serde_json::Value`,
and ecosystem crates like `serde_with` all go through. Because that path yields a
*sequence*, not a string:

- We were forced to write `visit_seq` byte-reconstruction visitors for values that are
  really just text/numbers (see `F64Visitor` in our app).
- Type-mismatch errors were unhelpful (`invalid type: sequence, expected u64`) instead of
  the standard, value-carrying `invalid type: string "34589", expected u64`.
- Standard tooling (`serde_with::DisplayFromStr`, etc.) doesn't work, because it expects
  `visit_str`.

`serde` itself is **strict by design** — it never coerces `"123"` into `123` for a `u64`
field; that's true of `serde_json` too. Coercion always requires opt-in
(`deserialize_with` / `serde_with`). The fork doesn't change that; it changes whether the
string reaches those tools as a *string* (so they can work) or as a *byte sequence*.

---

## Changes on `master` (shared by both branches)

### 1. Bug fix: error masking in `deserialize_map` / `deserialize_any`
Upstream ran `expect(b'}')?` **even after the visitor returned an error**, so the real
error (e.g. `invalid type: ..., expected u64`) was overwritten by a misleading structural
one (`Expected `}` but got `:`). We now short-circuit on the visitor error. This is a
plain correctness fix and is the difference between a cryptic message and a useful one.

### 2. `from_bytes_traced` + `TracedError`
A drop-in alternative to `from_bytes` that annotates errors with the **path to the failing
field** (via `serde_path_to_error`), e.g.:

```
at `invoice_details.client_id`: invalid type: ..., expected u64
```

This pulls in a `serde_path_to_error` dependency and re-exports a small `TracedError`
type, so the consuming app keeps a clean one-line call and doesn't carry the wiring itself.

### 3. `PhpDeserializer` + `PhpDeserializer::new` made public
Needed so `serde_path_to_error` (and any other wrapper) can drive the deserializer.

### 4. Fixed swapped `Display` labels on `Error`
`SerializationFailed` / `DeserializationFailed` had their human-readable strings swapped,
so deserialization errors printed "PHP Serialization failed". Corrected.

---

## Change on `visit-str` only — "Fix B"

In `deserialize_any`'s string branch, instead of always emitting `visit_seq`:

```rust
match std::str::from_utf8(&data) {
    Ok(s) => visitor.visit_str(s),   // readable, value-carrying errors; ecosystem-compatible
    Err(_) => visitor.visit_bytes(&data), // still binary-safe for non-UTF-8
}
```

**Tradeoff (intentional):**
- Breaks the documented `Vec<u8>` ↔ PHP-string mapping and 2 upstream tests
  (`deserialize_php_string`, `deserialize_array`). We don't want `Vec<u8>` strings, so
  this is acceptable on this branch.
- Loses upstream's *determinism*: a field that's usually ASCII (`visit_str`) but has one
  row with non-UTF-8 bytes (`visit_bytes`) takes different paths per row. For our invoice
  text/numbers this is a non-issue; if it ever bites, switch back to `master`.

---

## How the consuming app stays branch-agnostic

A serde `Visitor` may implement several `visit_*` methods; serde calls whichever the
active deserializer provides. **Write conversion visitors that implement *both*
`visit_str` and `visit_seq`** (plus `visit_i64`/`visit_u64`/`visit_bool` for values PHP
stored natively) and the *same* app code works on either branch with zero changes.

Our `F64Visitor` (for `Amount` / `ServicePercent`) is the template: it already handles
`visit_str`, `visit_seq`, and the native numeric/bool forms. New helpers for
string-encoded integer/bool fields (`client_id`, `process_credit_cards`, ...) should
follow the same shape. Then switching the fork branch is purely a fork-side flip.

---

## Upstreaming

These changes (except possibly Fix B) are arguably worth upstreaming — especially the
error-masking bug fix. Fix B is opinionated and probably not upstream-friendly. Keep
`master` close to upstream so the diff stays reviewable.
