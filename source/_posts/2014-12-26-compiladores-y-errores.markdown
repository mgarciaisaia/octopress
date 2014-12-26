---
layout: post
title: "Compiladores Y Errores"
date: 2014-12-26 19:17
comments: true
categories:
---

Hace un tiempo me llegó la siguiente consulta por mail:

{% blockquote %}
Cuando trato de correr un proceso por consola me tira un mensaje "violacion de segmento" ('core' generado). Como mi proceso compilaba correctamente, no me tira errores, me resultaba extraño que me tire en tiempo de ejecución ese error.

Mi pregunta es, ¿puede ser que los errores del segmentation fault no me los alerte en tiempo de compilacion?{% endblockquote %}<!--more-->

¡Claro que el programa puede fallar incluso cuando no tiene problemas de compilación! El software fallaría mucho menos si lo único que necesitáramos es que el compilador reporte cero errores. Pero hay montones de cosas que afectan la ejecución de un programa, y que exceden por completo las capacidades del compilador, y por eso el compilador **no puede** reportarlas.

Un error del compilador significa que el código es sintácticamente inválido, al punto de que el compilador no puede traducir tu texto a código de máquina. Pero entre código válido y código funcional hay un tremendo salto, y entre código válido y código sin errores, puff.

Pero además de "detectar código mal formado", el compilador tiene los warnings, que son avisos de cosas que probablemente estés haciendo mal. Un programa con warnings compila y puede ejecutarse, pero existe una gran probabilidad de que haya errores lógicos que hagan que funcione de modo inesperado.

Y hasta ahí puede llegar el compilador. Pero hay montones de errores que el compilador no puede detectar, por lo que no es una gran garantía la compilación.

Por esto es que hace años que se viene laburando un montón en crear cosas como los tests, las buenas prácticas, los patrones de diseño, la delegación y encapsulamiento como medios para reutilizar código, la abstracción, etc.
TL;DR: es muuuuuuuy difícil garantizar la ausencia de errores en el software, al punto de que es inviable para cualquier programa más o menos útil.

Así que hay que aprender a bailar con eso.

Segmentation fault, en particular, es un error en tiempo de ejecución, que indica un acceso inválido a memoria.

Lo mejor que podés hacer cuando tenés un problema de memoria en tu código es usar [`valgrind`](http://valgrind.org/docs/manual/quick-start.html). Es una herramienta con la que ejecutar tu programa, y que lleva una especie de _contabilidad_ de cómo estas usando la memoria. Cuando tu programa usa erróneamente la memoria (sea un error evitable o uno abortivo), `valgrind` te informa en qué línea de código intentaste acceder a qué parte de la memoria, y si puede te dice cerca de qué zonas de memoria válidas fue tu acceso (para, por ejemplo, darte cuenta de que estás tratando de escribir un byte de más porque te faltó reservar el espacio para el `\0` de un string, ponele).

Buscá en [el blog de la materia](http://utn.so/) que en la parte de [recursos](http://www.utn.so/recursos/) hay [un documento que ayuda a entender los reportes de valgrind](http://faq.utn.so/valgrind). El 90% de los problemas los podés atacar con eso.

Además, hay [un vídeo en que explico cómo poder inspeccionar con eclipse un proceso que falló](https://www.youtube.com/watch?v=-CI4MexyU3I), para poder ver el estado de las variables al momento del fallo. Eso ayuda bastante a ver por qué te ocurrió el problema que reporta valgrind cuando no son tan evidentes las causas.

En cuanto a los pseudo ejemplos de código que me pasas, no te olvides que hay una enorme diferencia entre `t_estructura` y `t_estructura *`. Una cosa es la estructura (un conjunto de bytes con los datos que estás modelando) y otra es el puntero (unos pocos bytes con la dirección de memoria en que uno espera encontrar la estructura con los datos del modelo).

No te olvides, tampoco, del scope de las variables en tu programa. Las variables locales, almacenadas en el stack (tal como en el TP), sólo viven hasta que finaliza la ejecución de la función en que son declaradas. Si tu función que crea la estructura lo hace en una variable local, al retornar de esa función la memoria en que se almacenó la estructura deja de estar reservada, y puede ser sobreescrita. Entonces puede que tu programa pinche por eso, también.

Y por esto usamos `malloc()`: reservamos memoria en el heap, que nosotros decidiremos cuándo liberar, pudiendo "sobrevivir" a la función que la reservó.

Como "moraleja" número 2 del mail, recorda lo que dijimos en las charlas sobre usar las herramientas adecuadamente.

No le escapes a `malloc()`. Hay motivos para que exista, y yo te prometo que es desproporcionadamente más fácil aprender a usar `malloc` y gestionar bien la memoria que aprobar un TP de Operativos sin aprenderlo (asumiendo que eso sea posible). Aprovecha, además, que el TP este tiene bastante que ver con la gestión de memoria, y debería aportar en ese sentido.
