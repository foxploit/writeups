# Elves can be Spooky Too

| **CTF**        | SpookyCTF |
| -------------- | --------- |
| **Categoría**  | Reversing |
| **Puntos**     | 377       |
| **Dificultad** | Hard      |

## Enunciado

> The elves left you a very small doorway. You are given a host, a port, and a byte budget. Send an x86-64 Linux ELF that: is at most 112 bytes (the service prints the live limit on connect), writes its own exact bytes to stdout, exits with status 0.

```
nc 3.129.162.36 31903
```

Binrev puro de "ELF golfing". Me apoyé en una IA (Claude) para la parte de diseño de bytes y en este artículo para entender bien la técnica base: [Handcrafting x64 ELF: From Specification to Bytes](https://medium.com/@dassomnath/handcrafting-x64-elf-from-specification-to-bytes-9986b342eb89).

## Resumen

Un `Ehdr` estándar (64 bytes) más un `Phdr` (56 bytes) ya suman 120 bytes solo de cabeceras, sin una sola instrucción. El límite son 112 bytes. Hay que solapar las dos estructuras.

**La clave:** el kernel, al hacer `execve`, solo valida unos pocos campos — el magic number, la clase, el endianness, `e_type`, `e_machine`, `e_phentsize` (debe ser 56), `e_phnum`, `e_phoff`, y dentro del segmento `PT_LOAD`: `p_type`, `p_flags`, `p_offset`, `p_vaddr` y `p_filesz`. Todo lo demás — `e_shoff`, `e_flags`, `e_ehsize`, `e_shnum`, `e_shstrndx`, el padding de `e_ident`, `p_paddr`, e incluso `p_align` — lo ignora. Esos campos ignorados son exactamente donde se puede esconder shellcode real en vez de un valor cualquiera.

## Solución

### El plan de layout

1. Poner `e_phoff = 56`, de modo que el `Phdr` solapa con la cola del `Ehdr` — el fichero entero queda en 112 bytes exactos.
2. Esconder shellcode en tres huecos libres: el padding de `e_ident` (9 bytes), el trío `e_shoff` / `e_flags` / `e_ehsize` (14 bytes), y `p_paddr` (8 bytes).
3. Usar `jmp` cortos para saltar por encima de los campos que el kernel sí valida (`p_offset = 0`, `p_vaddr`, `p_filesz = 112`...) sin que la CPU intente ejecutarlos como instrucciones.
4. Hacer que `e_entry` apunte al primer hueco libre.

### El shellcode, repartido en los tres huecos

```asm
push 1; pop rdi          ; rdi = 1 (stdout)
push 1; pop rax          ; rax = 1 (SYS_write)
jmp   +25                ; salta sobre e_type/e_machine/e_version/e_entry/e_phoff
; -- siguiente hueco --
mov esi, 0x400000        ; rsi = dirección base donde está mapeado el propio fichero
push 0x70; pop rdx       ; rdx = 112
syscall                  ; write(1, base, 112)  -> se imprime a sí mismo
xor edi, edi             ; rdi = 0
jmp   +26                ; salta sobre p_offset/p_vaddr/p_filesz/p_memsz/p_align
; -- siguiente hueco --
push 0x3C; pop rax        ; rax = 60 (SYS_exit)
syscall                   ; exit(0)
```

Es básicamente un quine: el propio binario, al ejecutarse, imprime sus propios bytes exactos.

### Payload final (112 bytes, hex)

```
7f454c46020101006a015f6a0158eb190002003e000100000007004000000000003800000000000000be000040006a705a0f0531ffeb1a38000100000005000000000000000000000000004000000000006a3c580f05000000700000000000000070000000000000000000000000000000
```

Lo probé primero en local (una VM/sandbox x86-64) antes de tirarlo contra el servidor: se ejecuta, sale con exit code 0, y `cmp`/`sha256sum` confirman que su `stdout` coincide byte a byte con el propio fichero.

### Script para hablar con el servicio

El protocolo del servicio es: banner → `SIZE 112\n` → los 112 bytes crudos → leer `PASS`/`FLAG`/`FAIL`.

```python
import socket, struct

HOST, PORT = "3.129.162.36", 31903

def build_elf():
    b = bytearray(112)
    def put(o, d): b[o:o+len(d)] = d
    put(0, bytes([0x7F,0x45,0x4C,0x46])); b[4]=2; b[5]=1; b[6]=1
    put(7, bytes([0x6A,0x01,0x5F,0x6A,0x01,0x58,0xEB,0x19,0x00]))
    BASE = 0x400000
    put(16, struct.pack('<H',2)); put(18, struct.pack('<H',0x3E))
    put(20, struct.pack('<I',1)); put(24, struct.pack('<Q',BASE+7))
    put(32, struct.pack('<Q',56))
    put(40, bytes([0xBE,0x00,0x00,0x40,0x00,0x6A,0x70,0x5A,0x0F,0x05,0x31,0xFF,0xEB,0x1A]))
    put(54, struct.pack('<H',56)); put(56, struct.pack('<H',1))
    put(58, struct.pack('<H',0)); put(60, struct.pack('<H',5)); put(62, struct.pack('<H',0))
    put(64, struct.pack('<Q',0)); put(72, struct.pack('<Q',BASE))
    put(80, bytes([0x6A,0x3C,0x58,0x0F,0x05,0x00,0x00,0x00]))
    put(88, struct.pack('<Q',112)); put(96, struct.pack('<Q',112)); put(104, struct.pack('<Q',0))
    return bytes(b)

payload = build_elf()
with socket.create_connection((HOST, PORT), timeout=15) as s:
    print(s.recv(4096).decode(errors="replace"))
    s.sendall(f"SIZE {len(payload)}\n".encode())
    s.sendall(payload)
    print(s.recv(4096).decode(errors="replace"))
```

## Flag

```
spooky{phdr_in_ehdr_c2d021cc4fc5}
```

## Qué aprendí

- El kernel de Linux solo valida un subconjunto pequeño y específico de la cabecera ELF y del program header al hacer `execve` — todo lo que queda fuera de ese subconjunto (bytes de padding, campos sin uso como `p_paddr`, `e_shoff`, `e_flags`) es espacio libre para meter código extra.
- Solapar el `Phdr` con la cola del `Ehdr` (haciendo que `e_phoff` apunte dentro del propio `Ehdr`) es el mayor ahorro de espacio en ELF golfing — elimina 56 bytes de estructura que de otro modo serían obligatorios.
- Los `jmp` relativos cortos son imprescindibles cuando el shellcode se intercala con campos de cabecera que el kernel sí comprueba: si no, la CPU intentaría ejecutar esos valores validados (offsets, tamaños, flags) como instrucciones.
- Un quine de "imprime tus propios bytes" se vuelve trivial en cuanto se sabe dónde está mapeado el fichero en memoria (la base de `e_entry`, aquí `0x400000` para un ejecutable no-PIE) — `write(1, base, tamaño_del_fichero)` es todo el programa.
- Probar el binario manipulado en local primero (comprobando el exit code y comparando el `stdout` con el fichero en disco) evita errores de layout antes de gastar un round-trip de red contra el servidor del reto.