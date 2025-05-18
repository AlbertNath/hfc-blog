+++
date = '2025-05-17T15:20:55-06:00'
draft = true 
title = 'Contribuye'
+++

# Documentación y lineamientos del blog | HFC

## URL

[https://blog.hackersfight.club/](https://blog.hackersfight.club/)

## Introducción

El blog tiene por objetivos:

- Crear contenido de interés para la comunidad de seguridad.
- Ayudar a todos los autores/colaboradores a crear su portafolio de publicaciones, con el fin de mejorar su CV.
- Ayudar a hacer más grande la agrupación, mediante la colaboración activa de sus miembros.

Para lograr estos objetivos, es necesario mantener cierto orden y calidad en las publicaciones. Por lo tanto, los miembros
de la grupación que colaboren en este proyecto deben tener claridad sobre el proceso que seguiremos para crear nuevas entradas
y las reglas a seguir al momento de redactar las mismas. Es por esto que creamos esta documentación.

Si tienes dudas, no dudes en contactar en el servidor de HFC de Discord a:

- @Albert
- @Vicarus
- @Lorne
- @sh4d3
- @Vir

---

## Presentación

Antes de publicarse, cada entrada pasará por una revisión de calidad. Pero, para hacer este trabajo más sencillo, te pedimos tener cuidado con:

- Ortografía
- Redacción
- Presentación

**Lineamientos**:

- Todas las entradas del blog estarán escritas en _markdown_.
- Cada entrada deberá estar en español. Si quieres que tus entradas tengan versión en inglés, deberás compartir **adicionalmente** una versión en inglés.
- Utiliza el script `build-workspace.sh`, para crear la estructura de trabajo de la
  siguiente manera:

  ```bash
  $ ./build-workspace.sh [nombre_de_tu_entrada]
  ```

  Se crearán dos subdirectorios:

  - `content/posts/[nombre_de_tu_entrada].md`: aquí pudes escribir tu contribución.
  - `static/img/[nombre_de_tu_entrada]/`: si es **absoultamente necesario** , puedes poner las imágenes en este directorio.

  **Es importante que respetes la estructura creada**. Puedes descargar el script
  {{< download src="/downloads/code/build-workspace.sh"
    type="application/x-sh" filename="build-workspace.sh"  class="btn">}}
  aquí.
  {{</download >}}

- Usa doble numeral ( ## ) para los títulos de las diferentes secciones.
- Usa comillas para referenciar títulos (p. ej. el título de la máquina de HTB).
- Usa dos "backticks" ( \`ejemplo\` ) para escribir texto técnico corto (p. ej. comandos, funciones, parámetros HTTP, etc.).
- Usa bloques de código ( \`\`\`ejemplo\`\`\` ) para escribir texto técnico de varias líneas (p. ej. código fuente, peticiones HTTP, archivos de configuración, etc.).
- Usa letra cursiva (\*ejemplo\*) para escribir texto en inglés que prefieres no traducir a español. (p. ej. _Server-Side Request Forgery_).
- Si el texto de un bloque de código es demasiado largo y hay secciones que no aportan, puedes quitarlas y poner en su lugar la cadena "<--REDACTED-->".
- A menos que no haya alternativa, evita incluir imágenes. Por ejemplo, es preferible que pongas el texto de una petición HTTP en un bloque de código a poner una captura de BurpSuite.
- Si pones imágenes, procura editarlas para recalcar la información de interés. Esto lo puedes lograr con un recuadro rojo.
- Cada entrada incluye metadatos, como el autor. Sé consistente con tu alias a lo largo de diferentes entradas. Esto es sensible a mayúsculas y minúsculas.
- Para categorizar tu entrada, incluye las etiquetas que mejor se ajusten a la misma. Es importante que no crees tus versiones de dichas etiquetas. En su lugar tómalas de la siguiente sección.

## Etiquetas

### Por tipo de contenido

- Miscellaneous
- Research
- Tutorial
- Writeup

### Por tipo de enfoque

- Blue Team
- Red Team
- Purple Team

### Por tema

- Active Directory
- AI
- Architecture
- Blockchain
- Bug Bounty
- Cloud
- Crypto
- CTI
- Dev Sec Ops
- Forensics
- Game Hacking
- Hardening
- Hardware
- Intrusion Detection
- IoT
- Malware Analysis
- Mobile
- Networking
- OSINT
- OT
- Perimeter Security
- Pwning
- Red Team Ops
- Reversing
- Secure Coding
- Social Engineering
- Threat Hunting
- Web
- Wireless

### Por sistema operativo

- Android
- iOS
- Linux
- MacOS
- Windows

**Nota:** Si consideras que falta alguna etiqueta, comunícate con los miembros de la agrupación mencionados anteriormente.

---

## Envía tu contribución

Una vez que tengas lista tu contribución, comprime el directorio generado por el script
`build-workspace.sh` usando `zip` y envía el comprimido a:

- @Vir
- @Lorne

Se te informará si la contribución requiere modificaciones para satisfacer los líneamientos o si
es aprobada, en cuyo caso pasará a formar parte del blog.
