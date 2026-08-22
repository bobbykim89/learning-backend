---
title: Text encoding and Unicode
phase: 00-mental-models
status: learning
updated: 2026-08-17
---

## What it is

Three different things that all get called "a character", and most text bugs are a
confusion between two of them:

1. **Byte** — what is stored and transmitted.
2. **Code point** — a number Unicode assigns to a character, written `U+0041`.
3. **Grapheme cluster** — what a reader perceives as one character, which may be
   several code points.

`é` can be one code point (`U+00E9`) or two (`e` + combining acute `U+0301`).
The family emoji is one grapheme cluster and several code points. An emoji with a
skin-tone modifier is one perceived character, two code points, and eight bytes in
UTF-8.

**Encodings** map code points to bytes:

- **UTF-8** — variable width, 1 to 4 bytes. ASCII is unchanged, so ASCII text is
  valid UTF-8. Self-synchronising and the correct default for essentially
  everything.
- **UTF-16** — 2 bytes for most code points, 4 (a *surrogate pair*) for the rest.
  This is what JavaScript strings are made of, and what Java and Windows APIs use
  internally.
- **UTF-32** — 4 bytes always. Simple indexing, wasteful.

### Why "length" has no single answer

The runtimes disagree, and the disagreement is not a bug in either:

```python
s = "é"          # e + combining acute -> renders as é
len(s)                 # 2  — Python counts CODE POINTS
len(s.encode())        # 3  — bytes in UTF-8
```

```javascript
"👨‍👩‍👧".length         // 8 — JS counts UTF-16 CODE UNITS
[..."👨‍👩‍👧"].length    // 5 — spread iterates code points
// grapheme clusters need Intl.Segmenter
[...new Intl.Segmenter().segment("👨‍👩‍👧")].length   // 1
```

So a "max 20 characters" rule means at least four different things depending on
where it is enforced. A validation layer counting code points, a database column
counting something else, and a UI counting grapheme clusters will disagree, and the
disagreement surfaces as data that passes validation and then fails to store.

**Truncating by bytes splits characters.** Cutting UTF-8 at an arbitrary byte
offset can leave half a character, producing invalid UTF-8 that some consumers
reject and others render as a replacement character.

### Normalization

Unicode permits multiple encodings of the same text, so equality needs a
canonical form:

- **NFC** — composed. `é` becomes the single code point. The usual choice.
- **NFD** — decomposed. `é` becomes `e` + combining accent.
- **NFKC / NFKD** — *compatibility* forms, which additionally fold things like `ﬁ`
  to `fi` and full-width characters to ASCII. Lossy, and therefore dangerous for
  identifiers, useful for search.

The consequence: **two strings that look identical can be unequal**, and can hash
differently. So a user registers `josé` in NFC, logs in with an NFD keyboard, and
the lookup fails. Or worse, both rows exist. macOS filesystems have historically
stored filenames decomposed, so the same filename read on macOS and Linux is
literally different bytes.

The rule that follows: **normalize at the boundary, once, then compare.** Pick NFC
for storage, and be careful that normalization happens *before* validation, not
after — normalizing after a security check lets an attacker slip a string past the
check that becomes something else afterwards.

### Case, and why lowercasing is not a comparison

`"İ".lower()` is not `"i"` in Turkish; `ß` uppercases to `SS`, which is two
characters. Python has `str.casefold()` for caseless comparison, which is not the
same as `lower()`. Case-insensitive matching is locale-dependent, and using it for
security decisions — matching usernames, comparing tokens — is a bug generator.
Compare bytes for secrets, and normalized code points for identifiers.

**Homoglyphs** are the security face of this: Cyrillic `а` (U+0430) and Latin `a`
(U+0061) render identically in most fonts. Identifiers that admit arbitrary
Unicode admit impersonation, which is what internationalised-domain-name homograph
attacks exploit.

## Why it exists (what came before)

Before Unicode, every language had its own byte encoding: Latin-1 for Western
Europe, Shift-JIS for Japanese, KOI8-R for Russian, and dozens more. A byte stream
carried no indication of which, so text was only interpretable if you already knew
— which is why the web spent a decade rendering `Ã©` where `é` was meant. That
failure has a name in Japanese, *mojibake*, because it happened constantly.

Unicode's answer was one code space for every script. UTF-16 came first, on the
assumption that 65,536 code points would be enough — hence UTF-16's surrogate
mechanism, added when it turned out not to be, and hence JavaScript and Java
carrying a 2-byte internal representation forever. UTF-8's design was the
retrofit that made adoption possible: it is byte-compatible with ASCII, so every
existing ASCII file and protocol became valid without change.

## Smallest example

