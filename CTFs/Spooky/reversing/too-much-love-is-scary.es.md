# Philophobia

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Categoría**  | Reversing |
| **Puntos**     | 50        |
| **Dificultad** | -         |

## Enunciado

> too much love is scary

Un ELF estático x86-64 (`philophobia`) lee dos enteros y una línea de texto, comprueba la línea contra una cita hardcodeada en una posición dada por el primer entero, y — si coincide — descifra con XOR e imprime un flag oculto usando una clave derivada de ambos enteros.

## Resumen

El binario se analizó únicamente a partir de la salida de `objdump` — no hizo falta debugger, ya que es un binario estático pequeño sin trucos anti-debug.

### `main` — leyendo el input

```asm
lea    rcx, [rip+...]            ; format string "%d %d"
lea    rdx, [rbp-0x414]          ; &num2
lea    rax, [rbp-0x418]          ; &num1
call   __isoc23_scanf@plt        ; scanf("%d %d", &num1, &num2)
...
call   getchar@plt               ; consume el '\n' sobrante
...
call   fgets@plt                 ; fgets(buf, 1000, stdin)  -> buf en rbp-0x3f0
```

`main` primero pide dos enteros (`num1`, `num2`), luego consume el salto de línea sobrante con `getchar()`, y después lee una línea completa de hasta 999 bytes en un buffer de la pila (`buf`, en `rbp-0x3f0`).

### La cita hardcodeada

Justo después del `fgets`, el binario construye en la pila una cadena de 20 bytes a partir de inmediatos crudos:

```asm
movabs $0x2049206c6c616853, rax   ; "Shall I "
movabs $0x20657261706d6f63, rdx   ; "compare "
mov    [rbp-0x410], rax
mov    [rbp-0x408], rdx
movl   $0x65656874, [rbp-0x400]   ; "thee"
```

Leídos en little-endian, esos bytes forman **`Shall I compare thee`** (20 caracteres — el inicio del soneto 18 de Shakespeare).

### La comprobación

```asm
mov    eax, [rbp-0x418]          ; num1
lea    rdx, [rbp-0x3f0]          ; buf
movsxd rax, eax
lea    rcx, [rdx+rax]            ; buf + num1
lea    rax, [rbp-0x410]          ; "Shall I compare thee"
mov    edx, 0x14                 ; longitud 20
mov    rsi, rax
mov    rdi, rcx
call   strncmp@plt               ; strncmp(buf + num1, "Shall I compare thee", 20)
test   eax, eax
jne    <fail>
```

Así que `num1` es en realidad un **offset dentro de la línea escrita**: los primeros 20 bytes a partir de `buf[num1]` tienen que ser exactamente `"Shall I compare thee"`. Si `num1 = 0`, la línea escrita simplemente tiene que _empezar_ por esa cita.

### `print_flag` — la clave XOR

Si tiene éxito, `main` llama a `print_flag(num1, num2)` (como `edi`, `esi`):

```asm
mov    eax, [rbp-0x48]           ; num2 (esi)
sub    eax, [rbp-0x44]           ; - num1 (edi)
add    eax, 0x47                 ; + 71
mov    [rbp-0x34], eax           ; key = num2 - num1 + 71
```

Así que la clave XOR de un byte usada para descifrar el flag es:

```
key = num2 - num1 + 71
```

### Los bytes del flag cifrado

`print_flag` también escribe un bloque de 36 bytes en la pila a partir de inmediatos `movabs`/`movl` crudos (`rbp-0x30` hasta `rbp-0x10`). Leídos en little-endian, en orden, los bytes crudos son:

```
29 2a 35 35 31 23 21 37 6e 23 38 69 05 6b 05 34
69 69 3e 05 2e 6a 05 6b 6a 2c 69 05 37 23 29 69
6b 3c 27 00
```

El último byte es `0x00` — un terminador simple. El bucle de descifrado hace XOR de cada byte con `key` y lo imprime con `putchar`, deteniéndose en cuanto encuentra un byte **crudo** de `0x00` (el bucle comprueba el byte sin XOR antes de imprimir, así que el terminador nunca se imprime):

```asm
; por cada iteración, i = 0..34
al = buf[-0x30 + i]        ; byte crudo cifrado
al ^= key                  ; XOR con la clave derivada
putchar(al)
i++
; el bucle continúa mientras buf[-0x30 + i] != 0 (comprobación cruda, antes del XOR)
```

## Solución

### Elegir `num1` y `num2`

`num1` solo necesita ser `0` (la cita debe estar justo al principio de la línea escrita). Para la clave, cualquier par `num1`/`num2` que haga que `num2 - num1 + 71` descifre el primer byte (`0x29`) a un carácter imprimible y coherente funciona. Probando `num1 = 0` y comprobando qué clave convierte `0x29` en `'s'` (el inicio esperado de `spooky{...}`):

```
0x29 XOR key == 's' (0x73)
key == 0x29 XOR 0x73 == 0x5A == 90
num2 - 0 + 71 == 90  →  num2 == 19
```

Así que `num1 = 0`, `num2 = 19` da `key = 90`.

### Construir el input

```
0 19
Shall I compare thee to a summer's day?
```

La primera línea satisface `scanf("%d %d", ...)`. La segunda línea la lee `fgets` y empieza exactamente con los 20 bytes requeridos de la cita (el resto del verso de Shakespeare es solo decoración — solo se comprueban los primeros 20 bytes).

### Descifrar a mano (para comprobar antes de conectar)

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

### Enviarlo al servicio

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

## Qué aprendí

- Un binario estático sin símbolos eliminados (`setup`, `print_flag`, `main` todos nombrados) es una pista fuerte para simplemente leer el disassembly de arriba a abajo en vez de recurrir a un debugger — el flujo de control y el layout de la pila ya se ven claros en el listado de `objdump`.
- Los inmediatos `movabs`/`movl` que escriben valores fijos en la pila son una forma habitual con la que los compiladores (y los autores de CTFs) embeben cadenas cortas o bloques de bytes directamente en el código en vez de en la sección `.rodata` — leerlos solo requiere decodificar cada inmediato en little-endian, en el orden en que se escriben.
- Un "cifrado" XOR de un solo byte es tan fuerte como su clave, y aquí la clave se derivaba de dos enteros controlados por el atacante mediante aritmética simple (`num2 - num1 + 71`) — una vez se adivina el primer byte del plaintext (`'s'` de `spooky{`), la clave sale directamente haciendo XOR con el primer byte del ciphertext.
- La comprobación `strncmp(buf + num1, ..., 20)` reutiliza uno de los enteros "clave" en un segundo papel — un offset dentro del buffer de entrada — algo fácil de pasar por alto si se asume que los dos enteros solo alimentan la clave XOR; rastrear cada uso de cada variable en el disassembly evita ese error.
- Un bucle de descifrado que comprueba el byte **crudo** antes del XOR para decidir cuándo parar (en vez de comprobar el resultado ya descifrado) implica que el terminador del ciphertext tiene que ser él mismo `0x00` — vale la pena notarlo para no incluir por error el último byte de un blob cifrado como parte del flag.