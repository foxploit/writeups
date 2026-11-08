![[03.png]]

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Category**   | Misc      |
| **Points**     | 50        |
| **Difficulty** | Easy      |

## Statement

Trying to make someone laugh? Then check out this website for some awesome jokes!

_this challenge contains ... bright, flashing, strobe lights. Those who are seizure-prone / have epilepsy are advised to skip this challenge._

## Overview

The challenge is a webpage with a "Generate Joke" button. Each click loads a joke about some topic, and the page is full o visual noise.

![[static/01.png]]

## Solution

### Look at the URL

When a joke is generated, the URL contains a long path that looks like base64:

```
http://18.217.64.238/RUVFRUVFRUVFZWVlRUVFRUVFRUVFRUVFRWVlRWVFRWVFRUVFRUVFRUVlZWVlRWVFRUVFRUVFRUVFZWVlZUVlRUVFRUVFRUVFRWVlRUVFRWU=
```

The joke shown was about pizza, so the path probably encodes that word.

## Decode the path

Decoding the base64 with `Cetacean Cypher` returns `pizza` so the path is just the topic of the joke, encoded as `Cetacean Cypher -> base64`

![[static/02.png]]

## Encode the `flag` the same way

In cyberchef with `flag` as input, use `Cetacean Cypher Encode` and the `To Base64`:

```
RUVFRUVFRUVFZWVFRWVlRUVFRUVFRUVFRWVlRWVlRUVFRUVFRUVFRUVlZUVFRUVlRUVFRUVFRUVFZWVFRWVlZQ==
```

![[static/03.png]]

The same can be done with python (`solve.py`):

```python
import base64 
def encode(word): 
	bits = "".join(f"{b:08b}" for b in word.encode("utf-16-be")) 
	s = "".join("E" if c == "0" else "e" for c in bits) 
	return base64.b64encode(s.encode()).decode() 
print(encode("flag"))
```

### Request it

You can either request it via web or via command:

![[static/04.png]]

## Flag

```
spooky{th4t_w4snt_funn3h}
```

## What I learned

- A long base64 in a URL is always worth decoding, and the result can be another layer of encoding. 
- A string made of only two symbols (here `E` and `e`) usually points to a known esoteric cipher, and CyberChef's "Magic" or a quick search of the pattern helps identify it. 
- Once the encoding is understood, you can control the input by encoding your own value.