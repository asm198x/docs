# Asm198x documentation

Documentation for the [Asm198x](https://github.com/asm198x) assembler family.

Start here:

- **[ISA spec format](isa-spec-format.md)** — the declarative instruction-set
  format that the `isa` crate holds and the assembler consumes. The single
  source of truth for instruction encoding across the family.
- **[6502 dialect](dialects/6502.md)** — the source syntax the 6502 assembler
  accepts today: addressing modes, directives, operators, and the gaps still to
  close on the way to ca65 compatibility.

## How the pieces fit

```
datasheet / reference library          (hardware facts, with provenance)
            │  authored into
            ▼
      isa spec  ──consumed by──▶  asm198x assembler  ──emits──▶  bytes
        │                              ▲
        │                              │ dialect (per-CPU parser)
        └──validated against by──▶  Emu198x decoders
```

The spec describes *encoding*; each CPU's *dialect* (how source syntax maps to
addressing modes) lives in the assembler. Hardware facts themselves are not
documented here — they live in the 198x primary reference library, and the spec
is their machine-readable distillation for the narrow slice of "how an
instruction encodes."

## Conventions

These docs are plain Markdown for now. A generated docs site can follow once
there's enough to warrant it; nothing here assumes one.
