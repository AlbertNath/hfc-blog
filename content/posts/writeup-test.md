+++
title = "HTB Cyber Apocalypse 2025 | CyberAttack WriteUp"
date = "2025-02-16T09:12:08-06:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "Vir"
authors = ["Vir"]
authorTwitter = "" #do not include @
cover = ""
tags = ["Writeup", "Web", "HTB"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
+++

# Introducción
Recientemente se llevó a cabo el CTF anual de HackTheBox "Cyber Apocalypse". Uno de los retos que más me gustó dentro de la categoría "Web" fue "Cyber Attack". Como todos los retos de este CTF, este está ambientado en el mundo fantástico de Eldoria. Se nos presenta una aplicación web destinada a atacar a los enemigos de Malakar.

A grandes rasgos, el reto requería explotar una inyección CRLF, una vulnerabilidad de tipo SSRF y una inyección de comandos. Adicionalmente para lograr resolverlo, fue necesario comprender algo llamado "ambigüedad semántica" en servidores Apache y aplicar un pequeño truco en IPv6 para conseguir inyectar comandos. Para resolver el reto, se nos provee el código fuente de la aplicación y archivos de configuración del servidor web.