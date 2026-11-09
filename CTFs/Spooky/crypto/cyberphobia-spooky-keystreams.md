# Spooky Keystreams

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Category**   | Crypto    |
| **Points**     | 50        |
| **Difficulty** | Easy      |

## Statement

> Nothing is secure nowadays, not even the own ciphers my machine generates. Take a look for yourself, I don't know how people on the internet were able to retrieve my encrypted flag with just these files...

## Overview

```
Spooky Keystreams

Read backup.py, then run python solve.py.
Enter hex or filename:line when prompted. Use cat -n data.txt to see line numbers.

Known message: 546865206861756e746564207661756c7420636c6f736573206174206d69646e696768742e2053746179206f7574736964652e20202020202020202020202020
Encrypted message: e4a8bb39921d6f0e3940a040ae22a1afc698fdeb3058ea8e1002fe25142227c6f7f018ddf3eb9ce75e69dbdc59720fcffd98a9c0bb9f03f0d644f34c68a4c306
Encrypted flag: c3b0b176910561033e16f151920887be9298bea77f0bafdd1043aa25596b6388beb75089fdebefb31f30db930c265c86b9dda7c0bb9f03f0d644f34c68a4c306
```

`backup.py` shows how the server encrypts messages:

```python
from Crypto.Cipher import AES

def encrypt(message, key):
    cipher = AES.new(key, AES.MODE_CTR, nonce=b"ghost123", initial_value=0)
    return cipher.encrypt(message.ljust(64, b" "))
```

The cipher is AES in **CTR mode**, with a **fixed, hardcoded nonce** (`b"ghost123"`) and a **fixed initial counter value** (`0`). CTR mode works by generating a keystream from `(key, nonce, counter)` and XOR-ing it with the plaintext: `ciphertext = plaintext XOR keystream`.

The critical property of CTR mode is: if the same `(key, nonce, counter)` combination is ever reused for two different messages, both messages get **the exact same keystream**. And that's exactly what's happening here — every call to `encrypt()` uses the same key, the same nonce, and starts the counter at 0 every time. Both the known message and the flag were encrypted with the identical keystream.

Since `ciphertext = plaintext XOR keystream`, this means:

```
keystream = known_plaintext XOR known_ciphertext
```

and once the keystream is recovered, it can be reused on any other ciphertext produced under the same reused nonce/counter:

```
flag = keystream XOR encrypted_flag
```

No key recovery is needed at all — the keystream itself is the key to breaking everything encrypted with it.

## Solution

```python
known_pt = bytes.fromhex("546865206861756e746564207661756c7420636c6f736573206174206d69646e696768742e2053746179206f7574736964652e20202020202020202020202020")
enc_msg  = bytes.fromhex("e4a8bb39921d6f0e3940a040ae22a1afc698fdeb3058ea8e1002fe25142227c6f7f018ddf3eb9ce75e69dbdc59720fcffd98a9c0bb9f03f0d644f34c68a4c306")
enc_flag = bytes.fromhex("c3b0b176910561033e16f151920887be9298bea77f0bafdd1043aa25596b6388beb75089fdebefb31f30db930c265c86b9dda7c0bb9f03f0d644f34c68a4c306")

keystream = bytes(a ^ b for a, b in zip(known_pt, enc_msg))
flag = bytes(a ^ b for a, b in zip(keystream, enc_flag))
print(flag)
```

1. XOR the known plaintext with its ciphertext to recover the keystream.
2. XOR that same keystream with the encrypted flag's ciphertext to recover the flag's plaintext.

## Flag

```
spooky{cs351JKS}
```

## What I learned

- Stream ciphers (and block ciphers run in stream modes like CTR, OFB, CFB) are only as safe as their nonce/IV management: reusing the same `(key, nonce, counter)` pair for two messages means both get the same keystream — a "two-time pad", the stream-cipher analogue of the classic one-time-pad reuse attack.
- Recovering the keystream never requires breaking AES itself: `keystream = known_plaintext XOR ciphertext` is pure XOR algebra, valid regardless of the cipher underneath, as long as the same keystream was used.
- Once a keystream is recovered for one position range, it decrypts _anything else_ encrypted at that same position range under the same reused nonce/counter — including secrets the attacker was never meant to see, like the flag here.
- A hardcoded nonce (`b"ghost123"`) combined with an `initial_value=0` that never changes between calls is the root cause; a correct implementation would use a fresh, unique nonce (or counter start) for every message.