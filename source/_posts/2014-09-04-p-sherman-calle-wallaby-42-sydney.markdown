---
layout: post
title: "P. Sherman, Calle Wallaby 42, Sydney"
date: 2014-09-04 00:42
comments: true
categories:
---

{% blockquote %}
Con algo de suerte, este será el primero de una serie de al menos 3 posts hablando sobre la gestión de la memoria en C. Sólo parece no aportar tanto, pero probablemente entre los 3 den un mensaje más o menos copado. Stay tuned :)
{% endblockquote %}
Una de las complejidades más importantes asociadas a la programación en C es la gestión y el uso de la memoria.<!--more-->

Para un programa C, la memoria de la computadora es poco más que un gran vector (_array_) de bytes consecutivos. Todos y cada uno de estos bytes tienen una dirección: un número que permite al programa identificar a cada byte unequívocamente. Las direcciones arrancan (como todo en C) por la dirección 0. Y, como suelen representarse en hexa, vamos a decir que el primer byte de memoria es el de la dirección `0x0`.

Es importante recordar que C trabaja con _punteros_ (referencias a direcciones de memoria) _absolutos_, por lo que, en principio, el byte `0x0` no será necesariamente el primero de los que use nuestro programa para almacenar datos, sino el primero de la memoria de la computadora[^1].

[^1]: En particular, el byte `0x0` suele tener un significado muy especial en la mayoría de los sistemas operativos (`NULL` suele ser una referencia al `0x0`), por lo que en la inmensa mayoría de los casos el byte `0x0` _no va_ a pertenecer a nuestro proceso.

Las direcciones son consecutivas, por lo que el byte siguiente al `0x0` será el `0x1`, `0x2`, y así siguiendo. Hablamos en hexa, así que el décimo byte será el `0x9`, pero el undécimo será `0xa`, y así.

Pero, en general, uno como programador no usa bytes de memoria, sino una abstracción un poquito por encima: las variables.

Al declarar una variable, le estamos indicando al compilador que reserve una determinada cantidad de bytes de memoria, según el tipo declarado, y además le asociamos un nombre a ese conjunto de bytes, de modo que podamos identificarlo y referirlo de modo más simple.

Es importante recalcar que el nombre de la variable sólo existe hasta la compilación: al ejecutar nuestro programa, todas las referencias a memoria se hacen a través de direcciones y tamaños[^2].

[^2]: Cuando indicamos compilar en modo debug, el compilador incluye en el programa final información extra (como los nombres de las variables y funciones) a fin de poder mostrar mensajes de error más apropiados que simplemente muchas direcciones de memoria, pero esta información **NO ES REQUERIDA** para la ejecución del programa.

Entonces, así como la declaración `unsigned int nombre[10];` reserva (en una arquitectura de 32 bits) 40 bytes, la referencia `nombre[0]` habla de la misma dirección que `nombre` (por ejemplo, podría referir a `0x0842f5`), mientras que `nombre[2]` referirá a la dirección 8 bytes más grande, `0x0842fd`. Como el tipo de datos es `unsigned int`, de 32 bits, el compilador sabe cuántos bytes desplazarse para cada elemento (sabiendo, así, que `nombre[2]` está 8 bytes después de `nombre[0]`, y no 4, 2, 16 o cualquier otra cantidad), y cuántos bytes a partir de esa dirección tiene que leer para obtener el dato buscado.
