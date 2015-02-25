---
layout: post
title: "Aprendiendo C con GDB"
date: 2015-02-24 22:07
comments: true
categories:
---

_Este post es mi traducción del post [Learning C with GDB](https://www.hackerschool.com/blog/5-learning-c-with-gdb) de [Alan O'Donnell](https://github.com/happy4crazy), del blog de [Hacker School](https://www.hackerschool.com/). La traducción fue hecha con el permiso de Hacker School, y obviamente todos los derechos sobre el post original les pertenecen a ellos.
Cualquier error o sugerencia sobre la traducción es más que bienvenida._

Viniendo del mundo de lenguajes de más alto nivel como Ruby, Scheme o Haskell, aprender C puede ser complicado. Además de tener que pelear con las características de bajo nivel de C como el manejo manual de memoria y los punteros, tenés que arreglártelas sin un REPL. Una vez que te acostumbrás a programar explorando en un REPL, tener que lidiar con el ciclo escribir-compilar-correr es bastante un desalentador.

Hace poco se me ocurrió que podría usar `gdb` como un pseudo-REPL para C. Estuve experimentando el uso de `gdb` como una herramienta para aprender C, en lugar de simplemente para debuggear C, y está bastante bueno.

Mi objetivo en este post es mostrarte que `gdb` es una gran herramienta para aprender C. Te voy a mostrar algunos de mis comandos de `gdb` favoritos, y después te voy a mostrar cómo podés usar `gdb` para entender una parte bastante complicada de C: la diferencia entre los arrays y los punteros.<!-- more -->

## Una introducción a GDB

Empecemos creando este pequeño programa C, `minimal.c`:

{% codeblock lang:c minimal.c %}
int main()
{
  int i = 1337;
  return 0;
}
{% endcodeblock %}

Notá que el programa no hace nada, y ni siquiera tiene un `printf`[^sin_printf]. ¡Bienvenido al nuevo mundo de aprender C con `gdb`!

Compilalo con la opción `-g` para que `gdb` tenga información de debug, y después correlo con `gdb`:

{% codeblock lang:bash %}
$ gcc -g minimal.c -o minimal
$ gdb minimal
{% endcodeblock %}

Ahora deberías encontrarte con el prompt bastante austero de `gdb`. Te prometí un REPL, así que acá va:

{% codeblock lang:c-objdump %}
(gdb) print 1 + 2
$1 = 3
{% endcodeblock %}

¡Excelente! `print` es un built-in de `gdb` que imprime el resultado de evaluar una expresión de C. Si en algún momento tenés dudas sobre el funcionamiento de un comando de `gdb`, probá corriendo `help nombre-del-comando`.

Acá hay un ejemplo un poco más interesante:

{% codeblock lang:c-objdump %}
(gbd) print (int) 2147483648
$2 = -2147483648
{% endcodeblock %}

Ignoremos los motivos por los que `2147483648 == -2147483648`. El punto es que `gdb` también entiende la aritmética, que puede ser bastante complicada en C.

Pongamos un breakpoint en la función `main` y arranquemos el programa:

{% codeblock lang:c-objdump %}
(gbd) break main
(gdb) run
{% endcodeblock %}

El programa ahora está pausado en la línea 3, justo antes de inicializar `i`. Resulta interesante que, por más que `i` no haya sido inicializada aún, podamos ver su valor con el comando `print`:

{% codeblock lang:c-objdump %}
(gbd) print i
$3 = 32767
{% endcodeblock %}

En C, el valor de una variable local no inicializada es indefinido, por lo que podrías ver distintos valores cuando lo pruebes vos.

Podemos ejecutar la línea actual usando el comando `next`:

{% codeblock lang:c-objdump %}
(gdb) next
(gdb) print i
$4 = 1337
{% endcodeblock %}

## Examinando la memoria con `x`

Las variables en C son simplemente nombres que se le pone a bloques de memoria. El bloque de memoria de una variable está caracterizada por dos números:

1. La dirección numérica del primer byte del bloque
2. El tamaño del bloque, medido en bytes. El tamaño del bloque de una variable está determinado por el tipo de la variable

Una de las características distintivas de C es que tenés acceso directo al bloque de memoria de las variables. El operador `&` computa la dirección de una variable, y el operador `sizeof` computa el tamaño de esa variable en memoria.

Podemos jugar con estos conceptos en `gdb`:

{% codeblock lang:c-objdump %}
(gdb) print &i
$5 = (int *) 0x7fff5fbff584
(gdb) print sizeof(i)
$6 = 4
{% endcodeblock %}

En palabras, eso dice que el bloque de memoria de `i` empieza en la dirección `0x7fff5fbff584`, y que ocupa 4 bytes de memoria.

Antes mencioné que el tamaño de una variable en memoria es determinado por su tipo, y, de hecho, el operador `sizeof` puede operar directamente sobre tipos:

{% codeblock lang:c-objdump %}
(gdb) print sizeof(int)
$7 = 4
(gdb) print sizeof(double)
$8 = 8
{% endcodeblock %}

Esto significa que, al menos en mi máquina, las variables de tipo `int` ocupan 4 bytes de memoria, y las `double` ocupan 8.

`gdb` incluye una herramienta muy poderosa para examinar la memoria: el comando `x`. `x` examina la memoria, comezando en una dirección en particular. Incluye distintos comandos de formato para tener un control preciso de cuántos bytes querés examinar y cómo querés que se muestren. En caso de dudas, `help x` en el prompt de `gdb`.

El operador `&` computa la dirección de una variable, por lo que podemos pasarle `&i` a `x` y ver qué hay en los bytes que conforman el valor de `i`:

{% codeblock lang:c-objdump %}
(gdb) x/4xb &i
0x7fff5fbff584: 0x39    0x05    0x00    0x00
{% endcodeblock %}

Los flags indican que queremos examinar `4` valores, formateados como números he`x`adecimales, de a un `b`yte a la vez. Elegí examinar 4 bytes porque ese es el tamaño de `i` en memoria - la salida muestra la representación byte a byte de `i` en memoria.

Un detalle a tener en cuenta con las representaciones byte a byte en las máquinas de la familia x86 es que los bytes se almacenan en _little-endian_: contrario a la notación humana, los bytes menos significativos de un número se almacenan primero en memoria.

Una forma de clarificar el tema es darle a `i` un valor más significativo y volver a examinar su bloque de memoria:

{% codeblock lang:c-objdump %}
(gdb) set var i = 0x12345678
(gdb) x/4xb &i
0x7fff5fbff584: 0x78 0x56 0x34 0x12
{% endcodeblock %}

## Examinando tipos con `ptype`

El comando `ptype` puede que sea mi comando favorito. Te dice el tipo de una expresión C:

{% codeblock lang:c-objdump %}
(gdb) ptype i
type = int
(gdb) ptype &i
type = int *
(gdb) ptype main
type = int (void)
{% endcodeblock %}

Los tipos en C pueden volverse [complejos](http://c-faq.com/decl/spiral.anderson.html), pero `ptype` te permite explorarlos interactivamente.

## Punteros y arrays

Los arrays en C son un concepto sumamente delicado. La idea de esta sección es que escribamos un programa simple y nos pongamos a jugar con `gdb` hasta que los arrays empiecen a tener sentido.

Escribí este programa `arrays.c`:

{% codeblock lang:c arrays.c %}
int main()
{
    int a[] = {1,2,3};
    return 0;
}
{% endcodeblock %}

Compilalo con la opción `-g`, correlo con `gdb`, y dale `next` para saltar la línea de inicialización:

{% codeblock lang:c-objdump %}
$ gcc -g arrays.c -o arrays
$ gdb arrays
(gdb) break main
(gdb) run
(gdb) next
{% endcodeblock %}

En este punto ya deberías poder hacer `print` del contenido de `a` y examinar su tipo:

{% codeblock lang:c-objdump %}
(gdb) print a
$1 = {1, 2, 3}
(gdb) ptype a
type = int [3]
{% endcodeblock %}

Ahora que nuestro programa ya inició en `gdb`, lo primero que deberíamos hacer es usar `x` para ver cómo se ve `a` internamente:

{% codeblock lang:c-objdump %}
(gdb) x/12xb &a
0x7fff5fbff56c: 0x01  0x00  0x00  0x00  0x02  0x00  0x00  0x00
0x7fff5fbff574: 0x03  0x00  0x00  0x00
{% endcodeblock %}

Esto significa que el bloque de memoria de `a` comienza en la dirección `0x7fff5fbff56c`. Los primeros 4 bytes almacenan `a[0]`, los 4 siguientes almacenan `a[1]`, y los últimos 4 almacenan `a[2]`. De hecho, podés ver que `sizeof` sabe que el tamaño de `a` en memoria son 12 bytes:

{% codeblock lang:c-objdump %}
(gdb) print sizeof(a)
$2 = 12
{% endcodeblock %}

Hasta acá, los arrays parecen comportarse bastante como arrays. Tienen su propio tipo como arrays, y almacenan sus componentes en bloques de memoria contiguos. De todos modos, en algunas situaciones los arrays se comportan _bastante_ como punteros. Por ejemplo, podemos hacer aritmética de punteros con `a`:

{% codeblock lang:c-objdump %}
(gdb) print a + 1
$3 = (int *) 0x7fff5fbff570
{% endcodeblock %}

En palabras, esto dice que `a + 1` es un puntero a un `int`, y ubicado en la dirección `0x7fff5fbff570`. A esta altura, pasarle los punteros que veas a `x` debería serte un acto reflejo, así que probemos:

{% codeblock lang:c-objdump %}
(gdb) x/4xb a + 1
0x7fff5fbff570: 0x02  0x00  0x00  0x00
{% endcodeblock %}

Notá que `0x7fff5fbff570` es 4 más que `0x7fff5fbff56c`, la dirección del primer byte de memoria de `a`. Dado que los valores `int` ocupan 4 bytes, esto significa que `a + 1` apunta a `a[1]`.

De hecho, los subíndices de los arrays de C son _syntactic sugar_ (agregados sintátcitos) para hacer aritmética de punteros: `a[i]` es equivalente a `*(a + i)`. Podés probarlo en `gdb`:

{% codeblock lang:c-objdump %}
(gdb) print a[0]
$4 = 1
(gdb) print *(a + 0)
$5 = 1
(gdb) print a[1]
$6 = 2
(gdb) print *(a + 1)
$7 = 2
(gdb) print a[2]
$8 = 3
(gdb) print *(a + 2)
$9 = 3
{% endcodeblock %}

Vimos que, en algunas situaciones, `a` se comporta como un array, meintras que en otras actúa como un puntero a su primer elemento. ¿Qué está pasando?

La respuesta es que cuando se usa el nombre de un array en una expresión C, _degenera_[^degenera_decays] en un puntero al primer elemento del array. Existen sólo dos excepciones a esta regla: cuando el nombre del array se le pasa a `sizeof`, y cuando se le pasa al operador `&`[^excepciones_decay].

El hecho de que `a` no degenere en un puntero cuando se lo pasamos al operador `&` dispara una pregunta interesante: ¿hay alguna diferencia entre el puntero al que `a` degenera y `&a`?

Numéricamente hablando, representan la misma dirección:

{% codeblock lang:c-objdump %}
(gdb) x/4xb a
0x7fff5fbff56c: 0x01  0x00  0x00  0x00
(gdb) x/4xb &a
0x7fff5fbff56c: 0x01  0x00  0x00  0x00
{% endcodeblock %}

Así y todo, sus tipos son diferentes. Ya vimos que el valor al que `a` degenera es un puntero al primer elemento de `a`, por lo que su tipo debe ser `int *`. En cuanto al tipo de `&a`, preguntémosle a `gdb`:

{% codeblock lang:c-objdump %}
(gdb) ptype &a
type = int (*)[3]
{% endcodeblock %}

En palabras, `&a` es un puntero a un array de tres `int`s. Tiene sentido: `a` no degenera al ser pasado a `&`, y el tipo de `a` es `int[3]`.

Podés observar la diferencia entre el valor al que degenera `a` y `&a` probando cómo se comportan con la aritmética de punteros:

{% codeblock lang:c-objdump %}
(gdb) print a + 1
$10 = (int *) 0x7fff5fbff570
(gdb) print &a + 1
$11 = (int (*)[3]) 0x7fff5fbff578
{% endcodeblock %}

Notá que sumarle 1 a `a` le suma 4 a su dirección, mientras que sumarle 1 a `&a` le agrega ¡12!

El puntero al que `a` realmente degenera es `&a[0]`:

{% codeblock lang:c-objdump %}
(gdb) print &a[0]
$11 = (int *) 0x7fff5fbff56c
{% endcodeblock %}

## Conclusión

Con un poquito de suerte, ya te convencí de que `gdb` es un lindo ambiente de exploración para aprender C. Podés imprimir el resultado de evaluar expresiones con `print`, e`x`aminar los bytes de memoria en crudo, y jugar con el sistema de tipos usando `ptype`.

Si querés experimentar un poco más con `gdb` para aprender C, estas son algunas sugerencias:

* Usá `gdb` para resolver el [desafío Ksplice de punteros](https://blogs.oracle.com/ksplice/entry/the_ksplice_pointer_challenge)
* Investigá cómo se almacenan los `struct`s en memoria. ¿Cómo se comparan con los arrays?
* ¡Usá el comando `dissasemble` de `gdb` para aprender assembler! Un ejercicio particularmente entretenido es investigar cómo funciona el stack de llamadas.[^post_gdb_assembler]
* Probá el modo `tui` de `gdb`, que ofrece una interfaz a `gdb` hecha en ncurses[^gdb_tui].

_Alan es facilitador en Hacker School. Agradece a [David Albert](http://dave.is/), [Tom Ballinger](https://github.com/thomasballinger), [Nicholas Bergson-Shilcock](http://nick.is/) y [Amy Dyer](https://github.com/oxling) por sus muy útiles comentarios._

_¿Te intriga Hacker School? Leé [sobre ello](https://hackerschool.com/about), mirá [lo que dicen sus alumnos](https://hackerschool.com/testimonials), y [aplicá](https://hackerschool.com/apply) para la próxima camada._

[^sin_printf]: Dependiendo de cuán agresivo sea tu compilador a la hora de optimizar código inútil, podrías tener que hacer que efectivamente haga algo :) Yo probé estos ejemplos en mi Raspberry Pi y funcionó bien.
[^degenera_decays]: **NdT**: el artículo original dice que _it “decays” to a pointer to the array’s first element_. No conocía esa expresión, y no encuentro a mano algún material que diga _decay_ y del que conozca su traducción, para ver cuál es el término usado en español (si es que hubiera uno consensuado). Elegí _degenera_ porque se usa en algunas temáticas relacionadas, y porque me pareció que expresa un poco la intención original. Si así no lo hiciese, que `$DEITY` y la patria me lo demanden, o que alguien me deje un comentario con el término apropiado. Y así fue como, hace mil años, algún español nos pegó el karma de "instancia" en lugar de "individuo, especimen". Soy un criminal, ¡oh yeah!
[^excepciones_decay]: Formalmente hablando, el nombre de un array es un "non-modifiable lvalue" (_valor izquierdo no modificable_). Cuando es usado en una expresión que requiere un rvalue (_valor derecho_), el nombre del array degenera en un puntero a su primer elemento. En cuanto a las excepciones, el operador `&` requiere un lvalue, y `sizeof` simplemente es raro.
[^post_gdb_assembler]: **NdT**: Podés leer este post relacionado: [Aprendiendo Assembler Para Entender C](/blog/2015/02/19/aprendiendo-assembler-para-entender-c/)
[^gdb_tui]: **NdT**: podés leerte la [Guía rápida de Beej para GDB](http://beej.us/guide/bggdb/), que es corta, bastante buena, y usa todo el tiempo el modo `tui`.
