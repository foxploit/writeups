# Inyección SQL en cláusula WHERE para recuperar datos ocultos

| | |
|---|---|
| **Plataforma** | PortSwigger Web Security Academy |
| **Categoría** | SQL Injection |
| **Dificultad** | Apprentice |
| **Estado** | Resuelto |

## Objetivo

Saltarse un filtro de categorías que oculta productos no publicados, mediante inyección SQL en un parámetro de la URL.

## Reconocimiento

- Parámetro candidato identificado: `category`.
- Se probó la entrada reflejada para comprobar si llegaba a la base de datos. Un payload de `<script>` se imprimió como texto plano, así que el vector era SQL del lado del servidor y no XSS.
- El valor del parámetro va entre comillas, lo que determina cómo hay que cerrar el payload.

## Razonamiento

- La consulta filtra filas con una cláusula `WHERE` que comprueba la categoría y oculta los productos no publicados.
- Cerrar el literal de cadena con una comilla permite añadir una condición propia a la cláusula `WHERE`.
- Una condición siempre verdadera hace que el filtro deje de excluir filas, y una secuencia de comentario descarta el resto de la consulta original.

## Impacto

Un atacante podría saltarse restricciones de visibilidad. En una aplicación real, el mismo punto de inyección podría ampliarse con `UNION SELECT` para leer datos de otras tablas.

## Mitigación

- Usar consultas parametrizadas o prepared statements en lugar de concatenar la entrada del usuario en SQL.
- Validar la entrada en el servidor como defensa en profundidad.

## Lección aprendida

- Antes de elegir un payload, revisa cómo se entrecomilla la entrada del usuario en la consulta.
- Corregir el punto de inyección importa más que memorizar un payload concreto.
