
# I Feel Like Someone's Watching Me

A letter is found beside a tombstone in a graveyard, but it seems to have been **scrambled around**.

## Resumen

El reto da el texto de una carta (unas 1500 letras) que a primera vista no tiene sentido:

```
Ypejpof Erpveaypj,

Pmpj odagp zlw rptf, D'mp tprf zlwj eiopagp. Iwf D'mp erol tprf fcp qjpopagp lt ealfcpj, fclwuc D yla'f nalx xcl df do. ...
```

## Solución

### Identificar el cifrado

La carta parece inglés normal con las letras cambiadas: los espacios, la puntuación, las mayúsculas y los apóstrofos están en su sitio, solo son distintas las letras. Eso encaja con un **cifrado de sustitución monoalfabética**, donde cada letra se sustituye siempre por la misma otra letra. Algunas pistas:

- `D` es una palabra de una letra y siempre abre las contracciones: `D'rr`, `D'mp`, `D'h` son `I'll`, `I've` e `I'm`.
- `fcp` es la palabra de tres letras que más se repite, así que es `the`.

El enunciado dice que la carta está "revuelta" (*scrambled around*), lo que encaja.

### Resolverlo

Un cifrado de sustitución de esta longitud (unas 1500 letras) se puede resolver automáticamente con [quipqiup](https://quipqiup.com): se pega el texto y la primera solución ya es la legible.

```
Dearest Alexander, Ever since you left, I've felt your absence. But I've also felt the presence of another, though I don't know who it is. ...
```

### Leer la carta

La flag no es una cadena evidente, se explica dentro de la carta. Cerca del final:

```
But I'll put a clue that should help you find me if you still care. eyez are watching me. Just remember underscores instead of spaces and to wrap it correctly.
```

Así que la flag es la pista `eyez are watching me` (con la `z`, tal como está escrita en la carta), con guiones bajos en vez de espacios y dentro del formato de la flag (`spooky{...}`).

### Comprobar la clave

Una vez resuelto, se puede escribir la sustitución. Letras del texto normal y las letras cifradas que las sustituyen:

```
plain:  a b c d e f g h i j k l m n o p q r s t u v w x y z
cipher: e i g y p t u c d b n r h a l q k j o f w m x v z s
```

Se puede aplicar en local con `tr`, de cifrado a normal:

```bash
tr 'eigyptucdbnrhalqkjofwmxvzsEIGYPTUCDBNRHALQKJOFWMXVZS' 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ' < letter.txt
```

La pista del texto cifrado (`pzps ejp xefgcdau hp`) da `eyez are watching me`.

## Flag

```
spooky{eyez_are_watching_me}
```

## Lo aprendido

- Un cifrado de sustitución se reconoce porque conserva la estructura del texto (espacios, puntuación, palabras de una letra, contracciones), solo cambian las letras.
- Los solucionadores automáticos como quipqiup lo resuelven en segundos si el texto es lo bastante largo, así que no hace falta hacer análisis de frecuencias a mano.
- Merece la pena leer todo el texto descifrado: la flag estaba explicada dentro de la carta, con el formato que había que usar.