---
layout: post
title: "Bajando el nivel"
date: 2013-10-21 01:19
comments: true
categories: lo-c-tutto
---

"Che, me acabás de decir que el 0 sigue existiendo por más que _lo pise_ con otro valor. ¿El 0 es un objeto al que la VM le mantiene referencias y por eso no se lo lleva el Garbage Collector?"<!--more-->

What!? No way, papá. Por un cuatrimestre, las palabras "objeto" y "Garbage Collector" dejalas en la oficina, y VM = VirtualBox :)

En C no hay objetos.

Perdón si fui duro, pero es necesario: en C no hay objetos. Y no hay GC.

"Entonces, ¿qué es el 0?"

El 0 es una constante. Si recordás, en Arquitectura (bazinga) vimos que los números se representan en binario en la PC. Las variables de C son meras referencias a algún bloque de memroia, siendo el tipo de la variable el que anuncia qué tamaño tiene esa referencia.

Digamos, esto es una memoria de 30 bytes[^1] (éramos tan pobres):

![Una RAM de 30 bytes]({{ site.url }}/assets/memoria-libre.png)

Esa es toda la memoria de esta computadora hipotética. Si en mi programa declaro una variable `int exit_status;`, podríamos pensarlo como que C hará algo así:

![La RAM con una variable]({{ site.url }}/assets/memoria-reservada.png)

En algún lugar de la memoria (en este caso, a partir del byte 13), C reservó[^2] unos bytes para nuestra variable ¿Cuántos bytes reservó? Eso depende del tipo de dato que le declaramos a la variable. Por definición, el tipo `int` equivale al [tamaño de palabra del procesador](http://es.wikipedia.org/wiki/Palabra_(informática\)). Esto significa que en máquinas de 32 bits (como esta computadora hipotética, o las VMs de la cátedra, o la enorme mayoría de los Windows XP y Vista), un `int` va a ocupar 4 bytes (32 bits).

Entonces, al hacer algo como `exit_status = 0;` (y sabiendo que `exit_status` es una variable entera de 4 bytes), el compilador sabe que tiene que hacer que los 4 bytes que están a partir del byte en que empieza `exit_status` valgan un 0. En Arquitectura aprendimos que los enteros con signo se representan usando el [complemento a 2](http://es.wikipedia.org/wiki/Complemento_a_dos), por lo que el 0 en 32 bits es simplemente [`0x0000 0000`](http://es.wikipedia.org/wiki/Sistema_hexadecimal) (4 bytes en 0).

Entonces, ¿dónde está ese 0? [Hardcodeado](http://es.wikipedia.org/wiki/Hard_code) en el binario. Compilar un programa es pedirle al compilador (`gcc` en nuestro caso) que traduzca todo ese C hermoso que escribimos en las instrucciones de Assembler que nuestro procesador tiene que ejecutar para que el código funcione con nuestro sistema operativo. De todas esas, alguna va a ser algo como `mov eax, 0x00000000`[^3]: **ese** es tu 0.

"¿Y a mí cuál?"

Meh, no mucho. Sólo quería dejar en claro cómo funciona esto: C es una mini abstracción de la programación en Assembler, por lo que no está tan lejos. No tenés que ser un capo de Assembler para programar C[^4] (no necesitás saber Assembler, siquiera), pero tenés que tener un claro entendimiento de cómo funciona la computadora a bajo nivel para entender qué es lo que está haciendo tu código.


[^1]: La memoria de una PC tiene muchísimo más tamaño, pero dejame hacerla dibujable.
[^2]: C es un lenguaje. Decir "C hizo tal cosa" es medio vago: probablemente sea algo que hizo el compilador, o el sistema operativo, o alguna biblioteca más o menos estándar. Cuando decimos "C hizo tal cosa" es porque: a) no nos interesa mucho quién lo hizo (importa que lo hizo _otra persona_, y que es más o menos lo mismo para cualquier programa en C), o b) no sabemos quién lo hizo (y nos da fiaca averiguarlo, de momento).
[^3]: No se Assembler, creo que el parámetro no va en la misma línea que el `mov`, pero anda por ahí cerca.
[^4]: Lo aclaré en la anterior: no se Assembler.
