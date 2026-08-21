# The `asm198x` command line

This page has moved into the assembler's own repository and is published at
**<https://asm198x.github.io/reference/cli/>**.

It moved because it had drifted. Maintained here by hand, it fell behind the
binary in two ways nobody caught: five working dialects were missing from it
entirely, and the ROM-less MCS-48 parts were listed as *aliases* of the 8048
when they refuse instructions the 8048 accepts — so following this page would
have picked the wrong one and produced a confusing failure on a working
program.

The reference now lives at [`docs/book/src/reference/cli.md`](https://github.com/asm198x/asm198x/blob/main/docs/book/src/reference/cli.md)
in `asm198x/asm198x`, and its dialect table is generated from the same list
`--dialect` resolves against. CI fails if the two disagree, so the drift that
prompted the move is now a build failure rather than a discovery months later.

This repo keeps the specifications it owns:

- [`debug198x.md`](debug198x.md) — the Debug198x sidecar format, frozen at v1
- [`isa-spec-format.md`](isa-spec-format.md) — the ISA spec format
- [`dialects/`](dialects/) — per-dialect references
