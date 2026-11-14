# Real-mode memory model

The port preserves the original DOS program's memory layout. Instead of
translating every game global into a new C variable, it stores the executable,
game state, tables, buffers, and VGA page in a byte array and accesses them
through segment:offset addresses.

## Address translation

`src/mem.h` defines `mem[MEM_SIZE]` with `MEM_SIZE = 0x110000`. The real-mode
address calculation is:

```text
linear_address = (segment << 4) + offset
```

The executable image is placed at `LOAD_SEG = 0x1000`. An address written
`SSSS:OOOO` in the original executable's file/disassembly convention uses
`SEG(SSSS)`, which adds the load segment to that file segment. The original
Microsoft C 5.1 medium-model data group is `DGROUP = SEG(0x1BE4)`, or segment
`0x2BE4`; `DS:offset` references resolve relative to it.

The image is expected to be `0x28A90` bytes and must end before the DOS heap.
`HEAP_BOTTOM` is immediately above the 64 KB DGROUP area; DOS allocation
replacements carve blocks from there to `HEAP_TOP = 0xA000`. This lets original
far pointers refer to allocated blocks inside `mem[]`.

Unlike ports that keep video memory outside the emulated address space, this
TD3 port stores VGA mode 13h page 0 at `A000:0000` inside `mem[]`; the screen is
320 × 200 bytes. The game also uses a RAM page. `platform/vga.c` models the
palette DAC and CRTC start offset, and composes the displayed image from the
memory-backed VGA page.

## Reading and writing state

`symbols.h` provides names for many original data-segment and code-segment
offsets. The macros in `mem.h` turn addresses into typed lvalues:

```c
DSB(DS_race_state) = 3;     /* unsigned byte in DGROUP */
DSW(DS_game_state) = 4;     /* unsigned word in DGROUP */
DSS(DS_some_signed) = -2;   /* signed word in DGROUP */
SEGW(0x0E12, 0x0FF2) = x;   /* word at a file code-segment address */
```

`DSB`/`DSC` read or write unsigned/signed bytes; `DSW`/`DSS` are unsigned/signed
words; `DSL`/`DSSL` are unsigned/signed 32-bit values. `SEGB`/`SEGC`/`SEGW`/
`SEGS` address other original segments by file-segment number. For unnamed
state, source uses a raw offset and a brief field comment.

`rd8`/`rd16`/`rd32` and `wr8`/`wr16`/`wr32` explicitly assemble or split
little-endian values. Typed accessor macros expose lvalues into `mem[]`, so
the build disables strict aliasing. They do not bounds-check: accesses must
remain inside `mem[]`, even when reproducing a documented original quirk.

The porting rule in `ENGINE.md` is to keep persistent game state in the
original memory locations, not shadow it in long-lived C globals. This
preserves aliasing and makes behavior across ported routines match the original.

## Far pointers and code pointers

`FarPtr` stores a 16:16 pointer as `{off, seg}`. Helpers such as `far_rd()`,
`far_wr()`, `ds_far()`, `far_mp()`, and `far_make()` handle stored far pointers.
The original program stores pointers to resources, game pages, and data in
this form.

Some data tables hold original near or far **code pointers** (for example
keyboard or sound routines). They remain represented in `mem[]`; `codeptr.h`
and `codeptr.c` map those original pointer values to C functions. They are
not converted into arbitrary native pointers.

## Fixed-width arithmetic and faults

The game depends on 8-, 16-, and 32-bit widths, signedness, wraparound,
truncation, and carry/borrow behavior. Ported routines use explicit fixed-width
types and casts at points where the original CPU or compiler truncated a
result. Divide operations use the helpers in `mem.h`, which call `div_error()`
on zero divisors or quotient overflow to reproduce the Microsoft C runtime's
R6003 failure, except at explicitly documented clamp sites.

## Practical rules

- Resolve a `DS_*` symbol in `src/symbols.h` and select the accessor matching
  its size and signedness.
- Use `SEG*` accessors for segment-addressed data; use `FarPtr` for stored
  segment:offset pointers.
- Do not model a persistent original global with a separate C global.
- Preserve specified fixed-width arithmetic and quirks; never read or write
  outside `mem[]`.
- Pump `host_pump()` in busy waits so timer/input events progress.

`ENGINE.md` is the authoritative contributor guide, including detailed
constraints and examples.
