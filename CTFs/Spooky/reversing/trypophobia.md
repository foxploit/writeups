# Trypophobia

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Category**   | Reversing |
| **Points**     | 75        |
| **Difficulty** | -         |

## Statement

> Classic reverse engineering, right?

A Java class (`binary`) checks a flag passed as a command-line argument against a chain of per-character arithmetic/bitwise conditions.

## Overview

```java
public class binary {
    private static boolean check(String string) {
        if (string.length() == 34) {
            if ((0x1F ^ string.charAt(0)) + 7 == 115 && (73 - string.charAt(1) ^ 0xB) == -46) {
                if ((string.charAt(2) + 5 ^ 0x16) == 98 && 19 - string.charAt(3) + 51 == -41) {
                    // ... 17 nested if-pairs in total, one per two characters
                                                                            return true;
                }
            }
        }
        return false;
    }

    public static void main(String[] stringArray) {
        if (stringArray.length == 0) {
            System.out.println("Usage: java binary <flag>");
            return;
        }
        if (binary.check(stringArray[0])) {
            System.out.println("Correct!");
        } else {
            System.out.println("Incorrect.");
        }
    }
}
```

`check()` requires a 34-character string and then tests it two characters at a time, nested inside 17 `if` blocks. Each pair of characters has its own independent equation — character `i` never depends on any other character, it only has to satisfy its own constant. Every condition is a mix of XOR, addition/subtraction and a fixed constant, something like:

```
(0x1F XOR charAt(0)) + 7 == 115
(73 - charAt(1)) XOR 0xB == -46
```

Since each equation involves exactly one unknown (`charAt(i)`) and only reversible operators (XOR is its own inverse, add/subtract just move terms across), every character can be solved independently by algebra — no brute force needed.

## Solution

### Invert each equation

For an XOR-based condition such as `(K ^ c) + A == R`:

```
c = K ^ (R - A)
```

For a subtraction-based condition such as `(B - c) ^ M == R`:

```
c = B - (R ^ M)
```

Applying the matching inverse to each of the 34 conditions in the decompiled source gives one `chr(...)` expression per character:

```python
flag = [''] * 34

flag[0]  = chr(0x1F ^ (115 - 7))
flag[1]  = chr(73 - (-46 ^ 0xB))
flag[2]  = chr((98 ^ 0x16) - 5)
flag[3]  = chr(19 + 51 + 41)
flag[4]  = chr(0x5E ^ (66 - 13))
flag[5]  = chr(37 - (-69 ^ 0x17))
flag[6]  = chr((128 ^ 0xC) - 17)
flag[7]  = chr(3 + 83 + 15)
flag[8]  = chr(0x29 ^ (124 - 29))
flag[9]  = chr(66 - (-44 ^ 9))
flag[10] = chr((142 ^ 9) - 21)
flag[11] = chr(15 + 55 + 25)
flag[12] = chr(0x58 ^ (69 - 27))
flag[13] = chr(14 - (-74 ^ 0x1F))
flag[14] = chr((50 ^ 0x4F) - 7)
flag[15] = chr(25 + 33 + 43)
flag[16] = chr(0x3D ^ (90 - 11))
flag[17] = chr(27 - (-71 ^ 0x11))
flag[18] = chr((66 ^ 0x30) - 13)
flag[19] = chr(25 + 88 - 13)
flag[20] = chr(0x20 ^ (136 - 9))
flag[21] = chr(69 - (-57 ^ 0x1C))
flag[22] = chr((96 ^ 0xD) - 12)
flag[23] = chr(31 + 50 + 37)
flag[24] = chr(0x57 ^ (69 - 15))
flag[25] = chr(31 - (-62 ^ 0x22))
flag[26] = chr((23 ^ 0x44) - 18)
flag[27] = chr(37 + 12 + 5)
flag[28] = chr(0x31 ^ (27 - 21))
flag[29] = chr(86 - (-15 ^ 5))
flag[30] = chr((105 ^ 0x1E) - 24)
flag[31] = chr(8 + 67 - 18)
flag[32] = chr(0xB ^ (85 - 27))
flag[33] = chr(48 - (-72 ^ 0xB))

print(''.join(flag))
```

Running it reconstructs the flag character by character, with no need to execute the Java binary or brute-force anything.

## Flag

```
spooky{ever_reversed_java?A67b_91}
```

## What I learned

- Decompiling with `cfr` (or any Java decompiler) turns bytecode checks back into readable `if` conditions, which is usually the fastest path for this kind of "obfuscated validator" challenge — no need to touch bytecode or a debugger if the decompiled source is already this explicit.
- When a validator checks each character (or pair of characters) with its own independent equation, the whole thing can be solved algebraically, character by character, instead of brute-forcing the keyspace.
- The two operators used here, XOR and addition/subtraction, are both trivially invertible: `a ^ b == c` always gives `a == b ^ c`, and `a + b == c` always gives `a == c - b` — so any chain built only from these can be undone mechanically, equation by equation.
- Deeply nested `if` statements (one per character pair) are a common decompiler artifact of a single `&&`-chained boolean expression or a loop that got unrolled; they don't add real complexity, just visual noise.