```python
import unicodedata as ud

a = "café"                      # composed:   c a f é
b = "café"                # decomposed: c a f e ́
print(a == b)                   # False  — they render identically
print(len(a), len(b))           # 4 5
print(a.encode(), b.encode())   # 5 bytes vs 6 bytes

print(ud.normalize("NFC", b) == a)   # True — normalize, then compare

# Truncating by bytes breaks characters
s = "naïve"
raw = s.encode()
print(raw[:3])                       # b'na\xc3'  -> invalid UTF-8
print(raw[:3].decode(errors="replace"))   # 'na�'

# Caseless comparison, done properly
print("Straße".casefold() == "STRASSE".casefold())   # True
print("Straße".lower() == "STRASSE".lower())         # False

# Homoglyph: identical on screen, different data
print("аdmin" == "admin", ud.name("а"))   # False, CYRILLIC SMALL LETTER A
```

```javascript
const a = "café";
const b = "café";
console.log(a === b);                       // false
console.log(a.normalize("NFC") === b.normalize("NFC")); // true

// The three lengths
const s = "👍🏽";
console.log(s.length);                                     // 4 UTF-16 units
console.log([...s].length);                                // 2 code points
console.log([...new Intl.Segmenter().segment(s)].length);   // 1 grapheme

// Byte length is a fourth number, and the one a DB column may care about
console.log(Buffer.byteLength(s, "utf8"));                 // 8
```

## Tradeoffs & when it's wrong

**NFC everywhere at the boundary** is right as a default: one canonical form,
stable comparisons, minimal surprise. It costs a normalization pass on input, and
it is *wrong* to apply blindly to data you must return byte-exact — cryptographic
material, signatures over exact bytes, or content whose fidelity is the point.

**NFKC for identifiers** resists some spoofing by folding compatibility variants,
and is lossy: it destroys distinctions that matter in some scripts, and it will
mangle legitimate names. Restricting identifiers to a limited script set is
usually a better tool than aggressive folding.

**Counting grapheme clusters for length limits** matches what users perceive and
costs a segmentation dependency plus a limit that no database column can enforce.
Counting bytes matches storage and produces limits users find arbitrary. Pick per
field and document it: display fields by graphemes, storage-bound fields by bytes.

**Allowing arbitrary Unicode in usernames** is inclusive and correct for human
names; it also admits homoglyph impersonation. The resolution is usually two
fields — a display name that is permissive, and a login identifier that is
restricted and normalized.

What would change these: whether the field is compared, displayed, or stored, and
whether an attacker chooses its contents.

## Failure modes & operational cost

- **Mojibake.** Bytes decoded with the wrong encoding. Usually a missing charset
  declaration, or a database connection whose encoding differs from the data.
- **Double encoding.** Text encoded to UTF-8, then treated as Latin-1 and encoded
  again. Recoverable only by guessing.
- **MySQL's `utf8`.** Historically an alias for a 3-byte subset that cannot store
  4-byte characters — so emoji and some CJK fail. `utf8mb4` is real UTF-8. This has
  broken a great many production inserts.
- **Length limits that disagree across layers.** Validation passes, storage
  truncates or errors.
- **Normalizing after validating.** A string that passed a filter becomes something
  else afterwards — a real bypass technique, and the same shape as path-traversal
  normalization bugs.
- **Sorting and collation.** Byte order is not alphabetical in any language;
  collation is locale-dependent, and changing a database's collation changes index
  ordering and uniqueness semantics.
- **Lone surrogates from JavaScript.** A UTF-16 string can hold an unpaired
  surrogate, which is not valid Unicode and cannot be encoded to UTF-8 — an
  exception on serialisation, from data a client happily produced.
- **Byte-order marks.** A BOM at the start of a UTF-8 file is legal and invisible,
  and breaks naive parsers of CSV, JSON, and shell scripts.
- **Case-insensitive uniqueness.** Two accounts that differ only by case or by
  normalization form, both created, neither reachable reliably.

## Open questions / to verify

Not checked against primary sources in this session:

- Whether the databases used here count `varchar(n)` in characters or bytes, and
  what the default collation is.
- Whether the frameworks in use normalize incoming request bodies or query strings
  at all — the assumption should be that they do not.
- Whether `Intl.Segmenter` is available in the Node version in use and in the
  target browsers.
- Python's default filesystem encoding and error handler on this platform, and how
  they interact with filenames that are not valid UTF-8.
- Whether the JSON encoders here reject lone surrogates or emit replacement
  characters.
- What normalization form, if any, the identity provider applies to email addresses
  and usernames before they reach the application.

Candidate for `practiced`: add a normalization boundary to a real signup flow, then
try to register two accounts whose names differ only by normalization form, and
one pair differing only by a Cyrillic homoglyph. Report which the system rejects
before and after the change.

## Sources

- [Unicode Standard Annex #15 — Normalization Forms](https://unicode.org/reports/tr15/)
- [Unicode Standard Annex #29 — Text Segmentation (grapheme clusters)](https://unicode.org/reports/tr29/)
- [Unicode Technical Report #36 — Security Considerations](https://unicode.org/reports/tr36/)
- [RFC 3629 — UTF-8](https://www.rfc-editor.org/rfc/rfc3629)
- [Python `unicodedata`](https://docs.python.org/3/library/unicodedata.html)
