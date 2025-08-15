+++
title = "Del hook a la evasión con Frida"
date = "2025-08-07T20:27:25-06:00"
author = "Lorne"
authors = ["Lorne"]
tags = ["Mobile","Android"]
showFullContent = false
hideComments = true 
+++

> Contexto: Este artículo habla sobre el uso de la herramienta Frida, utilizada para pruebas de penetración en entornos móviles como Android y iOS, su objetivo es enseñar a realizar scripts propios evitando depender de los ya existentes.

## Preparando el entorno

Para poder realizar nuestros ejercicios con frida, es necesario subir un binario de frida server que será el que nos permitirá hookear procesos. Es necesario que el dispositivo esté rooteado, por lo que basta con utilizar algun AVD que no tenga la API de Google Play, o uno rooteado con aplicaciones como Magisk y RootAVD.

[---> Descargar Frida desde Github <---](https://github.com/frida/frida/releases)

Una vez descargada la versión de nuestra preferencia (Yo usé la 16.x), lo subimos a alguna ruta cómoda para trabajar, como `/data/local/tmp` y lo ejecutamos, puede ser en segundo plano -o no- , ya saben, dando permisos de ejecución y listo:

`chmod +x ./frida-server*` y `./frida-server*`

Una vez listo el servidor y corriendo, podríamos empezar a hookear procesos, para este ejemplo particular me decidí por trabajar con un reto sencillo de Hack the Box llamado Pinned, pero puede ser realmente cualquier APK, el objetivo no es resolver el reto en sí, sino entender cómo funciona frida y aprender a craftear nuestros scripts personalizados. Nota: tener bases en reversing (de binarios ELF o SAST puede ayudar a interiorizar mejor el contenido de este artículo, pero realmente **no es indispensable**).

Una vez que tenemos iniciada la APK a analizar, tenemos que encontrar el identificador de proceso al que Frida se adjuntará (attach), esto puede hacerse con `frida-ps -U` y para filtrar por el proceso que nos interesa, podemos usar algún grep case insensitive, quedándonos tal que así: `frida-ps -U | grep -i <APP>`. Con lo anterior preparado, vamos ahora sí a lo que nos interesa: hooking!

Una última cosa, para cargar nuestros scripts al proceso (nótese que no estoy diciendo inyectar) podemos usar el comando `frida -U -p <PID> -l <SCRIPT>`.

---

## ¿Qué quéremos lograr con nuestro script?

Una vez que tenemos la posibilidad de cargar nuestros scripts y tal, nos viene a la mente la duda: ¿Cómo sabemos qué script queremos para según qué app? bueno, pues la respuesta es: depende de qué querramos lograr. en mi caso, quise intentar entender paso a paso el código, e identificando las funciones de interés que se ejecutaban, procedí a hookearlas, agregando un mensaje casero de debug:

```javascript
if (Java.available) {
  Java.perform(function () {});
  //Mi lógica de scripting
}
```

El anterior no es más que el esqueleto que usaremos para trabajar, indica si el JVM (El entorno en tiempo de ejecución) está listo para que podamos empezar a trabajar. Para manipular el flujo de la aplicación -como dice el párrafo anterior- necesitamos tener en claro qué queremos lograr, en nuestro objetivo inicial será entender cómo funciona la aplicación, previamente podemos leer el código con herramientas como `jadx-gui`, pero manteniéndonos en las bases: ¿Cómo le indicamos a frida qué función, variable u objeto queremos manipular?

Toda aplicación de Android tiene una clase MainActivity, y para este caso, la manera de indicar que queremos usar la clase MainActivity es la siguiente:

```javascript
if(Java.available){
	Java.perform(function(){
		// Aquí metí un mensaje para asegurarnos que configuramos todo bien
		console.log("[+] Frida funciona!");

		const MainActivity = Java.use("com.example.pinned.MainActivity");
	);
}
```

De esta manera, podemos trabajar con las funciones que estén dentro de esta clase, en este caso vemos la función `w()`:
Primero, pongamos un mensaje casero de debugg cuando la función `w()` se active.

```javascript
if (Java.available) {
  Java.perform(function () {
    console.log("[+] Frida funciona!");

    const mainActivity = Java.use("com.example.pinned.MainActivity");

    //Copiamos la función original
    const wOriginal = mainActivity.w;

    mainActivity.w.implementation = function () {
      //metemos la lógica deseada
      console.log("[+] funcion w hookeada con exito!");
      // enviamos la función original como estaba
      return wOriginal.call(this);
    };
  });
}
```

Entonces cada que hagamos alguna acción en la App que ejecute dicha función, nos imprimirá en consola que se hookeó, la razón por la que escogí primero esa función es para que entendamos algo importante más adelante.

---

## Inspeccionando variables internas

Para empezar, un contexto, la siguiente es una imagen de la misma clase MainActivity, que instancia diferentes variables:

{{< image src="/img/Del_hook_a_la_evasion_con_Frida/20250727141406.png" alt="img1" position="center" >}}

Podemos leer variables internas de la siguiente manera, a continuación digamos un ejemplo para la variable o, que mañosamente escogí porque entre todas, es un String:

```javascript
console.log("[+] Valor original de o:", this.o.value);
```

y así con cualuier valor, ahora, ¿Podemos modificarlas? pero claro que sí, podemos hacer todo eso y más, con algo como:

```javascript
this.o.value = Java.use("java.lang.String").$new("Hola mundo");
```

En el código anterior cambiamos lo que séa que valga o, por "Hola mundo", preguntota, ¿Esto funciona para cualquier variable?, respuesta corta: no.
Aquí el primer obstáculo. — **no todo puede reemplazarse así**. Por ejemplo, si `this.p` es un `java.security.cert.Certificate`, no podemos meterle un `String`. Los tipos deben ser compatibles o usar `null` si el objetivo es forzar un fallo.

El siguiente es un ejemplo de lo que ocurre si tenemos alguna inconsistencia al tratar de modificar variables con tipos de datos:

{{< image src="/img/Del_hook_a_la_evasion_con_Frida/20250727141831.png" alt="img1" position="center" >}}

La salida muestra que la función w se hookea, (ya vimos como poner eso) pero algo pasa al tratar de imprimir un valor, si prestamos atención, notaremos que menciona que es un tipo de dato Certificate, en otras palabras, es el certificado bajo el cual se comunica con alguna API, y abajo lo confirmamos, donde además tenemos la URL y algunos headers, ojo, pareciera que lo que está ahí es la contraseña, y si leemos el código sabremos que sí es, pero para resolver el reto, debemos -en teoría- ser capaces de interceptar esta petición para obtener la bandera, pero SPOILER: el SSL Pinning nos bloquea.

{{< image src="/img/Del_hook_a_la_evasion_con_Frida/20250727141255.png" alt="img1" position="center" >}}

---

## SSL Pinning en pocas palabras (Y cómo nos vale)

Es un mecanismo que fija certificados de confianza en una aplicación, como medida contra los ataques de Man-in-the-Middle (MiTM), pero como tenemos frida y podemos modificar valores en tiempo de ejecución, eso no nos interesa, tenemos información suficiente.

En muchas apps, el método que carga el certificado es algo como:

`this.p = CertificateFactory.getInstance("X.509").generateCertificate(getResources().openRawResource(R.raw.certificate));`

Y ese certificado se inyecta luego en un `SSLContext`. (¿UN QUÉ?)

**SSLContext:**

> Es el encargado de decidir **en qué certificados confiar** cuando se conecta por HTTPS.  
> Si le das un certificado "duro" (como el que la app trae embebido), entonces **solo aceptará ese**.  
> Pero si tú le cambias el `SSLContext` por uno que acepta todo... ¡la app confía en cualquiera! (como Burp o tu propio proxy malicioso).

Entonces nuestra lógica debería quedar algo como:

```javascript
MainActivity.w.implementation = function () {
  console.log("[+] Bypass de SSL Pinning activado");
  // Opcional: Setear valores falsos o nulos
  this.p = null;
  this.q = null;
  return;
  // Omitimos lógica original
};
```

Eso o hacer que el método no haga nada, pero devuelva el control sin errores.

Finalmente, usando todo lo anterior, y con un poco de recursos sobre cuáles son y cómo encontrar las clases encargadas de todo este proceso, metemos una política de lo más laxa y que permita todos los certificados (incluso si son nulos).

```javascript
if (Java.available) {
  Java.perform(function () {
    console.log("[+] Iniciando hook de SSL Pinning");

    const SSLContext = Java.use("javax.net.ssl.SSLContext");
    const TrustManager = Java.use("javax.net.ssl.X509TrustManager");
    const HttpsURLConnection = Java.use("javax.net.ssl.HttpsURLConnection");

    const TrustAllManager = Java.registerClass({
      name: "hfc.lorne.TrustAllCerts",
      implements: [TrustManager],
      methods: {
        checkClientTrusted: function (chain, authType) {},
        checkServerTrusted: function (chain, authType) {},
        getAcceptedIssuers: function () {
          return [];
        },
      },
    });

    const trustManagers = Java.array("javax.net.ssl.TrustManager", [
      TrustAllManager.$new(),
    ]);
    const sslContext = SSLContext.getInstance("TLS");
    sslContext.init(null, trustManagers, null);

    // Hook global (más efectivo)
    HttpsURLConnection.setDefaultSSLSocketFactory(
      sslContext.getSocketFactory(),
    );

    // Hook SSLContext.init por si alguna clase lo usa directamente
    SSLContext.init.overload(
      "[Ljavax.net.ssl.KeyManager;",
      "[Ljavax.net.ssl.TrustManager;",
      "java.security.SecureRandom",
    ).implementation = function (a, b, c) {
      console.log(
        "[*] Interceptado SSLContext.init(), forzando TrustManager inseguro...",
      );
      this.init(a, trustManagers, c);
    };

    console.log(
      "[+] Bypass aplicado. Todos los certificados serán confiables.",
    );
  });
}
```

Y una vez cargado nuestro script, tenemos evasión de SSL Pinning, somos capaces de interceptar y leer las peticiones realizadas, entre las cuales obtenemos la bandera de este reto.

{{< image src="/img/Del_hook_a_la_evasion_con_Frida/20250727140649.png" alt="img1" position="center" >}}

---

## Conclusión

Hookear funciones con frida no es nada dificil, y crear scripts personalizados tampoco, bueno, puede resultar complejo al principio, pero no toma mucho tiempo entender qué pasa por detrás, de manera que podemos anular las lógicas de seguridad, tanto como la que vimos, como controles de seguridad biométricos, detección de root, en fin...

Este flujo que comienza con `console.log("Frida funciona!")` y termina aceptando todos los certificados de una app demuestra que aprender Frida es una herramienta imprescindible para cualquier pentester móvil que busque entender a profundidad las aplicaciones que audita.

Los reto a hacer su propia versión del script, creando su propio certificado, y a explorar cómo hookear actividades dinámicamente cuando se lanzan en diferentes pantallas, entendiendo esto.

@Lorne (PD: el eMAPT está blandito, con esto ya tienen suficiente para la parte práctica, suerte!)

---

## ¿Y ya?

Bueno, plot twist, mientras realizaba este artículo y jugaba con frida, mediante la técnica más básica, fui capaz de evadir más allá de la lógica de seguridad del SSL, obteniendo la bandera incluso antes de enviarse (client-side issues).

{{< image src="/img/Del_hook_a_la_evasion_con_Frida/20250727142425.png" alt="img1" position="center" >}}

**\*\*** Mientras intentaba modificar el valor de la variable `o`, fui capaz de leer el valor de la misma, que contenía la bandera (tal como en la petición). **\*\*\***
