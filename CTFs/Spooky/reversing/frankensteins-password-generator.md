# Frankenstein's Password Generator

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Category**   | Reversing |
| **Points**     | 75        |
| **Difficulty** | Easy      |

## Statement

> Dr. Frankenstein left behind a strange password generator in an abandoned NJIT laboratory. The machine refuses to start without an activation phrase. His last notebook entry reads, "A bolt at midnight gave it life." Can you figure out what the program expects and wake the creature?

A binary (`frankenstein`) asks for a password and won't start without the right one.

## Overview

The binary doesn't store the password in cleartext — it keeps it encrypted in memory as `*$*)C7EP\]erto`. For each character of the input it applies:

```
result = (input[i] XOR 0x5d) + i*7 + 11
```

and compares the result byte by byte against the encrypted bytes. If every byte matches, the password is accepted.

Since this is a simple per-character transform with no dependency between characters, it's directly invertible: each output byte only depends on the input byte at the same index and its position `i`.

## Solution

### Invert the formula

Given `result[i] = (input[i] XOR 0x5d) + i*7 + 11`, solving for `input[i]`:

```
input[i] = (result[i] - i*7 - 11) XOR 0x5d
```

### Apply it to the encrypted secret

```python
secret = b'*$*)C7EP\\]erto'
password = bytes([(secret[i] - i*7 - 0xb) ^ 0x5d for i in range(14)])
print(password.decode())  # → BOLTATMIDNIGHT
```

The notebook quote was the hint all along: "A bolt at midnight gave it life" → `BOLTATMIDNIGHT`.

### Run it

```bash
echo "BOLTATMIDNIGHT" | ./frankenstein
# The laboratory lights flicker. IT'S ALIVE!
# The creature whispers: <flag>
```

## Flag

```
spooky{its_alive_at_midnight}
```

## What I learned

- A per-character transform of the form `f(input[i], i)` with no cross-character dependency (no running state, no feedback from previous bytes) is always directly invertible — just solve the equation for `input[i]`.
- When a challenge gives you both an encrypted constant and the exact transform (XOR + arithmetic), there's no need to brute-force: write the inverse function and apply it once.
- A narrative hint buried in flavor text ("A bolt at midnight gave it life") is often the literal password in disguise — worth testing obvious readings of the quote before assuming it's purely thematic.