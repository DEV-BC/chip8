# CHIP-8 Emulator: Design

A CHIP-8 emulator with a live terminal debugger, written in Rust. This is a
living document: when the implementation teaches us something new, we update it.

## Goals and non-goals

| Goals | Non-goals (for now) |
|---|---|
| All 35 classic instructions | SUPER-CHIP and XO-CHIP extensions |
| Plays classic games in a terminal | A graphical window |
| Live debugger: registers, stack, memory, current instruction | Real audio (a visual indicator instead) |
| Pause, single-step and resume | JIT compilation |
| Passes a community test ROM suite | Save states (a stretch goal) |

## Architecture

The emulator knows nothing about terminals.

```
chip8/                     Cargo workspace
├── crates/
│   ├── chip8-core/        library: pure logic, no I/O, no clock
│   │   └── src/
│   │       ├── lib.rs           public API
│   │       ├── types.rs         Address, Reg, Key
│   │       ├── error.rs         Chip8Error
│   │       ├── memory.rs        RAM + font
│   │       ├── stack.rs         call stack
│   │       ├── display.rs       64×32 framebuffer
│   │       ├── instruction.rs   decode + disassemble
│   │       └── cpu.rs           Chip8, step(), execute
│   └── chip8-tui/         binary: terminal frontend
│       └── src/
│           ├── main.rs          CLI args, setup (thin)
│           ├── app.rs           event loop, pause/step
│           ├── input.rs         keyboard → keypad
│           └── ui.rs            ratatui drawing
└── docs/DESIGN.md
```

```
              keys, "step", "tick"
 ┌───────────┐ ─────────────────▶ ┌────────────┐
 │ chip8-tui │                    │ chip8-core │
 │ (terminal)│ ◀───────────────── │ (computer) │
 └───────────┘   read-only views  └────────────┘
```

This split gives us:

- **Easy testing:** core tests run without a terminal.
- **No time dependency:** the frontend decides when the clock ticks, so tests can
  run thousands of cycles instantly.
- **Reuse:** a different frontend could use the same core without changing it.

## Memory layouts

The CHIP-8 machine's 4 KB of RAM:

```
0x000 ┌──────────────────────────┐
      │ reserved (interpreter)   │
0x050 │ built-in font (80 bytes) │  sprites for digits 0–F
0x0A0 │ unused                   │
0x200 ├──────────────────────────┤ ◀── PC starts here
      │ ROM, loaded here         │
      │ (3,584 bytes maximum)    │
0xFFF └──────────────────────────┘
```

The `Chip8` struct in the host computer's memory:

```
STACK (main's frame)                     HEAP
┌──────────────────────────────┐
│ machine: Chip8 (6,232 bytes) │         (nothing)
│ ┌──────────────────────────┐ │
│ │ memory    4,096 B        │ │         Every field has a fixed size,
│ │ display   2,048 B        │ │         so it all sits inline with
│ │ stack        34 B        │ │         no pointers to chase.
│ │ v[0..16]     16 B        │ │
│ │ keys[0..16]  16 B        │ │
│ │ pc, i, timers, rng, etc. │ │
│ └──────────────────────────┘ │
└──────────────────────────────┘
```

Sizes were measured with `std::mem::size_of` on a prototype of these types, on
x86-64 with Rust 1.99.1. Moving a `Chip8` copies about 6 KB, so we create it once
and pass references.

## Core data structures

**Newtypes make bad values impossible.** A CHIP-8 address has 12 bits, and the
constructor enforces that:

```rust
/// A 12-bit CHIP-8 memory address (0x000..=0xFFF).
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Address(u16);          // size: 2 bytes, same as a bare u16

impl Address {
    pub const fn new(raw: u16) -> Self {
        Self(raw & 0x0FFF)        // keeps the low 12 bits: 0x1ABC → 0xABC
    }
}

/// One of the 16 registers, V0..=VF. Always a valid index.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Reg(u8);
```

The inner field is private, so every `Address` passes through the mask. Memory
reads therefore can't go out of bounds and don't need to return an error.

**Instructions are an enum, with decoding kept separate from execution:**

```rust
pub enum Instruction {                   // size: 4 bytes
    ClearScreen,                         // 00E0
    Return,                              // 00EE
    Jump(Address),                       // 1NNN
    Call(Address),                       // 2NNN
    SkipIfEqByte { x: Reg, byte: u8 },   // 3XNN
    AddReg { x: Reg, y: Reg },           // 8XY4
    Draw { x: Reg, y: Reg, height: u8 }, // DXYN
    WaitKey { x: Reg },                  // FX0A
    // ...and the rest, one variant each
}
```

