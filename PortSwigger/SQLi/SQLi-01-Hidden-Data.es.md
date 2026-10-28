# Inyección SQL en cláusula WHERE que permite recuperar datos ocultos

| | |
|---|---|
|**Autor**|foxploit|
|**Plataforma**|PortSwigger Web Security Academy|
|**Categoría**|SQL Injection|
|**Dificultad**|⭐☆☆☆☆ (Apprentice)|
|**Estado**|✅ Resuelto|
|**URL del laboratorio**|[portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)|

## Objetivo

Saltarse el filtro de productos para recuperar productos ocultos/no publicados mediante inyección SQL en el parámetro `category`.

## Reconocimiento

- Parámetros controlables identificados: `productId` y `category`.
- Se probó `category=1 OR 1=1 --` → se reflejaba sin sanitizar.
- Se descartó XSS: `<script>alert("1")</script>` se imprimía como texto plano, sin ejecutarse — confirmando que el vector era SQL del lado del servidor, no del cliente.

## Explotación

1. Se construyó un payload teniendo en cuenta el contexto de cadena del parámetro `category`:

```
Gifts' OR 1=1 --
```

2. Se envió la petición:

```
GET /filter?category=Gifts'+OR+1=1+--+
```

3. La respuesta incluyó productos ocultos/no publicados — filtro saltado.

|Antes|Después|
|---|---|
|<img src="./screenshots/01-before.png" width="400">|<img src="./screenshots/01-after.png" width="400">|

**Payload aplicado:** `Gifts' OR 1=1 --`

## Impacto

Un atacante puede saltarse las restricciones de categoría/visibilidad para ver productos no publicados. En un entorno de producción, el mismo punto de inyección podría extenderse para extraer datos de otras tablas mediante `UNION SELECT`.

## Mitigación

- Usar consultas parametrizadas / prepared statements en lugar de concatenar la entrada del usuario en SQL.
- Aplicar validación de entrada del lado del servidor como medida de defensa en profundidad.

## Lección aprendida

- Una comilla simple (`'`) cierra un literal de cadena SQL — ese es el punto de inyección.
- `OR 1=1` siempre es verdadero → salta la condición `WHERE` precedente.
- `--` comenta el resto de la query original, neutralizando todo lo que va después del payload.