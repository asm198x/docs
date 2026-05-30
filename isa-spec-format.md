# The ISA spec format

The `isa` crate is the **single source of truth for instruction encoding** in
the 198x family: mnemonic ↔ opcode bytes ↔ operand layout ↔ cycle counts ↔
affected flags. The assembler consumes it to turn source into bytes; the
disassembler will consume it to turn bytes back into source; Emu198x validates
its hand-written decoders against it.

The spec is **authored**, not generated from any emulator. Its facts come from
the primary reference library (datasheets, programmer's manuals); when the spec
and a datasheet disagree, the datasheet wins and the spec is corrected.

Everything is `&'static` data, so a whole instruction set is a compile-time
constant: zero dependencies, no allocation, and diffable in review.

## The types

A CPU's instruction set is an [`InstructionSet`]: a name, a byte order, and a
list of instructions.

```rust
pub struct InstructionSet {
    pub cpu: &'static str,          // "MOS 6502"
    pub endianness: Endianness,     // Little | Big
    pub instructions: &'static [Instruction],
}
```

Each `Instruction` is one mnemonic and every way it can be encoded — one `Form`
per addressing mode.

```rust
pub struct Instruction {
    pub mnemonic: &'static str,     // "LDA"
    pub summary: &'static str,      // "Load accumulator"
    pub forms: &'static [Form],
}
```

A `Form` is a single concrete encoding.

```rust
pub struct Form {
    pub opcode: &'static [u8],      // fixed opcode bytes: &[0xA9]
    pub mode: &'static str,         // addressing-mode label: "immediate"
    pub operands: &'static [Operand],
    pub cycles: Cycles,
    pub flags: &'static str,        // affected status flags: "NZ"
    pub undocumented: bool,
}
```

- **`opcode`** is the fixed leading bytes. One byte for the 6502; a prefix
  sequence for prefixed opcodes on other CPUs (e.g. a Z80 `&[0xCB, 0x40]`).
- **`mode`** is the addressing-mode label — *the shared contract* between the
  spec and each CPU's dialect parser (see [The mode-label contract](#the-mode-label-contract)).
- **`operands`** are the bytes emitted after the opcode.
- **`cycles`** is timing (see [Cycles](#cycles)).
- **`flags`** names the status bits the instruction affects, as a compact
  string (`"NZ"`, `"NZC"`, `"NZCV"`). It is documentation- and
  disassembler-grade; the assembler ignores it.
- **`undocumented`** marks illegal/undocumented opcodes.

### Operands

An `Operand` says *what kind* of value follows and *how wide* it is.

```rust
pub struct Operand {
    pub kind: OperandKind,
    pub bytes: u8,                  // width; laid out in the set's endianness
}

pub enum OperandKind {
    Immediate,    // a literal value
    Address,      // an absolute address or zero-page offset (told apart by `bytes`)
    RelativePc,   // a signed, PC-relative displacement (branches)
}
```

These three categories are deliberately CPU-agnostic — they describe *bytes on
the wire*, nothing more. The *flavour* of an addressing mode (zero-page versus
absolute, which index register) is carried by the `mode` label, not by
`OperandKind`. So a 6502 zero-page operand and an absolute operand are both
`Address`, distinguished only by `bytes` being 1 or 2.

### Cycles

```rust
pub struct Cycles {
    pub base: u8,
    pub page_cross: u8,    // +cycles when an indexed access crosses a page
    pub branch_taken: u8,  // +cycles when a branch is taken
}
```

Three constructors cover the common cases:

| Constructor | Meaning |
|-------------|---------|
| `Cycles::fixed(n)` | `n` cycles, no conditional extras |
| `Cycles::page_crossing(n)` | `n`, plus one if an indexed access crosses a page |
| `Cycles::branch(n)` | `n` if not taken, `+1` if taken (a further page-cross cycle is also possible) |

## The mode-label contract

The `mode` string is the join between two halves of the system:

1. The **spec** labels each form with a mode (`"immediate"`, `"absolute,x"`,
   `"(indirect),y"`, `"relative"`, …).
2. The **dialect parser** reads source operand syntax, decides which mode it is,
   and looks the form up by that exact string.

So the label strings are an API. The assembler does:

```rust
let insn = set.instruction("LDA")?;     // by mnemonic
let form = insn.form("absolute,x")?;    // by mode label
// emit form.opcode, then operand bytes per form.operands
```

The full set of 6502 mode labels is listed in the
[6502 dialect reference](dialects/6502.md#addressing-modes).

## A worked example

`LDA` (load accumulator) on the 6502 has eight forms. Three of them:

```rust
Instruction {
    mnemonic: "LDA",
    summary: "Load accumulator",
    forms: &[
        // #$nn  — one immediate byte, 2 cycles
        Form { opcode: &[0xA9], mode: "immediate",
               operands: &[Operand { kind: Immediate, bytes: 1 }],
               cycles: Cycles::fixed(2), flags: "NZ", undocumented: false },
        // $nn   — one zero-page address byte, 3 cycles
        Form { opcode: &[0xA5], mode: "zeropage",
               operands: &[Operand { kind: Address, bytes: 1 }],
               cycles: Cycles::fixed(3), flags: "NZ", undocumented: false },
        // $nnnn,X — two address bytes, 4 cycles + page-cross penalty
        Form { opcode: &[0xBD], mode: "absolute,x",
               operands: &[Operand { kind: Address, bytes: 2 }],
               cycles: Cycles::page_crossing(4), flags: "NZ", undocumented: false },
    ],
}
```

Note that **store** instructions encode the indexed page-cross differently from
loads: `STA absolute,x` is `Cycles::fixed(5)` — a write always pays the indexing
cycle, so there is no conditional penalty. The spec captures that distinction.

## Querying the spec

```rust
InstructionSet::instruction(&self, mnemonic: &str) -> Option<&Instruction>
Instruction::form(&self, mode: &str) -> Option<&Form>
Form::len(&self) -> usize    // opcode bytes + operand bytes
```

## Extending to other CPUs

The current types model **fixed-opcode-byte** CPUs (the 6502, and the Z80
including its prefixes). A 68000-class CPU encodes operands as *fields packed
into the opcode word*, which the fixed-`opcode` model doesn't capture; when that
backend is built, `Form` gains a pattern/mask variant. It is deliberately not
modelled yet — solve the CPU in front of you.

## Self-checking

The spec carries its own consistency tests, so a typo in the data fails the
build rather than producing wrong bytes silently. For the 6502:

- every opcode is unique,
- every form's total length is 1–3 bytes,
- spot-checks on known encodings (`LDA #` is `0xA9`, `JMP ()` is `0x6C`, `STA
  absolute,x` is `0x9D` at 5 fixed cycles, `BNE` is a 2-byte relative).

Round-tripping (assemble → disassemble → compare) and checking against Emu198x's
decoders and the Tom Harte test vectors are the next layers of validation.

[`InstructionSet`]: https://github.com/asm198x/asm198x/blob/main/crates/isa/src/lib.rs
