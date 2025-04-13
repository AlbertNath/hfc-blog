+++
title = "Reversing Basicos: Arquitectura de Computadoras"
date = "2025-02-16T09:12:08-06:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "lorne"
authors = ["lorne"]
authorTwitter = "" #do not include @
cover = ""
tags = ["Reversing", "Binary Explotation"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
+++

# Básicos

Toda la memoria está indexada en hexadecimal. En la memoria viven las instrucciones en ensamblador.

Dependiendo de la arquitectura, tendremos entre 8 y 32 registros. Un registro puede verse como variables globales. Permiten realizar opreaciones aritmético-lógicas según su tipo. El tamaño de estos registros depende de la arquitectura del procesador (32 o 64 bits).

Si los números que manejamos son más grandes que el tamaño máximo de nuestros registros, tendremos que **dividir el número entre más de un registro**. Si a pesar de esto el espacio es insuficiente, tendremos que echar mano de **la memoria** que no esté ocupada.

Uno de los registros más importantes es **Program Counter (PC)** que inidica al procesador la dirección de memoria con la siguiente instrucción a ejecutar. También es conocido como **Instruction Pointer**.

## Heap y stack

Son dos estructuras de datos asociadas a cualquier proceso. El stack se encuentra al inicio del espacio de memoria asignado al proceso y crece "hacia abajo". Tiene registros especiales asociados:

- **Stack Pointer (SP/ESP/RSP)** apunta a la dirección en memoria que está al tope de la pila.

## Caché

Idealmente, todo un proceso debería usar solo registros, pero esto es poco probable que ocurra (el software moderno es pesado).

Los accesos a memoria **son caros**, por lo que aprovechamos el caché de un CPU (estructuras de memoria especiales y muy rápidas) que almacenan direcciones de memoria si se usan frequentemente.

# Instrucciones de ASM

Ensamblador es un lenguaje de muy bajo nivel que permite comunicarnos casi directamente con el CPU. Se basa en **instrucciones**, que operan con los registros de una CPU.

La sintaxis de este lenguaje puede variar según la arquitectura y las convenciones, AT&T o Intel. La sintaxis y convención más usada es IA32 de Intel (presente desde el mítico i386).

## Banderas

Algunas de las instrucciones a contiunación modifican registros especiales llamados **banderas**, que indican información adicional tras una operación.

- `zeroflag` 1 si el último valor calculado fue cero, en otro caso su valor es 0.

## Instrucciones básicas

- `mov <reg_destino>, <valor>` mueve el valor al registro especificado `reg_destino`. Notemos que el valor puede ser otro registro. Si el valor es del tipo `[addr]`, entonces trae un valor de una dirección en memoria `addr` y lo carga en el registro destino.
- `add <reg_destino>, <valor>` suma el valor al contenido del registro, el resultado se almacena en el registro destino.
- `sub <reg_destino>, <valor>` resta el valor al contenido del registro, el resultado se almacena en el registro destino.
- `push <valor>` agrega un valor a la pila y hace que SP apunte a la nueva dirección en el tope. ``
- `pop <reg_dest>` el valor que está al tope de la pila es sacado y almacenado en el registro destino y el SP es decrementado.

## Instrucciones de flujo de control

Estas instrucciones ayudan a modificar o definir un fliujo que el programa debería seguir.

- `jmp <addr>` el PC se mueve a la dirección especificada
- `je <addr>` si `zeroflag` esta activa, ve a la dirección especificada
- `call` una mezcla entre `push`, `jmp` y `ret`. Primero pone en el tope de la pila la dirección de la siguiente instrucción a ejecutar, luego, va a la dirección de una función especificada y al terminar, ejecuta `ret`
- `ret` hace `pop` a la pila y pone la dirección en PC.
