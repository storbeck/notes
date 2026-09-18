# x86-64 Field Reference

## 1. Registers

| Register | Also called | Job |
|---|---|---|
| rdi, rsi, rdx, rcx, r8, r9 | — | first 6 function arguments, in order |
| rax | — | return value |
| rbx, r12 – r15 | — | callee-saved (must be preserved across calls) |
| r10, r11 | — | caller-saved scratch |
| rbp | frame/base pointer | anchors stack frame; often omitted at -O2 |
| rsp | — | stack pointer |
| rip | — | instruction pointer |
| eax / ax / al (etc.) | — | 32 / 16 / 8-bit views of rax — same physical register |

Note: no hardware zero register. `xor reg, reg` is the idiom to zero a register instead.

Note: writing a 32-bit form (`eax`) zero-extends into the full 64-bit register, same as AArch64's `w`-registers. Writing a 16-bit (`ax`) or 8-bit (`al`) form does **not** — the upper bits are left untouched.

## 2. Instruction operand order — two competing syntaxes

**Intel syntax** (used by r2 default, NASM, MASM, this document): `mnemonic  dst, src`
```
mov  eax, 5        // eax = 5
add  eax, ebx       // eax = eax + ebx
```

**AT&T syntax** (used by objdump default, gdb default, GCC's `-S` output): `mnemonic  src, dst` — operand order is reversed, registers prefixed `%`, immediates prefixed `$`, memory uses `offset(%base,%index,scale)`.
```
movl $5, %eax        // same instruction as above
addl %ebx, %eax
```

Toggle in r2: `e asm.syntax=att` or `e asm.syntax=intel`.

## 3. Condition flags & branch codes

`cmp a, b` computes `a - b`, discards the result, sets flags:

| Flag | Set when |
|---|---|
| ZF (zero) | result == 0 (a == b) |
| SF (sign) | result's top bit set |
| CF (carry) | unsigned add overflowed / unsigned subtract borrowed |
| OF (overflow) | signed add/subtract overflowed |

| Code | Branches if | Signed? |
|---|---|---|
| je / jz | equal / zero | either |
| jne / jnz | not equal / not zero | either |
| jl | less than | signed |
| jle | less than or equal | signed |
| jg | greater than | signed |
| jge | greater than or equal | signed |
| jb | below (less than) | unsigned |
| jbe | below or equal | unsigned |
| ja | above (greater than) | unsigned |
| jae | above or equal | unsigned |

No equivalent to AArch64's `cbz`/`cbnz` — x86 always requires a separate `cmp`/`test` before a conditional jump.

## 4. Instructions by category

### Arithmetic

| Instr | Meaning | Example |
|---|---|---|
| add | dst += src | `add eax,ebx` |
| sub | dst -= src | `sub eax,ebx` |
| imul | dst *= src (signed) | `imul eax,ebx` |
| neg | dst = -dst | `neg eax` |
| inc / dec | dst += 1 / dst -= 1 | `inc eax` |

### Bitwise / logic

| Instr | Meaning | Example |
|---|---|---|
| and | dst &= src | — |
| or | dst \|= src | — |
| xor | dst ^= src | `xor eax,eax` → zeroes eax |
| not | dst = ~dst | — |
| shl | dst <<= amt | `shl eax,2` → x4 |
| shr | dst >>= amt, unsigned | `shr eax,1` → ÷2 |
| sar | dst >>= amt, sign-preserving | — |

### Move

| Instr | Meaning | Example |
|---|---|---|
| mov | copy a value, literal, or memory into a register (or reverse) | `mov eax,[rbp-4]` |
| movzx | move with zero-extend (small → large, unsigned) | `movzx eax, byte [rbp-1]` |
| movsx | move with sign-extend (small → large, signed) | — |
| lea | dst = computed address (no memory access) | `lea rdi,[rip+0x2000]` |

Note: x86 has no separate load/store mnemonics — `mov` reads or writes memory directly depending on which operand is a memory reference. AArch64 requires a dedicated `ldr`/`str` for any memory access.

### Stack

| Instr | Meaning | Example |
|---|---|---|
| push | rsp -= 8; write reg to [rsp] | `push rbp` |
| pop | read [rsp] into reg; rsp += 8 | `pop rbp` |

### Control flow

| Instr | Meaning | Example |
|---|---|---|
| call | push return address, jump | `call printf` |
| ret | pop return address into rip | `ret` |
| leave | mov rsp,rbp; pop rbp — one-instruction epilogue | `leave` |
| cmp a, b | compute a - b, keep only the flags | see §3 |
| test a, b | compute a & b, keep only the flags | `test eax,eax` → checks if eax == 0 |
| j\<cond\> | jump if flags match condition | see §3 |
| jmp | unconditional jump | — |

Note: `call` pushes the return address directly onto the stack; `ret` pops it back into `rip`. There is no dedicated link register like AArch64's `x30` — the return address lives in memory from the moment `call` executes, never in a register that could be overwritten by a nested call.

## 5. Stack frame setup

```
push rbp              <- save caller's frame pointer (onto the stack, not a register)
mov  rbp, rsp          <- set this frame's base pointer
sub  rsp, N            <- allocate N bytes of locals
...
leave                  <- mov rsp,rbp; pop rbp  (undo the above in one instruction)
ret                    <- pop return address into rip
```

Because the return address is already on the stack (pushed by `call`), a leaf function that calls nothing still typically keeps this `push rbp`/`pop rbp` pattern for `rbp`'s sake alone — unlike AArch64, where a leaf function skips the `x29`/`x30` save entirely.

## 6. Worked example — main() calling printf

```
push rbp                     // save caller's frame pointer
mov  rbp, rsp                 // set this frame's base pointer
sub  rsp, 0x10                // allocate 16 bytes of locals
lea  rdi, [rip+0x2000]        // rdi = address of "Hello, World!\n" (1st arg)
call printf                   // call printf(rdi)
mov  eax, 0                   // return value = 0
leave                         // mov rsp,rbp; pop rbp
ret                           // pop return address into rip
```

## 7. Worked example — add(arg1, arg2)

```
push rbp                     // save caller's frame pointer
mov  rbp, rsp                 // set this frame's base pointer
mov  dword [rbp-0x4], edi     // arg1 (edi) -> stack
mov  dword [rbp-0x8], esi     // arg2 (esi) -> stack
mov  edx, dword [rbp-0x4]     // arg1 -> edx
mov  eax, dword [rbp-0x8]     // arg2 -> eax
add  eax, edx                 // eax = arg1 + arg2 (return value)
pop  rbp                      // restore caller's frame pointer
ret                           // pop return address into rip
```

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
| e asm.syntax=att / intel | switch disassembly syntax |

### Patch → verify → run

```
r2 -w -c 'wx <hexbytes> @ <addr>' binary    # patch raw bytes
codesign --force -s - binary                # required after any edit, on macOS
rasm2 -d -a x86 -b 64 <hexbytes>             # decode raw bytes independently
```