```
 fetch               decode                  execute
 bytes 0xD1 0x25 ──▶ Draw{x:V1,y:V2,h:5} ──▶ changes the machine
                            │
                            └──▶ debugger displays "DRW V1, V2, 5"
```

Decoding is a pure function, so the debugger can disassemble memory without
running anything, using the same code as the CPU. The compiler also rejects a
`match` that misses a variant, so an instruction can't be forgotten.

**Machine state is an enum too.** `FX0A` means "wait for a key", but blocking
inside that instruction would freeze the debugger as well. So the machine records
that it's waiting:

```rust
pub enum ExecState {
    Running,
    WaitingForKey { into: Reg },
}
```

## Public API

```rust
impl Chip8 {
    pub fn new(config: Config) -> Self;
    pub fn load_rom(&mut self, rom: &[u8]) -> Result<(), Chip8Error>;

    // Driving the machine: the frontend calls these
    pub fn step(&mut self) -> Result<(), Chip8Error>;  // one fetch-decode-execute
    pub fn tick_timers(&mut self);                     // called at 60 Hz
    pub fn set_key(&mut self, key: Key, pressed: bool);

    // Read-only views: `&self` means the debugger can't change anything
    pub fn pc(&self) -> Address;
    pub fn registers(&self) -> &[u8; 16];
    pub fn stack(&self) -> &[Address];
    pub fn memory(&self) -> &[u8];
    pub fn display(&self) -> &Display;
    pub fn sound_active(&self) -> bool;
}
```

`Config` holds the random seed and a `Quirks` struct. Quirks exist because CHIP-8
interpreters disagree on small behaviours, such as whether a shift reads `VY`. A
fixed seed also makes the `CXNN` random instruction repeatable in tests.

## Error handling

```rust
#[derive(Debug, thiserror::Error, PartialEq, Eq)]
pub enum Chip8Error {
    #[error("ROM is {size} bytes, but only {max} fit")]
    RomTooLarge { size: usize, max: usize },
    #[error("unknown opcode {opcode:#06X} at {at}")]
    UnknownOpcode { opcode: u16, at: Address },
    #[error("stack overflow: more than 16 nested calls")]
    StackOverflow,
    #[error("return with an empty stack")]
    StackUnderflow,
}
```

- **Core library:** uses `thiserror`, so callers can `match` on the exact error.
- **TUI binary:** uses `anyhow` and adds context, such as the ROM's file name.
- **Bad ROMs:** a broken ROM must never crash the debugger. It pauses and shows
  the error.

## Dependencies

| Crate | Purpose |
|---|---|
| `thiserror` | Core error types |
| `fastrand` | Seedable random numbers |
| `ratatui` 0.30 | Terminal UI (includes `crossterm`) |
| `clap`, `anyhow` | CLI arguments, application errors |
| `tracing` | Logging to a file, since `println!` would corrupt a full-screen TUI |

## Known risks

1. **Key releases:** games need to know when a key is held, but many terminals
   only report presses and repeats. Use release events where the terminal supports
   them. Otherwise, treat a key as held for about 100 ms after its last event.
2. **Screen size:** half-block characters (`▀`) fit two pixels in one character
   cell, so the display needs 64×16 cells. With the debugger panes, that's about
   105×24 in total.
3. **Timing:** the frontend runs about 11 instructions per 60 Hz frame, or roughly
   660 per second. This is configurable.

## Milestones

| # | Milestone | Main concept |
|---|---|---|
| 0 | Workspace, CI, first commit | Cargo, git, GitHub Actions |
| 1 | `Address`, `Memory`, font, ROM loading | Hex and binary, memory as a byte array |
| 2 | Fetch and decode | Bitwise operations, enums, `match` |
| 3 | Six instructions and the display: IBM logo appears | The fetch-decode-execute cycle |
| 4 | Call and return | The call stack |
| 5 | Arithmetic and skips, passing test ROMs | Flags, overflow, integration tests |
| 6 | Timers, keypad, random | State machines, seeded tests |
| 7 | TUI screen and event loop | ratatui, RAII terminal cleanup |
| 8 | Debugger panes, pause and step | Borrowing, layouts |
| 9 | Keyboard input: playing Pong | Terminal input |
| 10 | Quirks, README, GIF, release, blog post | Finishing like a professional |
