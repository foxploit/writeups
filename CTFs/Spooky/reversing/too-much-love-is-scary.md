# Philophobia

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Category**   | Reversing |
| **Points**     | 50        |
| **Difficulty** | -         |


## Statement

> too much love is scary

A static x86-64 ELF (`philophobia`) reads two integers and a line of text, checks the line against a hardcoded quote at a position given by the first integer, and — if it matches — XOR-decrypts and prints a hidden flag using a key derived from both integers.

## Overview

The binary was analyzed purely from `objdump` output — no debugger needed, since it's a small static binary with no anti-debug tricks.

### `main` — reading the input

```asm
lea    rcx, [rip+...]            ; "%d %d" format string
lea    rdx, [rbp-0x414]          ; &num2
lea    rax, [rbp-0x418]          ; &num1
call   __isoc23_scanf@plt        ; scanf("%d %d", &num1, &num2)
...
call   getchar@plt               ; consume the leftover '\n'
...
call   fgets@plt                 ; fgets(buf, 1000, stdin)  -> buf at rbp-0x3f0
```

`main` first prompts for two integers (`num1`, `num2`), then consumes the trailing newline with `getchar()`, then reads a whole line of up to 999 bytes into a buffer on the stack (`buf`, at `rbp-0x3f0`).

### The hardcoded quote

Right after the `fgets`, the binary builds a 20-byte string on the stack out of raw immediates:

```asm
movabs $0x2049206c6c616853, rax   ; "Shall I "
movabs $0x20657261706d6f63, rdx   ; "compare "
mov    [rbp-0x410], rax
mov    [rbp-0x408], rdx
movl   $0x65656874, [rbp-0x400]   ; "thee"
```

Read little-endian, those bytes spell **`Shall I compare thee`** (20 characters — the opening of Shakespeare's Sonnet 18).

### The check

```asm
mov    eax, [rbp-0x418]          ; num1
lea    rdx, [rbp-0x3f0]          ; buf
movsxd rax, eax
lea    rcx, [rdx+rax]            ; buf + num1
lea    rax, [rbp-0x410]          ; "Shall I compare thee"
mov    edx, 0x14                 ; length 20
mov    rsi, rax
mov    rdi, rcx
call   strncmp@plt               ; strncmp(buf + num1, "Shall I compare thee", 20)
test   eax, eax
jne    <fail>
```

So `num1` is actually an **offset into the line you typed**: the first 20 bytes starting at `buf[num1]` must be exactly `"Shall I compare thee"`. If `num1 = 0`, the typed line simply has to _start_ with that quote.

### `print_flag` — the XOR key

On success, `main` calls `print_flag(num1, num2)` (as `edi`, `esi`):

```asm
mov    eax, [rbp-0x48]           ; num2 (esi)
sub    eax, [rbp-0x44]           ; - num1 (edi)
add    eax, 0x47                 ; + 71
mov    [rbp-0x34], eax           ; key = num2 - num1 + 71
```

So the single-byte XOR key used to decrypt the flag is:

```
key = num2 - num1 + 71
```

### The encrypted flag bytes

`print_flag` also writes a 36-byte blob onto the stack from raw `movabs`/`movl` immediates (`rbp-0x30` through `rbp-0x10`). Read little-endian, in order, the raw bytes are:

```
29 2a 35 35 31 23 21 37 6e 23 38 69 05 6b 05 34
69 69 3e 05 2e 6a 05 6b 6a 2c 69 05 37 23 29 69
6b 3c 27 00
```

The last byte is `0x00` — a plain terminator. The decryption loop XORs each byte with `key` and `putchar`s it, stopping as soon as it hits a **raw** byte of `0x00` (the loop checks the un-XORed byte before printing, so the terminator itself is never printed):

```asm
; per iteration, i = 0..34
al = buf[-0x30 + i]        ; raw encrypted byte
al ^= key                  ; XOR with the derived key
putchar(al)
i++
; loop while buf[-0x30 + i] != 0 (raw check, before XOR)
```

## Solution

### Pick `num1` and `num2`

`num1` just needs to be `0` (the quote should sit at the very start of the typed line). For the key, any `num1`/`num2` pair that makes `num2 - num1 + 71` decrypt the first byte (`0x29`) to a printable, sensible character works. Trying `num1 = 0` and checking what key turns `0x29` into `'s'` (the expected start of `spooky{...}`):

```
0x29 XOR key == 's' (0x73)
key == 0x29 XOR 0x73 == 0x5A == 90
num2 - 0 + 71 == 90  →  num2 == 19
```

So `num1 = 0`, `num2 = 19` gives `key = 90`.

### Build the input

```
0 19
Shall I compare thee to a summer's day?
```

The first line satisfies `scanf("%d %d", ...)`. The second line is read by `fgets` and starts with exactly the required 20-byte quote (the rest of the Shakespeare line is just flavor — only the first 20 bytes are checked).

### Decrypt manually (to double check before connecting)

```python
enc = bytes.fromhex(
    "292a353531232137"
    "6e233869056b0534"
    "69693e052e6a056b"
    "6a2c690537232969"
    "6b3c2700"
)
key = 90  # num2 - num1 + 71 = 19 - 0 + 71
flag = bytes(b ^ key for b in enc if b != 0)
print(flag.decode())
```

### Send it to the service

```python
import socket

HOST, PORT = "<host>", "<port>"
payload = b"0 19\nShall I compare thee to a summer's day?\n"

with socket.create_connection((HOST, PORT), timeout=10) as s:
    s.sendall(payload)
    print(s.recv(4096).decode(errors="replace"))
```

## Flag

```
spooky{m4yb3_1_n33d_t0_10v3_mys31f}
```

## What I learned

- A static binary with no stripped symbols (`setup`, `print_flag`, `main` all named) is a strong hint to just read the disassembly top to bottom rather than reach for a debugger — control flow and stack layout are already clear from the `objdump` listing.
- `movabs`/`movl` immediates writing fixed values onto the stack are a common way compilers (and CTF authors) embed short strings or byte blobs directly in code instead of the `.rodata` section — reading them just means decoding each immediate little-endian, in write order.
- A single-byte XOR "encryption" is only as strong as its key, and here the key was derived from two attacker-controlled integers via simple arithmetic (`num2 - num1 + 71`) — once the first plaintext byte is guessed (`'s'` for `spooky{`), the key falls out directly by XOR-ing it with the first ciphertext byte.
- The `strncmp(buf + num1, ..., 20)` check repurposes one of the "key" integers as a second role — an offset into the input buffer — which is easy to miss if the two integers are assumed to only feed the XOR key; tracing every use of each variable in the disassembly avoids that mistake.
- A decrypt loop that checks the **raw** byte before XOR-ing to decide when to stop (rather than checking the XOR'd result) means the terminator in the ciphertext must itself be `0x00` — worth noticing so the last byte of an encrypted blob isn't mistakenly included as flag data.