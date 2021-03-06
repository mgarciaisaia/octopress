---
layout: post
title: "Dream of Serialization"
date: 2015-02-27 00:30
comments: true
categories: 
---

_Disclaimer: escribí este post hace unos días, a mano, y, ahora que lo estoy releyendo para tipearlo en la compu, no me parece tan genial como cuando lo escribí la primera vez. Recomiendo tomarlo bastante más con pinzas que de costumbre, y donde encuentren algo que les haga ruido - o directamente sientan que lo que digo es incorrecto, o que el enfoque no es bueno - avisen, así lo rewordeo. Por favor, perdón y gracias._

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

## Manejando distintos mensajes

Pero probablemente nuestro programa pueda/tenga que mandar y recibir distintos tipos de mensajes. Para saber de qué tipo de mensaje se trata, nuestro protocolo podría especificar que antes de enviar cada mensaje se envíe una cantidad fija de bytes que identifique qué tipo de mensaje estamos enviando, de modo que el receptor sepa cómo decodificarlo.

Otro problema frecuente es el de los mensajes de longitud variable: los `int`s pesan siempre lo mismo en una arquitectura determinada, pero el string `"hola"` no pesa lo mismo que `"Hola, che, ¿cómo te va?"`. ¿Cómo serializamos eso? ¿Cómo sabe el proceso remoto cuántos bytes tendría que `recv`ir para leer el mensaje completo?

Lo mejor es tratar de trabajar siempre con tamaños fijos, cosa de que de ambos extremos de la comunicación sepan cuántos bytes enviar y cuántos esperar recibir. Pero _tamaño fijo_ podría interpretarse como _todos los strings que mando van a ser de 80 bytes_ y, si bien en algunos casos (en los que la comunicación está bastante acotada y predefinida) eso puede ser suficiente, en muchos otros casos puede ser un problema: un tamaño fijo muy grande desperdicia muchos bytes para mensajes chicos (imaginen mandar 1MB por internet sólo para decir "Hola"), y tamaños muy chicos podrían obligarnos a truncar los mensajes o tener que pensar algún mecanismo para mandar el mensaje en varias partes[^olor-fragmentacion].

Entonces, hagamos algo mejor: elijamos una cantidad fija de bytes en la que especificar _cuánto mide nuestro string_, y enviemos ese número primero, y luego todos los bytes que conforman el mensaje. De este modo, el cliente siempre recibe una cantidad fija de bytes (por ejemplo, 4), interpreta esos bytes como un número, y luego recibe esa cantidad de bytes como mensaje:

{% codeblock lang:c Enviando mensajes de longitud variable %}
size_t messageLength = strlen(message);
void *buffer = malloc (4 + messageLength);
memcpy(buffer, messageLength, 4);
memcpy(buffer + 4, message, messageLength);
send(mySocket, buffer, 4 + messageLength, ...);
{% endcodeblock %}

Y el cliente:

{% codeblock lang:c Recibiendo mensajes de longitud variable %}
size_t messageLength;
recv(mySocket, &messageLength, 4, ...);
void *message = malloc(messageLength);
recv(mySocket, message, messageLength, ...);
{% endcodeblock %}

De este modo, el cliente no se bloquea, porque siempre sabe cuánto debe recibir.

Ojo que nosotros enviamos `strlen(message)` bytes, por lo que el `\0` que termina los strings en C no viajó: si queremos que lo que recibimos sea un string, tenemos que agregarlo a mano al final del mensaje (y reservar un byte más en el `malloc` del receptor)[^enviar-barra-cero].

También es importante notar que lo que enviamos es el contenido del string, y no el puntero. Para mandar un `int`, a send le pasamos `&myInt`, porque lo que necesita es un puntero a los bytes a enviar. Pero para mandar un string, en los ejemplos que vimos acá le pasamos directamente `myString`, porque ese es el puntero: si pasáramos `&myString` a `send`, le estaríamos pidiendo que mande _la dirección del string en nuestro proceso_, dato que al proceso remoto **no le sirve** para leer el contenido del mensaje.

Por último, y para hacer un único `recv` bloqueante en lugar de hacer varios `recv` que bloqueen por pocos datos, podría ser útil mandar siempre como primer dato del mensaje el tamaño total del mismo, obviamente eligiendo una cantidad fija de bytes en la que representar este dato. De este modo, el receptor siempre lee esa cantidad fija de bytes, luego hace un único `recv` del mensaje completo, y una vez que lo tiene, lo procesa por completo.

## Serialización vs sockets

_Che, y... Para serializar, ¿necesito saber sockets?_

Bueno, no[^saber-sockets]. La serialización es una herramienta que permite representar en un modo "primitivo", simple de almacenar y/o transmitir, alguna estructura más o menos compleja de un sistema.

En el TP es muy común serializar estructuras para enviarlas a otro proceso mediante sockets, pero también podríamos serializar el estado del proceso (o una parte del mismo) si necesitáramos _guardar_ el estado de una tarea para resumirla más adelante. Por ejemplo, cuando usamos VirtualBox, al cerrar la VM con la opción "Save the machine state", VirtualBox serializa la memoria y todo el estado de la VM actual y lo persiste en un[os] archivo[s][^serializacion-smalltalk].

Estos ejemplos claramente son independientes de los sockets (una vez serializado _el dato_, podemos simplemente hacer un `fwrite` a un archivo en disco), y aún así son perfectos casos de serialización.

[^endianness]: No olvidar, tampoco, little-endian vs big-endian, etc.
[^disclaimer-codigo]: El código que se muestra en el post tiene como prioridad ilustrar las ideas relacionadas con serialización con la menor cantidad de código extra posible. **No sigue** muchísimas de las buenas prácticas de programación, ni es digno de ser usado en código productivo, ni código de Trabajo Práctico. En particular, además de modularizar, elegir buenos nombres, etc, cualquiera que pretenda usar esto debería agregar chequeo de errores, reintentos, validaciones, etc... Y completar los parámetros faltantes de los `send` y `recv` :)
[^olor-fragmentacion]: Si sienten un olorcito a _fragmentación_, vienen bien
[^enviar-barra-cero]: Esto colabora un poco con el encapsulamiento: en otro lenguaje, los strings podrían no necesitar el `\0` al final - es una cuestión de implementación que ese `\0` exista.
[^saber-sockets]: Lo que no significa que no debas saber sockets para aprobar la materia :)
[^serializacion-smalltalk]: Lo mismo ocurre cuando cerramos un ambiente de Smalltalk guardando los cambios
