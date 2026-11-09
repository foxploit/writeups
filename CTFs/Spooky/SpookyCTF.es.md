# SpookyCTF 2026 - Spanish Writeups

Writeups de los retos resueltos en el SpookyCTF de 2026 (17 retos y 1651 puntos). Cada uno incluye el enunciado, los pasos para resolverlo, la flag y que he aprendido al resolver el reto.

Realizado por [foxploit](https://github.com/foxploit) - [LinkedIn](https://www.linkedin.com/in/adrian-rodriguez-ortiz/)

## Retos

| Category  | Challenge                         | Points | Writeup                                                  |
| --------- | --------------------------------- | ------ | -------------------------------------------------------- |
| Web       | J*b Application                   | 50     | [writeup](web/job-application.es.md)                        |
| Web       | Hemophobia                        | 50     | [writeup](web/hemophobia.es.md)                              |
| Web       | Scopophobia                       | 50     | [writeup](web/scopophobia.es.md)                         |
| Crypto    | Heads will Roll                   | 50     | [writeup](crypto/heads-will-roll.es.md)                     |
| Crypto    | Numerophobia                      | 50     | [writeup](crypto/numerophobia.es.md)                        |
| Crypto    | Cyberphobia: SpOoKy KeYsTrEaMs    | 50     | [writeup](crypto/cyberphobia-spooky-keystreams.es.md)       |
| Crypto    | I Feel Like Someone's Watching Me | 50     | [writeup](crypto/someone-watching-me.es.md)                 |
| Crypto    | Diagraphephobia                   | 75     | [writeup](crypto/diagraphephobia.es.md)                     |
| Crypto    | No Nonsense on this Server        | 222    | [writeup](crypto/no-nonsense-on-this-server.es.md)          |
| Reversing | Cyberphobia: Overflowed Bits      | 50     | [writeup](reversing/cyberphobia-overflowed-bits.es.md)      |
| Reversing | Too Much Love is Scary            | 50     | [writeup](reversing/too-much-love-is-scary.es.md)           |
| Reversing | Frankenstein's Password Generator | 75     | [writeup](reversing/frankensteins-password-generator.es.md) |
| Reversing | Cyberphobia: Formatting Nightmare | 75     | [writeup](reversing/cyberphobia-formatting-nightmare.es.md) |
| Reversing | Trypophobia                       | 75     | [writeup](reversing/trypophobia.es.md)                      |
| Reversing | Elves can be Spooky Too           | 377    | [writeup](reversing/elves-can-be-spooky-too.es.md)          |
| Forensics | The Haunted Portrait              | 50     | [writeup](forensics/the-haunted-portrait.es.md)             |
| Misc      | Gelotophobia                      | 50     | [writeup](misc/gelotophobia.es.md)                          |

## Structure

Cada categoría de reto tiene su propia carpeta, y cada reto dentro de ella se guarda como un par de archivos — uno en inglés (`.md`) y su traducción (`.es.md`) — más una carpeta `static/` con las capturas que acompañan el writeup.

```
Spooky
├── crypto
│   ├── cyberphobia-spooky-keystreams.md / .es.md
│   ├── diagraphephobia.md / .es.md
│   ├── heads-will-roll.md / .es.md
│   ├── numerophobia.md / .es.md
│   ├── someone-watching-me.md / .es.md
│   └── static/
├── forensics
│   ├── the-haunted-portrait.md / .es.md
│   └── static/
├── misc
│   ├── gelotophobia.md / .es.md
│   └── static/
├── reversing
│   ├── cyberphobia-formatting-nightmare.md / .es.md
│   ├── cyberphobia-overflowed-bits.md / .es.md
│   ├── elves-can-be-spooky-too.md / .es.md
│   ├── frankensteins-password-generator.md / .es.md
│   ├── too-much-love-is-scary.md / .es.md
│   ├── trypophobia.md / .es.md
│   └── static/
├── web
│   ├── hemophobia.md / .es.md
│   ├── job-application.md / .es.md
│   ├── scopophobia.md / .es.md
│   └── static/
├── SpookyCTF.md        
└── SpookyCTF.es.md    ← este índice
```

**Convenciones:**

- Un archivo por reto y por idioma: `<slug>.md` (inglés) y `<slug>.es.md` (español).
- Capturas de pantalla en `static/`, nombradas `<slug>-NN.png` cuando hay varias por reto.
- `<slug>` es el nombre del reto en minúsculas y con guiones (`job-application`, `heads-will-roll`...).

## Herramientas

Burp Suite, CyberChef, GDB, ltrace, Python, curl