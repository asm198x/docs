# The Debug198x format

> **Status: draft v0.1 — subject to change until the first consumer ships.**
> The format freezes when a real consumer (the Emu198x importer) has exercised
> it end-to-end; until then, field names and record shapes may still move. The
> freeze event and the rules for evolution afterwards are governed by the
> [format decision record](https://github.com/asm198x/asm198x/blob/main/decisions/debug198x-format.md).

Debug198x is the 198x family's **cross-CPU debug-info format**: one sidecar
file that maps assembled bytes back to source lines and symbols, identically
for a C64, Spectrum, NES, Amiga, or Intellivision program. Asm198x writes it
(`--debug`); a debugger — first the Emu198x importer — reads it to render
`jsr init` instead of `jsr $c012`, resolve a breakpoint set on a source line
to an address, and highlight the line when it hits.

This page is the format's specification, written so that you can **implement a
reader from this page plus the
[conformance fixtures](https://github.com/asm198x/asm198x/tree/main/crates/asm198x/tests/fixtures/debug198x)
without reading Asm198x source**. The fixtures are the spec's executable half:
each is a source file plus its exact expected sidecar, enforced in CI.

## The file

A `.debug198x` file is **NDJSON**: UTF-8 text, one JSON object per line. Every
record carries a `t` field naming its type. Four types exist today:

| `t` | one per file? | carries |
|-----|---------------|---------|
| `header` | yes, first line | format + tool identity, CPU, dialect, sources |
| `section` | one per section | id, name, optional absolute base |
| `symbol` | one per symbol | name, kind, location or value |
| `line` | one per source-bearing span | file, line, section, offset, length |

Two rules make the format durable:

- **A reader must skip records with an unknown `t`** instead of failing. New
  record types are added this way, without a version break.
- **A reader must ignore unknown fields** inside a known record. New fields are
  added this way.

Numbers are **decimal JSON integers** throughout, u64-capable. Hexadecimal is a
rendering concern for tools, never a wire concern.

## `header` — identity

```json
{"t":"header","format":"debug198x","format_version":"0.1","tool":"asm198x","tool_version":"0.0.7","cpu":"z80","dialect":"pasmo","sources":["game.z80"]}
```

| field | meaning |
|-------|---------|
| `format` | always `"debug198x"` |
| `format_version` | the version of *this specification* the file conforms to |
| `tool`, `tool_version` | the producing tool — informational, never load-bearing |
| `cpu` | the target CPU (`"z80"`, `"6502"`, `"68000"`, `"cp1610"`, …) |
| `dialect` | the source syntax (`"pasmo"`, `"acme"`, `"ca65"`, `"vasm"`, `"asl"`, …) |
| `sources` | the source file(s) the image was assembled from |

Branch on `format_version` for a breaking change; additive changes never bump
it incompatibly.

## `section` — where bytes live

Every address in the file is a **(section, offset)** pair. A section is a
segment of the image with an id, a human-readable name, and — when the
producer knows it — an absolute base address.

```json
{"t":"section","id":4,"name":"CODE","base":32768}
{"t":"section","id":0,"name":"code"}
```

| field | meaning |
|-------|---------|
| `id` | the integer other records reference |
| `name` | the segment/hunk name (`"main"`, `"CODE"`, `"ZEROPAGE"`, `"code"`, `"bank1"`) |
| `base` | absolute address of offset 0 — **absent when the section is relocatable or not CPU-addressable** |

Three base postures cover every producer:

- **Flat and linked-absolute images** (the flat engine's single `main` section,
  ca65's NES segments) carry their real base: absolute addressing is the
  degenerate case of (section, offset), with zero ceremony for the reader.
- **Relocatable sections** (Amiga hunks) carry no base. The reader's caller
  supplies a **base map** — section id → loaded address — at lookup time; see
  *The consumer model* below.
- **Non-CPU-addressable sections** (the NES iNES header, CHR data in PPU
  space) carry no base either, so they can never alias a CPU address in a
  lookup. A consumer that addresses another space (a PPU viewer) supplies its
  own base map for them.

## `symbol` — names

```json
{"t":"symbol","name":"start","kind":"label","section":0,"offset":0}
{"t":"symbol","name":"start","kind":"entry","section":0,"offset":0}
{"t":"symbol","name":"BORDER","kind":"const","value":254}
{"t":"symbol","name":"draw","kind":"label","section":0,"offset":16,"space":{"slot":3,"page":1}}
```

| `kind` | fields | meaning |
|--------|--------|---------|
| `label` | `section`, `offset`, optional `space` | a code/data location |
| `entry` | `section`, `offset`, optional `space` | the program's entry point (an `end <addr>` targeting a label upgrades that label in place — one symbol, not two) |
| `const` | `value` | an `equ`/`=` binding — a value, never an address |

The `const`/`label` split is load-bearing: a constant sharing its low bits with
a label's address stays distinguishable, and constants never answer
address-to-symbol lookups.

### `space` — the address-space qualifier

Most records carry no `space` field: a flat CPU's addresses need no
qualification, and **producers never fabricate one** (a Z80 file contains no
bank data; a 65816 program placed in bank 0 carries plain addresses). Two
shapes exist for machines that need more:

- `{"bank": 126}` — a 65816-style bank byte, the high 8 bits of a 24-bit
  address.
- `{"slot": 3, "page": 1}` — a banked/paged location: hardware slot `slot`
  with memory page (bank) `page` paged into it. Two symbols sharing a CPU
  address in different pages stay distinct.

The paged shape is exercised today by the hand-authored Spectrum 128 fixture
(`spectrum128-banked.*`); emission paths populate it when a machine needs it.
The fixture's companion table (`spectrum128-banked-sld.md`) shows every banked
record projecting onto sjasmplus SLD long addresses by pure arithmetic — the
record model loses nothing an SLD consumer needs.

## `line` — the source map

```json
{"t":"line","file":"game.z80","line":4,"section":0,"offset":0,"length":2}
```

`length` units at `(section, offset)` were produced by line `line` of `file`.
One record per source-bearing statement. Two rules:

- **The padding rule:** bytes with no source — `org` gap fill, alignment
  padding (an Amiga `even`, a linker gap) — belong to **no** line record. A
  reader asked about such an address answers "no source here", which is the
  truth.
- Lines that emit nothing (`equ`, comments, blank lines) get no record; their
  text is in the source file the header names.

## Address units

Offsets, lengths, and bases are in the **CPU's own address units** — what the
programmer's addresses mean on that machine. For every byte-addressed CPU that
is bytes. For the word-addressed CP1610 (Intellivision) it is **decles**
(16-bit words): a 12-byte image is 6 decles, its spans total 6, and a label's
address is a decle address, matching how every Intellivision address is
written. The `cp1610-intellivision` fixture pins this: span lengths sum to
half the image's byte count.

## The consumer model

A reader implements three lookups over the records (the fixtures exercise all
three):

- `addr_of(name)` — symbol name → absolute address.
- `symbol_at(addr)` — absolute address → the symbol defined there.
- `line_at(addr)` — absolute address → the line record covering it.

Each takes an optional **base map** (section id → absolute base). Resolution
of a record's absolute address is: *the base map's entry for its section if
present, else the section's recorded `base`, else the record does not resolve*
— it is silently outside the addressable world for this lookup.

That last clause is the design's hinge, and it is deliberate:

- **A relocatable program** (Amiga hunks) resolves once the loader's actual
  hunk addresses are supplied as the base map.
- **A banked machine's paging state *is* the base map.** Map only the sections
  currently paged in: with bank 1 in slot 3, `symbol_at($C010)` names `draw`;
  swap the map to bank 3 and the same address names `music`. No map, no
  answer — which is correct, because without paging state the question is
  ambiguous.
- **Non-CPU sections never pollute CPU lookups**, because nothing maps them.

A base-map entry always wins over a recorded `base`, so a consumer can rebase
even a based section (a flat blob loaded somewhere unusual).

## Producer guarantees

- Writing a sidecar **never changes the assembled image**: bytes with and
  without `--debug` are identical. The fixture corpus asserts this for every
  family.
- Addresses are **post-link** (what a debugger needs), never file offsets:
  a linker-placed NES label resolves to its ROM address.
- Sidecars are deterministic for a given source and tool version.

## Changelog

- **0.1** (2026-07-06) — initial draft: `header`/`section`/`symbol`/`line`
  records; label/entry/const symbol kinds; bank and paged space shapes; the
  section/offset addressing model with base-map rebasing; the padding rule;
  address units; unknown-record and unknown-field tolerance.
