# Trypophobia

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Categoría**  | Reversing |
| **Puntos**     | 75        |
| **Dificultad** | -         |

## Enunciado

> Classic reverse engineering, right?

Una clase Java (`binary`) valida un flag pasado como argumento de línea de comandos contra una cadena de condiciones aritméticas/bitwise por carácter.

## Resumen

```java
public class binary {
    private static boolean check(String string) {
        if (string.length() == 34) {
            if ((0x1F ^ string.charAt(0)) + 7 == 115 && (73 - string.charAt(1) ^ 0xB) == -46) {
                if ((string.charAt(2) + 5 ^ 0x16) == 98 && 19 - string.charAt(3) + 51 == -41) {
                    // ... 17 pares de if anidados en total, uno por cada dos caracteres
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

`check()` exige una cadena de 34 caracteres y luego la comprueba de dos en dos, anidada dentro de 17 bloques `if`. Cada par de caracteres tiene su propia ecuación independiente — el carácter `i` nunca depende de ningún otro carácter, solo tiene que satisfacer su propia constante. Cada condición es una mezcla de XOR, suma/resta y una constante fija, algo así:

```
(0x1F XOR charAt(0)) + 7 == 115
(73 - charAt(1)) XOR 0xB == -46
```

Como cada ecuación involucra exactamente una incógnita (`charAt(i)`) y solo operadores reversibles (XOR es su propia inversa, sumar/restar solo mueve términos de un lado a otro), cada carácter se puede resolver de forma independiente mediante álgebra — sin necesidad de fuerza bruta.

## Solución

### Invertir cada ecuación

Para una condición basada en XOR como `(K ^ c) + A == R`:

```
c = K ^ (R - A)
```

Para una condición basada en resta como `(B - c) ^ M == R`:

```
c = B - (R ^ M)
```

Aplicando la inversa correspondiente a cada una de las 34 condiciones del código decompilado se obtiene una expresión `chr(...)` por carácter:

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

Al ejecutarlo se reconstruye el flag carácter a carácter, sin necesidad de ejecutar el binario Java ni hacer fuerza bruta.

## Flag

```
spooky{ever_reversed_java?A67b_91}
```

## Qué aprendí

- Decompilar con `cfr` (o cualquier decompilador de Java) convierte las comprobaciones del bytecode de vuelta en condiciones `if` legibles, que suele ser el camino más rápido para este tipo de reto de "validador ofuscado" — no hace falta tocar el bytecode ni usar un debugger si el código fuente decompilado ya es así de explícito.
- Cuando un validador comprueba cada carácter (o par de caracteres) con su propia ecuación independiente, todo el reto se puede resolver algebraicamente, carácter a carácter, en vez de hacer fuerza bruta sobre el espacio de claves.
- Los dos operadores usados aquí, XOR y suma/resta, son ambos trivialmente invertibles: `a ^ b == c` siempre da `a == b ^ c`, y `a + b == c` siempre da `a == c - b` — así que cualquier cadena construida solo con estos se puede deshacer mecánicamente, ecuación a ecuación.
- Los `if` profundamente anidados (uno por cada par de caracteres) son un artefacto habitual de los decompiladores a partir de una única expresión booleana encadenada con `&&`, o de un bucle que se desenrolló; no añaden complejidad real, solo ruido visual.