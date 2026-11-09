# I Feel Like Someone's Watching Me

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Category**   | Crypto    |
| **Points**     | 50        |
| **Difficulty** | Easy      |

## Statement

A letter is found beside a tombstone in a graveyard, but it seems to have been **scrambled around.**

## Overview

The challenge gives the text of a letter (about 1500 letters) that makes no sense at first sight:

```
Ypejpof Erpveaypj,

Pmpj odagp zlw rptf, D'mp tprf zlwj eiopagp. Iwf D'mp erol tprf fcp qjpopagp lt ealfcpj, fclwuc D yla'f nalx xcl df do. ...
```

## Solution

### Identify the cipher

The letter looks like normal English with its letters changed: the spaces, punctuation, capital letters and apostrophes are in place, only the letters are different. That matches a **monoalphabetic substitution cipher**, where each letter is always replaced by the same other letter. Some clues:

- `D` is a word of one letter and always starts the contractions: `D'rr`, `D'mp`, `D'h` are `I'll`, `I've` and `I'm`.
- `fcp` is the most repeated word of three letters, so it is `the`.

The statement says that the letter is "scrambled around", which fits.

### Solve it

A substitution cipher of this length (about 1500 letters) can be solved automatically with [quipqiup](https://quipqiup.com): paste the text and the first solution is already the readable one.

```
Dearest Alexander, Ever since you left, I've felt your absence. But I've also felt the presence of another, though I don't know who it is. ...
```

### Read the letter

The flag is not an obvious string, it is explained inside the letter. Near the end:

```
But I'll put a clue that should help you find me if you still care. eyez are watching me. Just remember underscores instead of spaces and to wrap it correctly.
```

So the flag is the clue `eyez are watching me` (with the `z`, as written in the letter), with underscores instead of spaces and inside the flag format (`spooky{...}`).

### Check the key

Once solved, the substitution can be written down. Plain letters and the cipher letters that replace them:

```
plain:  a b c d e f g h i j k l m n o p q r s t u v w x y z
cipher: e i g y p t u c d b n r h a l q k j o f w m x v z s
```

It can be applied locally with `tr`, from cipher to plain:

```bash
tr 'eigyptucdbnrhalqkjofwmxvzsEIGYPTUCDBNRHALQKJOFWMXVZS' 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ' < letter.txt
```

The clue in the ciphertext (`pzps ejp xefgcdau hp`) gives `eyez are watching me`.

## Flag

```
spooky{eyez_are_watching_me}
```

## What I learned

- A substitution cipher is recognized because it keeps the structure of the text (spaces, punctuation, single-letter words, contractions), only the letters change.
- Automatic solvers like quipqiup solve it in seconds if the text is long enough, so there is no need to do frequency analysis by hand.
- It is worth reading the whole decoded text: the flag was explained inside the letter, with the format to use.