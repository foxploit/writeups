# Spooky Keystreams

|**CTF**|SpookyCTF|
|---|---|
|**Category**|Crypto|
|**Points**|50|
|**Difficulty**|Easy|

## Enunciado

> Nothing is secure nowadays, not even the own ciphers my machine generates. Take a look for yourself, I don't know how people on the internet were able to retrieve my encrypted flag with just these files...

## Resumen

```
Spooky Keystreams

Read backup.py, then run python solve.py.
Enter hex or filename:line when prompted. Use cat -n data.txt to see line numbers.

Known message: 546865206861756e746564207661756c7420636c6f736573206174206d69646e696768742e2053746179206f7574736964652e20202020202020202020202020
Encrypted message: e4a8bb39921d6f0e3940a040ae22a1afc698fdeb3058ea8e1002fe25142227c6f7f018ddf3eb9ce75e69dbdc59720fcffd98a9c0bb9f03f0d644f34c68a4c306
Encrypted flag: c3b0b176910561033e16f151920887be9298bea77f0bafdd1043aa25596b6388beb75089fdebefb31f30db930c265c86b9dda7c0bb9f03f0d644f34c68a4c306
```

`backup.py` muestra cómo el servidor cifra los mensajes:

```python
from Crypto.Cipher import AES

def encrypt(message, key):
    cipher = AES.new(key, AES.MODE_CTR, nonce=b"ghost123", initial_value=0)
    return cipher.encrypt(message.ljust(64, b" "))
```

El cifrado es AES en **modo CTR**, con un **nonce fijo, hardcodeado** (`b"ghost123"`) y un **valor inicial de contador fijo** (`0`). El modo CTR funciona generando un keystream a partir de `(key, nonce, counter)` y haciendo XOR con el plaintext: `ciphertext = plaintext XOR keystream`.

La propiedad crítica del modo CTR es: si la misma combinación `(key, nonce, counter)` se reutiliza para dos mensajes distintos, ambos mensajes obtienen **exactamente el mismo keystream**. Y eso es justo lo que pasa aquí — cada llamada a `encrypt()` usa la misma clave, el mismo nonce, y arranca el contador en 0 cada vez. Tanto el mensaje conocido como el flag se cifraron con el mismo keystream idéntico.

Como `ciphertext = plaintext XOR keystream`, esto significa:

```
keystream = known_plaintext XOR known_ciphertext
```

y una vez recuperado el keystream, se puede reutilizar sobre cualquier otro ciphertext producido bajo el mismo nonce/counter reutilizado:

```
flag = keystream XOR encrypted_flag
```

No hace falta recuperar la clave en absoluto — el propio keystream es la llave para romper todo lo cifrado con él.

## Solución

```python
known_pt = bytes.fromhex("546865206861756e746564207661756c7420636c6f736573206174206d69646e696768742e2053746179206f7574736964652e20202020202020202020202020")
enc_msg  = bytes.fromhex("e4a8bb39921d6f0e3940a040ae22a1afc698fdeb3058ea8e1002fe25142227c6f7f018ddf3eb9ce75e69dbdc59720fcffd98a9c0bb9f03f0d644f34c68a4c306")
enc_flag = bytes.fromhex("c3b0b176910561033e16f151920887be9298bea77f0bafdd1043aa25596b6388beb75089fdebefb31f30db930c265c86b9dda7c0bb9f03f0d644f34c68a4c306")

keystream = bytes(a ^ b for a, b in zip(known_pt, enc_msg))
flag = bytes(a ^ b for a, b in zip(keystream, enc_flag))
print(flag)
```

1. Hacer XOR del plaintext conocido con su ciphertext para recuperar el keystream.
2. Hacer XOR de ese mismo keystream con el ciphertext del flag cifrado para recuperar el plaintext del flag.

## Flag

```
spooky{cs351JKS}
```

## Qué aprendí

- Los cifrados de flujo (y los cifrados de bloque usados en modos de flujo como CTR, OFB, CFB) son tan seguros como su gestión de nonce/IV: reutilizar el mismo par `(key, nonce, counter)` para dos mensajes hace que ambos obtengan el mismo keystream — un "two-time pad", el equivalente en cifrado de flujo del clásico ataque por reutilización de one-time-pad.
- Recuperar el keystream nunca requiere romper AES en sí: `keystream = known_plaintext XOR ciphertext` es álgebra XOR pura, válida sin importar el cifrado de fondo, siempre que se haya usado el mismo keystream.
- Una vez recuperado el keystream para un rango de posiciones, descifra _cualquier otra cosa_ cifrada en ese mismo rango bajo el mismo nonce/counter reutilizado — incluyendo secretos que el atacante nunca debería haber podido ver, como el flag aquí.
- Un nonce hardcodeado (`b"ghost123"`) combinado con un `initial_value=0` que nunca cambia entre llamadas es la causa raíz; una implementación correcta usaría un nonce (o punto de partida del contador) nuevo y único para cada mensaje.