# Cyberphobia: Formatting Nightmare

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Category**   | Reversing |
| **Points**     | 75        |
| **Difficulty** | Easy      |

## Statement

A binary (`chall2`) reads a secret from a file on the server and then asks the user for input. The source code is given:

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

## Overview

The bug is this single line:

```c
printf(input, secret);
```

`input` — whatever the user typed — is passed directly as the **format string** to `printf`, with `secret` as the only extra argument. Normally a program calls `printf("%s", secret)`, where the format string is fixed and trusted; here the format string itself comes from untrusted user input. This is a textbook [format string vulnerability](https://community.owasp.org/attacks/Format_String_attack).

Because the attacker controls the format string, they can choose which conversion specifier gets applied to whatever `printf` finds next on the stack — in this case, the `secret` argument that was already passed in.

## Solution

Sending `%s` as the input makes `printf` treat its one real argument (`secret`) as a string and print it:

```bash
nc 18.226.103.146 9002
```

```
Input: %s
f0rm4tStr1ngVulN3r4b1lityIsSc4rY
```

The server echoes the secret straight back, since `%s` tells `printf` "read a string pointer from the argument list and print it" — and the only pointer available is the one holding the file's secret.

## Flag

```
spooky{f0rm4tStr1ngVulN3r4b1lityIsSc4rY}
```

## What I learned

- Any `printf`-family call where the **first argument comes from user input** (`printf(input, ...)` instead of `printf("%s", input)`) is a format string vulnerability, independent of what the rest of the code does.
- The fix is always to fix the format string, not the input: `printf("%s", input)` instead of `printf(input)` — never let user data decide which conversion specifiers get applied.
- A format string bug doesn't require control over memory addresses to be useful: here, simply supplying `%s` is enough to leak an argument the program never intended to expose, because it was already sitting on the call's argument list.
- OWASP's write-up on format string attacks (https://community.owasp.org/attacks/Format_String_attack) is a good reference for the wider range of primitives this bug class enables (reading arbitrary stack data with `%x`/`%p`, writing memory with `%n`), beyond the simple leak used here.