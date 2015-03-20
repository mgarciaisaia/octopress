---
layout: post
title: "Dream of Serialization"
date: 2015-02-27 00:30
comments: true
categories: 
---
Las variables de C son nombres que refieren a una posición de memoria de la computadora. El compilador traduce cada nombre a una dirección de memoria, y el tamaño de la variable está determinado por su tipo.

Pero si bien el estándar de C pone algunas restricciones a los tamaños de los tipos de datos básicos, no todos tienen igual tamaño en las distintas plataformas, o incluso entre distintos compiladores de C para la misma plataforma.<!-- more -->

Además, para optimizar los accesos a memoria (operación lenta desde el punto de vista de la CPU), los compiladores suelen ubicar las variables _alineadas a la palabra de la CPU_. En general, la CPU no lee bytes individuales desde la memoria, si no que pide bloques del tamaño del ancho del bus de datos (4 bytes en máquinas de 32 bits, 8 en máquinas de 64 bits). Si una variable de 4 bytes estuviera ubicada en una posición no-múltiplo de 4 en un procesador de 32 bits, la CPU tendría que hacer 2 lecturas para accederla: una para leer el bloque de 4 bytes con los primeros bytes de la variable, y otra para el segundo bloque.

Esto aplica, también, a los `struct`s. Los campos de un `struct` se almacenan contiguamente en memoria, alineándose a la palabra. De hecho, `sizeof` nos va a devolver el tamaño del `struct` incluyendo los bytes de alineación (es decir, es muy probable que un `struct` compuesto por un `char` y un `int` - en ese orden - pese 8 bytes y no 5).

Todo esto fundamenta una única frase: no tenemos garantías sobre cómo van a organizarse los datos en un proceso que no sea el de nuestro propio programa. Encapsulamiento, que le dicen los hippies de Paradigmas :)

Entonces, ¿qué es la serialización? Bueno, es el arte de especificar un protocolo mediante el cual dos procesos van a comunicarse. Me encantaría poder hacer `send(socket, myStruct, sizeof(myStruct), ...)` y que eso funcione, pero si del otro lado me hacen `recv(socket, &myStruct, sizeof(myStruct), ...)` no tengo ninguna garantía de que los bytes estén en el orden correcto[^endianness] (ni de que el otro proceso también haya estado hecho en C, o que exista un `struct` como el que tengo yo, etc).

En fin, no puedo confiar en que mi estructura pueda viajar _como está_, entonces especifico mi propio protocolo: si tengo que mandar una estructura que tiene un `char` y un `int`, especifico - por ejemplo - que siempre voy a mandar un primer byte con el `char` y luego otros 4 bytes con el `int`.

De este modo, en lugar de hacer el `send()` de hace un rato, seguramente prefiera primero armar un buffer para meter todo el mensaje contiguo (_serializado_), y hacer `send` de eso[^disclaimer-codigo]:

{% codeblock lang:c Enviando una estructura medio primitivamente %}
void *buffer = malloc(5);
memcpy(buffer, &(myStruct.aChar), 1);
memcpy(buffer + 1, &(myStruct.anInt), 4);
send(mySocket, buffer, 5, ...);
{% endcodeblock %}

Del otro lado, al recibir la estructura, hago algo parecido: recibo 5 bytes (todo un mensaje), y copio el primer byte al `char`, y los 4 siguientes al `int`:

{% codeblock lang:c Recibiendo una estructura medio primitivamente %}
void *buffer = malloc(5);
recv(mySocket, buffer, 5, ...);
struct my_struct_t *myStruct = malloc(sizeof(struct my_struct_t));
memcpy(&(myStruct->aChar), buffer, 1);
memcpy(&(myStruct->anInt), buffer + 1, 4);
{% endcodeblock %}

De este modo, las estructuras de uno y otro lado están súper desacopladas: los campos podrían tener distintos nombres, podrían existir campos extras, tener distintas precisiones... Incluso podrían ser programas hechos en distintos lenguajes. Lo único que importa es que ambos extremos de la comunicación entiendan cómo deben interpretar esos bytes: si hago un programa en Prolog que relacione el primer byte recibido (andá a saber cómo) con un caracter, y los cuatro siguientes con un número, todos felices.

Es importante notar que tomamos decisiones re grosas: en nuestro protocolo, los caracteres son de 1 byte, y los enteros de 4, al menos para este mensaje. Si queremos hacer un cliente para plataformas cuyos `int` son de 2 bytes, vamos a tener que decidir cómo manejar los overflows (por ejemplo, usar variables de algún tipo más grande en nuestro `struct`, o truncar el valor recibido, o lo que sea), y mismo si estamos en 64 bits: nuestro `int` de 64 bits podría no ser representable en 4 bytes. No hay una receta universal para esto: son decisiones de diseño intrínsecas a nuestro problema.

_(Continuará...)_

[^endianness]: No olvidar, tampoco, little-endian vs big-endian, etc.
[^disclaimer-codigo]: El código que se muestra en el post tiene como prioridad ilustrar las ideas relacionadas con serialización con la menor cantidad de código extra posible. **No sigue** muchísimas buenas prácticas de programación, ni es digno de ser usado en código productivo, ni código de Trabajo Práctico. En particular, además de modularizar, elegir buenos nombres, etc, cualquiera que pretenda usar esto debería agregar chequeo de errores, reintentos, validaciones, etc... Y completar los parámetros faltantes de los `send` y `recv` :)