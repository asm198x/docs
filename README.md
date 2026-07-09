# Asm198x documentation

Documentation for [Asm198x](https://github.com/asm198x), the 198x family's assembler/disassembler tooling and shared ISA-spec layer.

## Start here

- **[ISA spec format](isa-spec-format.md)** — the declarative instruction-set format held by the `isa` crate. This is the single source of truth for instruction encoding across Asm198x and downstream consumers.
- **[Debug198x format](debug198x.md)** — the cross-CPU debug-info sidecar (`.debug198x`, NDJSON) emitted by asm198x and read by Emu198x importers.
- **[6502 dialect](dialects/6502.md)** — source-syntax reference for the 6502 family dialects. Treat older gap notes in dialect docs as document-local until checked against the current `asm198x` test suite.

## How the pieces fit

```text
datasheet / reference library
          │ authored into
          ▼
      isa spec ──consumed by──▶ asm198x assembler ──emits──▶ bytes + listings + symbols + debug sidecars
          │                         │
          │                         └── dialect front-ends map source syntax to encoding forms
          │
          ├──consumed by──▶ isa-disasm disassemblers
          └──validated with──▶ Emu198x decoder/oracle work where applicable
```

The spec describes instruction **encoding**. Each CPU's dialect — how source syntax maps to addressing modes, directives, labels, and output formats — lives in the assembler. Hardware facts themselves are not canonical here; they live in the 198x primary reference library and syntheses.

## Current status

The docs repo is an index and external-format reference, not the exhaustive implementation ledger. For the active crate layout, CPU surface, and validation model, use [`../asm198x/README.md`](../asm198x/README.md) and [`../asm198x/CLAUDE.md`](../asm198x/CLAUDE.md).

## Conventions

These docs are plain Markdown for now. A generated docs site can follow once there is enough stable user documentation to warrant it.
