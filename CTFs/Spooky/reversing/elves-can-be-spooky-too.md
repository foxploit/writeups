# Elves can be Spooky Too

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Category**   | Reversing |
| **Points**     | 377       |
| **Difficulty** | Hard      |

## Statement

> The elves left you a very small doorway. You are given a host, a port, and a byte budget. Send an x86-64 Linux ELF that: is at most 112 bytes (the service prints the live limit on connect), writes its own exact bytes to stdout, exits with status 0.

```
nc 3.129.162.36 31903
```

This is a pure "ELF golfing" / binrev challenge. I leaned on an AI (Claude) for the byte-layout design, and on this article to understand the base technique: [Handcrafting x64 ELF: From Specification to Bytes](https://medium.com/@dassomnath/handcrafting-x64-elf-from-specification-to-bytes-9986b342eb89).

## Overview

A standard `Ehdr` (64 bytes) plus one `Phdr` (56 bytes) already add up to 120 bytes of headers alone, before a single instruction. The limit is 112 bytes. The two structures have to overlap.

**The trick:** on `execve`, the kernel only validates a handful of fields — the magic number, class, endianness, `e_type`, `e_machine`, `e_phentsize` (must be 56), `e_phnum`, `e_phoff`, and, inside the `PT_LOAD` segment, `p_type`, `p_flags`, `p_offset`, `p_vaddr` and `p_filesz`. Everything else — `e_shoff`, `e_flags`, `e_ehsize`, `e_shnum`, `e_shstrndx`, the padding bytes of `e_ident`, `p_paddr`, even `p_align` — is ignored. Those ignored fields are exactly where real shellcode can be hidden instead of a throwaway value.

## Solution

### The layout plan

1. Set `e_phoff = 56`, so the `Phdr` overlaps the tail of the `Ehdr` — the whole file fits in exactly 112 bytes.
2. Hide shellcode in three unused gaps: the `e_ident` padding (9 bytes), the `e_shoff` / `e_flags` / `e_ehsize` trio (14 bytes), and `p_paddr` (8 bytes).
3. Use short `jmp`s to skip over the fields the kernel _does_ validate (`p_offset = 0`, `p_vaddr`, `p_filesz = 112`, ...) so the CPU never tries to execute them as instructions.
4. Point `e_entry` at the first free gap.

### The shellcode, split across the three gaps

```asm
push 1; pop rdi          ; rdi = 1 (stdout)
push 1; pop rax          ; rax = 1 (SYS_write)
jmp   +25                ; skip over e_type/e_machine/e_version/e_entry/e_phoff
; -- next gap --
mov esi, 0x400000        ; rsi = base address where the file itself is mapped
push 0x70; pop rdx       ; rdx = 112
syscall                  ; write(1, base, 112)  -> prints itself
xor edi, edi             ; rdi = 0
jmp   +26                ; skip over p_offset/p_vaddr/p_filesz/p_memsz/p_align
; -- next gap --
push 0x3C; pop rax        ; rax = 60 (SYS_exit)
syscall                   ; exit(0)
```

This is essentially a quine: the binary, when run, writes out its own exact bytes.

### Final payload (112 bytes, hex)

```
7f454c46020101006a015f6a0158eb190002003e000100000007004000000000003800000000000000be000040006a705a0f0531ffeb1a38000100000005000000000000000000000000004000000000006a3c580f05000000700000000000000070000000000000000000000000000000
```

I tested it locally (an x86-64 VM / sandbox) before throwing it at the server: it runs, exits with status 0, and `cmp` / `sha256sum` confirm its `stdout` matches the file itself byte for byte.

### Script to talk to the service

The service's protocol is: banner → `SIZE 112\n` → the 112 raw bytes → read back `PASS` / `FLAG` / `FAIL`.

```python
import socket, struct

HOST, PORT = "3.129.162.36", 31903

def build_elf():
    b = bytearray(112)
    def put(o, d): b[o:o+len(d)] = d
    put(0, bytes([0x7F,0x45,0x4C,0x46])); b[4]=2; b[5]=1; b[6]=1
    put(7, bytes([0x6A,0x01,0x5F,0x6A,0x01,0x58,0xEB,0x19,0x00]))
    BASE = 0x400000
    put(16, struct.pack('<H',2)); put(18, struct.pack('<H',0x3E))
    put(20, struct.pack('<I',1)); put(24, struct.pack('<Q',BASE+7))
    put(32, struct.pack('<Q',56))
    put(40, bytes([0xBE,0x00,0x00,0x40,0x00,0x6A,0x70,0x5A,0x0F,0x05,0x31,0xFF,0xEB,0x1A]))
    put(54, struct.pack('<H',56)); put(56, struct.pack('<H',1))
    put(58, struct.pack('<H',0)); put(60, struct.pack('<H',5)); put(62, struct.pack('<H',0))
    put(64, struct.pack('<Q',0)); put(72, struct.pack('<Q',BASE))
    put(80, bytes([0x6A,0x3C,0x58,0x0F,0x05,0x00,0x00,0x00]))
    put(88, struct.pack('<Q',112)); put(96, struct.pack('<Q',112)); put(104, struct.pack('<Q',0))
    return bytes(b)

payload = build_elf()
with socket.create_connection((HOST, PORT), timeout=15) as s:
    print(s.recv(4096).decode(errors="replace"))
    s.sendall(f"SIZE {len(payload)}\n".encode())
    s.sendall(payload)
    print(s.recv(4096).decode(errors="replace"))
```

## Flag

```
spooky{phdr_in_ehdr_c2d021cc4fc5}
```

## What I learned

- The Linux kernel validates only a small, specific subset of an ELF header and program header on `execve` — everything outside that subset (padding bytes, unused fields like `p_paddr`, `e_shoff`, `e_flags`) is free real estate for packing extra code.
- Overlapping the `Phdr` with the tail of the `Ehdr` (by setting `e_phoff` to point inside the `Ehdr` itself) is the single biggest space saver in ELF golfing — it eliminates 56 bytes of otherwise-mandatory structure.
- Short relative `jmp`s are essential whenever shellcode is interleaved with header fields the kernel does check: the CPU would otherwise try to execute those validated values (offsets, sizes, flags) as instructions.
- A "print my own bytes" quine becomes trivial once you know where the file is mapped in memory (`e_entry`'s base, here `0x400000` for a non-PIE executable) — `write(1, base, filesize)` is the whole program.
- Testing the crafted binary locally first (checking exit code and diffing `stdout` against the file on disk) catches layout mistakes before spending a network round-trip against the challenge server.