# AArch64 Field Reference

## 1. Registers

| Register | Also called | Job |
|---|---|---|
| x0 – x7 | — | first 8 function arguments |
| x0 | — | return value |
| x8 – x28 | — | scratch, no fixed job |
| x29 | fp | frame pointer |
| x30 | lr | link register (return address) |
| sp | — | stack pointer |
| pc | — | program counter |
| xzr / wzr | zero register | always reads as 0; writes discarded |
| w0 – w30 | — | low 32 bits of the matching x-register |

Note: writing to a `w`-register zero-extends into the corresponding `x`-register.

## 2. Instruction operand order

```
mnemonic  destination,  source,  amount
sub       sp,           sp,      0x20     // sp = sp - 0x20
add       x29,          sp,      0x10     // x29 = sp + 0x10
```

## 3. Condition flags & branch codes

`cmp a, b` computes `a - b`, discards the result, sets flags:

| Flag | Set when |
|---|---|
| Z (zero) | result == 0 (a == b) |
| N (negative) | result's top bit set |
| C (carry) | unsigned add overflowed / unsigned subtract didn't borrow |
| V (overflow) | signed add/subtract overflowed |

Signed and unsigned comparisons use different condition codes:

| Code | Branches if | Signed? |
|---|---|---|
| eq | equal (a == b) | either |
| ne | not equal (a != b) | either |
| lt | less than | signed |
| le | less than or equal | signed |
| gt | greater than | signed |
| ge | greater than or equal | signed |
| lo / cc | lower (less than) | unsigned |
| hs / cs | higher-or-same (>=) | unsigned |
| hi | higher (greater than) | unsigned |
| ls | lower-or-same (<=) | unsigned |

## 4. Instructions by category

### Arithmetic

| Instr | Meaning | Example |
|---|---|---|
| sub | dst = src - amt | `sub sp,sp,0x20` → sp 1000→968 |
| add | dst = src + amt | `add x29,sp,0x10` → sp=968 → x29=984 |
| mul | dst = src1 x src2 | `mul x0,x1,x2` → x1=6,x2=7 → x0=42 |
| neg | dst = -src | `neg x0,x1` → x1=5 → x0=-5 |
| madd | dst = (a x b) + c | array indexing: address = base + index x size |

### Bitwise / logic

| Instr | Meaning | Example |
|---|---|---|
| and | dst = src1 & src2 | 1100 & 1010 = 1000 |
| orr | dst = src1 \| src2 | 1100 \| 1010 = 1110 |
| eor | dst = src1 ^ src2 | `eor x0,x0,x0` → zeroes x0 |
| mvn | dst = ~src | `mvn x0,x1` |
| lsl | dst = src << amt | `lsl x0,x1,2` → x1=5 → x0=20 |
| lsr | dst = src >> amt, unsigned | `lsr x0,x1,1` → x1=10 → x0=5 |
| asr | dst = src >> amt, sign-preserving | — |

### Move

| Instr | Meaning | Example |
|---|---|---|
| mov | copy a value or literal into a register | `mov w8,0` → w8=0 |
| movz / movk | build a large constant across several instructions, 16 bits at a time | used when a value doesn't fit adrp+add |

### Memory — single value

| Instr | Meaning | Example |
|---|---|---|
| str | store one register to memory (positive/scaled offset only) | `str w8,[sp,8]` → writes w8 to sp+8 |
| ldr | load one value from memory | `ldr w0,[sp,8]` → reads sp+8 into w0 |
| stur | str, allows negative/unaligned offsets | `stur wzr,[x29,-4]` |
| ldur | ldr, allows negative/unaligned offsets | `ldur w8,[x29,-0xc]` |
| ldrb / strb | byte-sized variants | — |

### Memory — pair

| Instr | Meaning | Example |
|---|---|---|
| stp | store two registers — reg1→[addr], reg2→[addr+8] | `stp x29,x30,[sp,0x10]` → sp=968: x29→984, x30→992 |
| ldp | load two registers, same layout in reverse | `ldp x29,x30,[sp,0x10]` |

### Addressing

| Instr | Meaning | Example |
|---|---|---|
| adrp | reg = address of the 4KB page containing a target | `adrp x0,0x100000000` |
| add (after adrp) | reg += offset within that page | `add x0,x0,0x4a0` |
| adr | single-instruction adrp+add, range ±1MB | — |

### Control flow

| Instr | Meaning | Example |
|---|---|---|
| bl | call — jumps, writes return address into x30 | `bl sym.imp.printf` |
| ret | jumps to address in x30 | `ret` |
| cmp a, b | compute a - b, keep only the flags | see §3 |
| b.\<cond\> | branch if flags match condition | see §3 |
| cbz reg, L | branch to L if reg == 0 | `cbnz x0,.skip` |
| cbnz reg, L | branch to L if reg != 0 | — |
| b L | unconditional jump | — |

### Misc

| Instr | Meaning | Example |
|---|---|---|
| nop | no-op, padding/alignment | — |
| svc #0 | supervisor call (syscall) | — |

## 5. Frame pointer chain

```
+-------------+  <- sp
| caller's    |
| frame       |
+-------------+
| [x29] -> saved caller's x29 + x30
|  this frame's locals
+-------------+
```

Each frame's `x29` points to a saved copy of the caller's `x29`/`x30`, forming a linked list used for stack unwinding.

## 6. Worked example — main() calling printf

```
sub  sp, sp, 0x20              // allocate 32 bytes
stp  x29, x30, [sp, 0x10]      // save caller's fp + return addr
add  x29, sp, 0x10             // set this frame's fp
mov  w8, 0                     // w8 = 0
str  w8, [sp, 0x8]             // stage return value (0)
stur wzr, [x29, -0x4]          // zero-init unused local (argc slot)
adrp x0, 0x100000000           // x0 = page containing string
add  x0, x0, 0x4a0             // x0 = exact address of "Hello, World!\n"
bl   sym.imp.printf            // call printf(x0)
ldr  w0, [sp, 0x8]             // reload return value into w0
ldp  x29, x30, [sp, 0x10]      // restore caller's fp + return addr
add  sp, sp, 0x20              // deallocate stack space
ret                            // jump to address in x30
```

## 7. Worked example — add(arg1, arg2)

```
sub  sp, sp, 0x10              // allocate 16 bytes
str  w0, [sp, 0xc]             // arg1 -> stack
str  w1, [sp, 0x8]             // arg2 -> stack
ldr  w8, [sp, 0xc]             // arg1 -> w8
ldr  w9, [sp, 0x8]             // arg2 -> w9
add  w0, w8, w9                // w0 = arg1 + arg2 (return value)
add  sp, sp, 0x10              // deallocate stack space
ret                            // jump to address in x30
```

Leaf function: no `bl`, so x29/x30 are not saved.

## 8. radare2 quick commands

| Command | What it does |
|---|---|
| aaa | analyze everything |
| afl | list discovered functions |
| pdf @ main | disassemble the function at main |
| pd 10 | disassemble next 10 instructions |
| iz | list strings + addresses |
| v / V | visual mode / graph view |
| : | run one command without leaving visual mode |
| x | show cross-references to current address |

### Patch → verify → run

```
r2 -w -c 'wx <hexbytes> @ <addr>' binary    # patch raw bytes
codesign --force -s - binary                # required after any edit, on Apple Silicon
rasm2 -d -a arm -b 64 <hexbytes>             # decode raw bytes independently
```
