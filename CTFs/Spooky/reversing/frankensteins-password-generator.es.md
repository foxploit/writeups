# Frankenstein's Password Generator

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Categoría**  | Reversing |
| **Puntos**     | 75        |
| **Dificultad** | Medium    |

## Enunciado

> Dr. Frankenstein left behind a strange password generator in an abandoned NJIT laboratory. The machine refuses to start without an activation phrase. His last notebook entry reads, "A bolt at midnight gave it life." Can you figure out what the program expects and wake the creature?

Un binario (`frankenstein`) pide una contraseña y no arranca si no es la correcta.

## Resumen

El binario no guarda la contraseña en claro — la tiene cifrada en memoria como `*$*)C7EP\]erto`. Para cada carácter del input aplica:

```
resultado = (input[i] XOR 0x5d) + i*7 + 11
```

y compara el resultado byte a byte con los bytes cifrados. Si coinciden todos, la contraseña se acepta.

Al ser una transformación simple por carácter sin dependencia entre ellos, es directamente invertible: cada byte de salida depende solo del byte de entrada en el mismo índice y de su posición `i`.

## Solución

### Invertir la fórmula

Dado `resultado[i] = (input[i] XOR 0x5d) + i*7 + 11`, despejando `input[i]`:

```
input[i] = (resultado[i] - i*7 - 11) XOR 0x5d
```

### Aplicarla al secreto cifrado

```python
secret = b'*$*)C7EP\\]erto'
password = bytes([(secret[i] - i*7 - 0xb) ^ 0x5d for i in range(14)])
print(password.decode())  # → BOLTATMIDNIGHT
```

La pista del cuaderno lo decía desde el principio: "A bolt at midnight gave it life" → `BOLTATMIDNIGHT`.

### Ejecutarlo

```bash
echo "BOLTATMIDNIGHT" | ./frankenstein
# The laboratory lights flicker. IT'S ALIVE!
# The creature whispers: <flag>
```

## Flag

```
spooky{its_alive_at_midnight}
```

## Qué aprendí

- Una transformación por carácter del tipo `f(input[i], i)` sin dependencia entre caracteres (sin estado acumulado, sin realimentación de bytes anteriores) siempre es directamente invertible — basta despejar `input[i]` de la ecuación.
- Cuando un reto da tanto una constante cifrada como la transformación exacta (XOR + aritmética), no hace falta fuerza bruta: se escribe la función inversa y se aplica una sola vez.
- Una pista narrativa escondida en el texto de ambientación ("A bolt at midnight gave it life") suele ser la contraseña literal disfrazada — vale la pena probar lecturas obvias de la frase antes de asumir que es solo temática.