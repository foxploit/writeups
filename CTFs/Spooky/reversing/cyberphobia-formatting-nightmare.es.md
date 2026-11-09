# Cyberphobia: Formatting Nightmare

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Categoría**  | Reversing |
| **Puntos**     | 75        |
| **Dificultad** | Easy      |

## Enunciado

Un binario (`chall2`) lee un secreto de un archivo en el servidor y luego pide input al usuario. Se da el código fuente:

```c
void vuln(const char *secret) {
    char input[256];
    printf("Input: ");

    if (fgets(input, sizeof(input), stdin) == NULL)
        return;

    printf(input, secret);
    printf("\n");
}

int main(void) {
    char secret[256];

    FILE *fp = fopen("/opt/ctf/chall2/secret.txt", "r");
    if (fp == NULL) {
        perror("/opt/ctf/chall2/secret.txt");
        return 1;
    }

    if (fgets(secret, sizeof(secret), fp) == NULL) {
        fclose(fp);
        return 1;
    }
    fclose(fp);

    vuln(secret);
    return 0;
}
```

## Resumen

El fallo está en esta única línea:

```c
printf(input, secret);
```

`input` — lo que sea que escriba el usuario — se pasa directamente como el **format string** de `printf`, con `secret` como único argumento extra. Normalmente un programa llamaría a `printf("%s", secret)`, donde el format string es fijo y de confianza; aquí el propio format string viene de un input no confiable. Es un caso de manual de [format string vulnerability](https://community.owasp.org/attacks/Format_String_attack).

Como el atacante controla el format string, puede elegir qué especificador de conversión se aplica a lo siguiente que `printf` encuentre en la pila — en este caso, el argumento `secret` que ya se había pasado.

## Solución

Enviando `%s` como input, `printf` trata su único argumento real (`secret`) como una cadena y lo imprime:

```bash
nc 18.226.103.146 9002
```

```
Input: %s
f0rm4tStr1ngVulN3r4b1lityIsSc4rY
```

El servidor devuelve el secreto directamente, ya que `%s` le dice a `printf` "lee un puntero a cadena de la lista de argumentos e imprímelo" — y el único puntero disponible es el que contiene el secreto del archivo.

## Flag

```
spooky{f0rm4tStr1ngVulN3r4b1lityIsSc4rY}
```

## Qué aprendí

- Cualquier llamada de la familia `printf` donde el **primer argumento venga del input del usuario** (`printf(input, ...)` en vez de `printf("%s", input)`) es una vulnerabilidad de format string, sin importar qué haga el resto del código.
- El arreglo siempre está en fijar el format string, no el input: `printf("%s", input)` en vez de `printf(input)` — nunca dejar que datos del usuario decidan qué especificadores de conversión se aplican.
- Un bug de format string no necesita control sobre direcciones de memoria para ser útil: aquí, simplemente dar `%s` basta para filtrar un argumento que el programa nunca pretendía exponer, porque ya estaba en la lista de argumentos de la llamada.
- El artículo de OWASP sobre format string attacks (https://community.owasp.org/attacks/Format_String_attack) es una buena referencia sobre el abanico más amplio de primitivas que habilita esta clase de bug (leer datos arbitrarios de la pila con `%x`/`%p`, escribir en memoria con `%n`), más allá del simple leak usado aquí.


