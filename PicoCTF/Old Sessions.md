# Old Sessions

## Enunciado

Proper session timeout controls are critical for securing user accounts. If a user logs in on a public or shared computer but doesn’t explicitly log out (instead simply closing the browser tab), and session expiration dates are misconfigured, the session may remain active indefinitely.

This then allows an attacker using the same browser later to access the user’s account without needing credentials, exploiting the fact that sessions never expire and remain authenticated.

## Resumen

Lo primero al entrar a la URL es un panel de `login`:

![[Pasted image 20261109214201.png]]

## Solución

### Crear Usuario `test`

Creo un usuario llamado `test` y lo primero al iniciar sesión que encuentro es un comentario con una pista hacia la ruta `/sessions`

![[Pasted image 20261109214403.png]]

### Acceder a `/sessions`

Al acceder a la ruta `/sessions` encuentro las sesiones de todos los usuarios de la maquina.

```
1) session:s-KEbk3N1nXpm_YgvqKyR-jB9wpR6Py5sgwJSOApxD8, {'_permanent': True, 'key': 'admin'}

2) session:s6V2jyy4oZxUt2juZN-TpFUauKZhJTTiI18ANP8V3ls, {'_permanent': True, '_flashes': [['error', None]]}

3) session:B4_S-nPYGjXy56887rAAOLkbJoykf842vG0RzkUKgPg, {'_permanent': True, 'key': 'test'}
```

La que interesa es la del usuario `admin`

### Entrar como administrador 

Con Cookie-Editor modifico la cookie para que sea identica a la del administrador:

![[Pasted image 20261109214638.png]]

Recargo la página y ya veo la flag en texto plano.

![[Pasted image 20261109214718.png]]

## Lo aprendido

- Los comentarios de los demás usuarios pueden ser útiles cuando no tienes más información a tu alcance.
- Las rutas mal securizadas pueden exponer cookies de usuarios administradores que se pueden reutilizar para suplantar la identidad de los mismos.