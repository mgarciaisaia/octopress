---
layout: post
title: "Aprendiendo Assembler para entender C"
date: 2015-02-19 19:10
comments: true
categories:
---

_Este post es mi traducción del post [Understanding C by learning assembly](https://www.hackerschool.com/blog/7-understanding-c-by-learning-assembly) de [David Albert](https://github.com/davidbalbert), del blog de [Hacker School](https://www.hackerschool.com/). La traducción fue hecha con el permiso de Hacker School, y obviamente todos los derechos sobre el post original les pertenecen a ellos.
Cualquier error o sugerencia sobre la traducción es más que bienvenida._

La última vez, [Alan](https://github.com/happy4crazy) mostró cómo usar [GDB como una herramienta para aprender C](/blog/2015/02/24/aprendiendo-c-con-gdb/)[^gdb-post-ingles]. Hoy quiero ir un paso más allá y usar GDB para ayudarnos a entender assembler, también.

Las capas de abstracción son grandes herramientas para construir cosas, pero a veces pueden interponerse en el aprendizaje. Mi objetivo en este post es convencerte de que para aprender C rigurosamente, también necesitamos entender el código assembler que nuestro compilador de C genera. Voy a hacer esto mostrándote cómo desensamblar y leer un programa simple con GDB, y después vamos a usar GDB y nuestro conocimiento de assembler para entender cómo funcionan las variables locales estáticas en C.<!--more-->

_Nota: todo el código en este post fue compilado en una CPU x86_64 corriendo Mac OS X 10.8.1, usando Clang 4.0 con las optimizaciones deshabilitadas (`-O0`)[^compila_mac]._

## Aprendiendo assembler con GDB

Empecemos por desensamblar un programa con GDB y aprender a leer su salida. Tipeá el siguiente programa en un archivo de texto y grabalo como `simple.c`:

{% codeblock lang:c sample.c %}
int main()
{
  int a = 5;
  int b = a + 6;
  return 0;
}
{% endcodeblock %}

Ahora compilalo con los símbolos de depuración y sin optimizaciones, y corré GDB[^using_make]:

{% codeblock lang:bash %}
$ CFLAGS="-g -O0" make simple
cc -g -O0    simple.c   -o simple
$ gdb simple
{% endcodeblock %}

Dentro de GDB, pongamos un breakpoint en `main` y ejecutemos hasta llegar al `return`. Ponemos el número 2 después de `next` para especificar que queremos ejecutar el comando `next` dos veces:


{% codeblock lang:c-objdump %}
(gdb) break main
(gdb) run
(gdb) next 2
{% endcodeblock %}

Ahora usemos el comando `disassemble` para mostrar las instrucciones assembler de la función actual. También podés pasarle el nombre de una función a `disassemble` para examinar una función diferente.

{% codeblock lang:c-objdump %}
(gdb) disassemble
Dump of assembler code for function main:
0x0000000100000f50 <main+0>:    push   %rbp
0x0000000100000f51 <main+1>:    mov    %rsp,%rbp
0x0000000100000f54 <main+4>:    mov    $0x0,%eax
0x0000000100000f59 <main+9>:    movl   $0x0,-0x4(%rbp)
0x0000000100000f60 <main+16>:   movl   $0x5,-0x8(%rbp)
0x0000000100000f67 <main+23>:   mov    -0x8(%rbp),%ecx
0x0000000100000f6a <main+26>:   add    $0x6,%ecx
0x0000000100000f70 <main+32>:   mov    %ecx,-0xc(%rbp)
0x0000000100000f73 <main+35>:   pop    %rbp
0x0000000100000f74 <main+36>:   retq
End of assembler dump.
{% endcodeblock %}

El comando `disassemble` defaultea a mostrar las instrucciones en la sintaxis de AT&T, que es la misma sintaxis usada por el assembler de GNU[^sintaxis_assembler]. Las instrucciones en la sintaxis AT&T son del formato `mnemónico origen, destino`. El mnemónico es el nombre _humanamente legible_[^humanamente_legible] de la instrucción. `origen` y `destino` son operandos, que pueden ser valores inmediatos, registros, direcciones de memoria o etiquetas. Los valores inmediatos son constantes, y están prefijados por un `$`. Por ejemplo, `$0x5` representa al número 5 en hexadecimal. Los nombres de registros están prefijados por un `%`.

### Registros

Vale la pena dedicarle un ratito a entender los registros. Los registros son porciones de almacenamiento de datos que se encuentran directamente en el CPU. Salvo algunas excepciones, el tamaño, o _ancho_, de los registros de una CPU definen su arquitectura. Entonces, si tenés una CPU de 64 bits, tus registros van a medir 64 bits. Lo mismo ocurre con las CPUs de 32 bits (registros de 32 bits), 16 bits, y así[^ancho_registros]. Los registros son de muy rápido acceso, y en general son los operandos de las operaciones aritméticas y lógicas.

La familia x86 tiene registros de propósito general y de propósito específico. Los de propósito general pueden usarse para cualquier operación, y su valor no tiene un significado especial para la CPU. En cambio, la CPU depende de ciertos registros de propósito específico para su propia operación, y los valores que en ellos se almacenan tienen un significado especial según el registro de que se trate. En nuestro ejemplo anterior, `%eax` y `%ecx` son registros de propósito general, mientras que `%rbp` y `%rsp` son de propósito específico. `%rbp` es el _base pointer_ (puntero base), que apunta a la base del marco actual dentro del stack (_current stack frame_), y `%rsp` es el _stack pointer_ (puntero al stack), que apunta al tope del stack frame actual. `%rbp` siempre tiene un valor mayor a `%rsp` porque el stack comienza en una dirección de memoria alta y crece hacia abajo. Si no te es muy familiar el concepto de stack de llamadas, podés encontrar [una buena introducción en Wikipedia](http://en.wikipedia.org/wiki/Call_stack) (y [acá en español](http://es.wikipedia.org/wiki/Pila_de_llamadas)).

Un detalle de la familia x86 es que mantuvo compatibilidad hacia atrás desde el procesador 8086, de 16 bits. A medida que el x86 avanzó de 16 bits a 32, y de 32 a 64, los registros se expandieron y fueron tomando nuevos nombres, de modo de no romper la compatibilidad con código escrito para CPUs anteriores, menos anchos.

Tomemos de ejemplo al registro `AX` de propósito general, de 16 bits. El byte _alto_ puede accederse con el nombre `AH` (_A High_), y el bajo con `AL` (_A Low_). Cuando salió el 80386, de 32 bits, el registro AX Extendido (_Extended AX_, `EAX`) refirió al registro de 32 bits, mientras que `AX` se siguió refiriendo al registro formado por los 16 bits de la mitad baja del `EAX`. Del mismo modo, al salir la arquitectura x86_64, se usó el prefijo `R`, y el `EAX` pasó a ser la mitad baja del registro `RAX`, de 64 bits. Este es un diagrama basado en un [artículo de Wikipedia](http://en.wikipedia.org/wiki/X86#x86_registers) para visualizar estas relaciones:

{% codeblock Subdivisiones de los registros en x86_64 %}
|__64__|__56__|__48__|__40__|__32__|__24__|__16__|__8___|
|__________________________RAX__________________________|
|xxxxxxxxxxxxxxxxxxxxxxxxxxx|____________EAX____________|
|xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx|_____AX______|
|xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx|__AH__|__AL__|
{% endcodeblock %}

### Volviendo al código

Ya deberíamos tener información suficiente para entender nuestro programa desensamblado:

{% codeblock lang:c-objdump %}
0x0000000100000f50 <main+0>:    push   %rbp
0x0000000100000f51 <main+1>:    mov    %rsp,%rbp
{% endcodeblock %}

Las primeras dos instrucciones se llaman el _prólogo_ o _preámbulo_ de una función. Primero `push`eamos el antiguo base pointer al stack para guardarlo para más tarde. Después copiamos el valor del stack pointer al base pointer. Después de esto, `%rbp` apunta a la base del stack frame de `main`.

{% codeblock lang:c-objdump %}
0x0000000100000f54 <main+4>:    mov    $0x0,%eax
{% endcodeblock %}

Esta instrucción copia un 0 a `%eax`. La convención de llamadas (_calling convention_) del x86 dice que el valor de retorno de una función debe almacenarse en `%eax`, de modo que esta instrucción prepara el `return 0` al final de nuestra función.

{% codeblock lang:c-objdump %}
0x0000000100000f59 <main+9>:    movl   $0x0,-0x4(%rbp)
{% endcodeblock %}

Acá vemos algo que no habíamos visto antes: `-0x4(%rbp)`. Los paréntesis nos indican que eso es una dirección de memoria. En este caso, `%rbp` es el registro base, y `-0x4` es el desplazamiento. Esto es equivalente a `%rbp + -0x4`. Como el stack crece hacia abajo, restarle 4 a la base del stack frame actual nos mueve dentro del frame mismo, donde se almacenan las variables locales. Esto significa que esta instrucción almacena un 0 en `%rbp - 4`. Me llevó un rato entender para qué era esta línea, pero parece que clang [reserva una variable local oculta](http://lists.cs.uiuc.edu/pipermail/cfe-dev/2012-February/019767.html) para un valor de retorno implícito de `main`.

También habrás notado que el mnemónico tiene el sufijo `l`. Esto significa que los operandos van a ser de tipo `l`ong (32 bits, para los enteros). Otros sufijos válidos son `b`yte, `s`hort, `q`uad y `t`en. Si ves una instrucción que no tiene un sufijo, el tamaño de los operandos se infiere de los tamaños de los registros origen y destino. Por ejemplo, en la línea anterior, `%eax` mide 32 bits, por lo que la instrucción `mov` se infiere a `movl`.

{% codeblock lang:c-objdump %}
0x0000000100000f60 <main+16>:   movl   $0x5,-0x8(%rbp)
{% endcodeblock %}

¡Ahora estamos llegando a la papa de nuestro programa! Esta primer línea de assembler es la primera línea de C en `main`, y almacena el número 5 en el espacio de la próxima variable local (`%rbp - 0x8`), 4 bytes debajo de nuestra última variable local. Esa es la posición de `a`. Podemos usar GDB para verificarlo:

{% codeblock lang:c-objdump %}
(gdb) x &a
0x7fff5fbff768: 0x00000005
(gdb) x $rbp - 8
0x7fff5fbff768: 0x00000005
{% endcodeblock %}

Fijate que las direcciones de memoria son las mismas. Vas a notar que GDB tiene variables que representan nuestros registros, pero como todas las variables en GDB, las prefijamos con `$` en vez del `%` usado en el assembler de AT&T.

{% codeblock lang:c-objdump %}
0x0000000100000f67 <main+23>:   mov    -0x8(%rbp),%ecx
0x0000000100000f6a <main+26>:   add    $0x6,%ecx
0x0000000100000f70 <main+32>:   mov    %ecx,-0xc(%rbp)
{% endcodeblock %}

Entonces movemos (`mov`) `a` a `%ecx`, uno de nuestros registros de propósito general, le sumamos (`add`) 6, y almacenamos el resultado en `%rbp - 0xc`. Esta es la segunda línea de C en `main`. Quizá te diste cuenta que `%rbp - 0xc` es `b`, cosa que podemos verificar en GDB:

{% codeblock lang:c-objdump %}
(gdb) x &b
0x7fff5fbff764: 0x0000000b
(gdb) x $rbp - 0xc
0x7fff5fbff764: 0x0000000b
{% endcodeblock %}

El resto de `main` es simplemente limpieza, llamado el _epílogo_ de la función:

{% codeblock lang:c-objdump %}
0x0000000100000f73 <main+35>:   pop    %rbp
0x0000000100000f74 <main+36>:   retq
{% endcodeblock %}

`pop`eamos el viejo base pointer del stack y lo devolvemos al `%rbp`, y luego `retq` nos devuelve a nuestra dirección de retorno, que también está almacenada en el stack frame.

Hasta ahora, usamos GDB para desensamblar un pequeño programa C, le pegamos un vistazo a cómo leer la sintaxis de assembler de AT&T, y hablamos un poco de los operandos de registros y direcciones de memoria. También usamos GDB para verificar dónde se almacenan nuestras variables locales en relación a `%rbp`. Ahora vamos a ver cómo usar nuestras nuevas habilidades para explicar cómo funcionan las variables locales estáticas.

## Entendiendo las variables estáticas locales

Las variables estáticas locales son una característica muy copada de C. En dos palabras, son variables locales que se inicializan una única vez y persisten sus valores a través de las sucesivas llamadas a la función en que fueron definidas. Un caso de uso bastante simple de las variables estáticas locales es un generador _à la_ Python. Este ejemplo genera todos los números naturales hasta `INT_MAX`:

{% codeblock lang:c static.c %}
/* static.c */
#include <stdio.h>
int natural_generator()
{
  int a = 1;
  static int b = -1;
  b += 1;
  return a + b;
}

int main()
{
  printf("%d\n", natural_generator());
  printf("%d\n", natural_generator());
  printf("%d\n", natural_generator());

  return 0;
}
{% endcodeblock %}

Cuando lo compilamos y ejecutamos, este programa imprime los primeros tres números naturales:

{% codeblock lang:bash %}
$ CFLAGS="-g -O0" make static
cc -g -O0    static.c   -o static
$ ./static
1
2
3
{% endcodeblock %}

Pero, ¿cómo funciona esto? Para entender las estáticas locales, vamos a meternos en GDB y mirar el assembler. Saqué las direcciones que muestra GDB al desensamblado para que entre en la pantalla:

{% codeblock lang:c-objdump %}
$ gdb static
(gdb) break natural_generator
(gdb) run
(gdb) disassemble
Dump of assembler code for function natural_generator:
push   %rbp
mov    %rsp,%rbp
movl   $0x1,-0x4(%rbp)
mov    0x177(%rip),%eax        # 0x100001018 <natural_generator.b>
add    $0x1,%eax
mov    %eax,0x16c(%rip)        # 0x100001018 <natural_generator.b>
mov    -0x4(%rbp),%eax
add    0x163(%rip),%eax        # 0x100001018 <natural_generator.b>
pop    %rbp
retq
End of assembler dump.
{% endcodeblock %}

Lo primero que necesitamos descubrir es en qué instrucción estamos. Podemos hacer eso examinando el "instruction pointer" (_puntero de instrucción_) o "program counter" (_contador de programa_). El instruction pointer es un registro que almacena la dirección de memoria de la próxima instrucción. En x86_64, ese registro es `%rip`. Podemos acceder al instruction pointer usando la variable `$rip`, o podemos usar `$pc`, la alternativa independiente de la plataforma:

{% codeblock lang:c-objdump %}
(gdb) x/i $pc
0x100000e94 <natural_generator+4>:  movl   $0x1,-0x4(%rbp)
{% endcodeblock %}

El instruction pointer siempre contiene la dirección de la _próxima_ instrucción a ejecutar, lo que significa que la tercer instrucción aún no se ejecutó, pero está a punto.

Como es bastante útil conocer la próxima instrucción, vamos a hacer que GDB nos muestre la próxima instrucción cada vez que frenamos el programa. A partir de GDB 7.0, podés ejecutar `set disassemble-next-line on`, que muestra todas las instrucciones que conforman la próxima instrucción fuente, pero estamos usando Mac OS X, que trae GDB 6.3, así que vamos a tener que recurrir al comando `display`. `display` es como `x`, sólo que evalúa la expresión cada vez que nuestro programa se detiene:

{% codeblock lang:c-objdump %}
(gdb) display/i $pc
1: x/i $pc  0x100000e94 <natural_generator+4>:  movl   $0x1,-0x4(%rbp)
{% endcodeblock %}

Ahora GDB está listo para mostrarnos siempre la próxima instrucción a ejecutar antes del prompt.

Ya pasamos el prólogo de la función, del que hablamos antes, así que vamos a empezar en la tercer instrucción. Esto corresponde a la primer línea de código que asigna 1 a `a`. En lugar de `next`, que nos mueve a la próxima instrucción fuente, vamos a usar `nexti`, que nos mueve a la próxima instrucción assembler. Después, vamos a examinar `%rbp - 0x4` para verificar nuestra hipótesis de que `a` se almacena en `%rbp - 0x4`.

{% codeblock lang:c-objdump %}
(gdb) nexti
7           b += 1;
1: x/i $pc  mov   0x177(%rip),%eax # 0x100001018 <natural_generator.b>
(gdb) x $rbp - 0x4
0x7fff5fbff78c: 0x00000001
(gdb) x &a
0x7fff5fbff78c: 0x00000001
{% endcodeblock %}

Son lo mismo, tal como esperábamos. La próxima instrucción es más interesante:

{% codeblock lang:c-objdump %}
mov    0x177(%rip),%eax        # 0x100001018 <natural_generator.b>
{% endcodeblock %}

Acá es donde esperaríamos encontrar la línea `static int b = -1;`, pero se ve substancialmente distinta de cualquier otra cosa que hayamos visto antes. Por empezar, no hay referencia al stack frame, donde esperaríamos encontrar las variables locales. ¡Ni siquiera hay un `-0x1`! En cambio, tenemos una instrucción que carga `0x100001018`, ubicada en algún lugar después del instruction pointer, a `%eax`. GDB nos da un comentario bastante útil con el resultado del cálculo del operando en memoria, y nos da una pista diciéndonos que `natural_generator.b` se encuentra en esa dirección. Corramos esta instrucción y veamos qué pasa:

{% codeblock lang:c-objdump %}
(gdb) nexti
(gdb) p $rax
$3 = 4294967295
(gdb) p/x $rax
$5 = 0xffffffff
{% endcodeblock %}

Por más que el desensamblado muestre `%eax` como destino, nosotros imprimimos `$rax`, porque GDB sólo nos provee variables para los registros completos.

En este momento necesitamos recordar que, mientras que las variables tienen tipos que especifican si son signadas o no, los registros no, por lo que GDB imprime el valor de `%rax` sin signo. Probemos de nuevo, casteando `%rax` a un `int` signado:

{% codeblock lang:c-objdump %}
(gdb) p (int)$rax
$11 = -1
{% endcodeblock %}

Parece que encontramos a `b`. Podemos re-chequear esto usando el comando `x`:

{% codeblock lang:c-objdump %}
(gdb) x/d 0x100001018
0x100001018 <natural_generator.b>:  -1
(gdb) x/d &b
0x100001018 <natural_generator.b>:  -1
{% endcodeblock %}

Entonces, no es sólo que `b` está en una dirección de memoria baja, fuera del stack, si no que además se la inicializa a -1 antes de que se llame a `natural_generator`. De hecho, incluso si desensamblaras el programa completo, no encontrarías ningún código que setteara `b` a -1. Esto ocurre porque el valor de `b` se hardcodea en otra sección del ejecutable `sample`, y el loader del sistema operativo lo carga en memoria junto con todo el código de máquina cuando se lanza el proceso[^process_launch].

Después de todo esto, las cosas empiezan a tener más sentido. Tras almacenar `b` en `%eax`, pasamos a la próxima instrucción fuente, en la que incrementamos `b`. Esto se corresponde a las próximas dos instrucciones:

{% codeblock lang:c-objdump %}
add    $0x1,%eax
mov    %eax,0x16c(%rip)        # 0x100001018 <natural_generator.b>
{% endcodeblock %}

Acá le agregamos (`add`) 1 a `%eax`, y almacenamos el resultado en la memoria. Corramos estas instrucciones y verifiquemos el resultado:

{% codeblock lang:c-objdump %}
(gdb) nexti 2
(gdb) x/d &b
0x100001018 <natural_generator.b>:  0
(gdb) p (int)$rax
$15 = 0
{% endcodeblock %}

Las próximas dos instrucciones nos preparan para devolver `a + b`:

{% codeblock lang:c-objdump %}
mov    -0x4(%rbp),%eax
add    0x163(%rip),%eax        # 0x100001018 <natural_generator.b>
{% endcodeblock %}

Acá cargamos `a` en `%eax`, y luego le sumamos `b`. En este punto, esperaríamos que `%eax` valga 1. Verifiquemos:

{% codeblock lang:c-objdump %}
(gdb) nexti 2
(gdb) p $rax
$16 = 1
{% endcodeblock %}

Se usa `%eax` para almacenar el valor de retorno de `natural_generator`, así que estamos listos para el epílogo, que limpia el stack y retorna:

{% codeblock lang:c-objdump %}
pop    %rbp
retq
{% endcodeblock %}

Ahora que entendemos cómo se inicializa `b`, veamos qué pasa cuando corremos `natural_generator` otra vez:

{% codeblock lang:c-objdump %}
(gdb) continue
Continuing.
1

Breakpoint 1, natural_generator () at static.c:5
5           int a = 1;
1: x/i $pc  0x100000e94 <natural_generator+4>:  movl   $0x1,-0x4(%rbp)
(gdb) x &b
0x100001018 <natural_generator.b>:  0
{% endcodeblock %}

Como `b` no está almacenada en el stack junto con las otras variables locales, al volver a llamar a `natural_generator` sigue valiendo 0. No importa cuántas veces llamemos a nuestro generador, `b` siempre va a retener su valor. Esto ocurre porque está almacenada fuera del stack, y es inicializada cuando el loader mueve el programa a memoria en lugar de por nuestor código máquina.

## Conclusión

Comenzamos viendo cómo leer assembler y cómo desensamblar un programa con GDB. Luego, vimos cómo funcionan las variables estáticas locales, cosa que no podríamos haber hecho sin desensamblar nuestro ejecutable.

Pasamos mucho tiempo alternando entre leer instrucciones assembler y verificando nuestras hipótesis en GDB. Puede parecer repetitivo, pero hay una razón muy importante para hacerlo así: la mejor manera de aprender algo abstracto es volverlo más concreto, y una de las mejores maneras de hacer más concreto a algo es usando herramientas que te dejen sacarle capas de abstracción. La mejor manera de aprender estas herramientas es forzándote a usarlas hasta que te sean naturales.

[^gdb-post-ingles]: El post original, en inglés, está [en HackerSchool](https://www.hackerschool.com/blog/5-learning-c-with-gdb)
[^compila_mac]: **NdeT**: Lo de que es en OS X no es porque me haga el careta usando Mac, si no porque el post original estaba así, y todavía no tengo la sopa/tiempo/ganas de hacer todo lo mismo pero en Linux con `gcc` :)
[^using_make]: Tal vez notes que usamos Make para buildear `simple.c` sin un makefile. Podemos hacerlo porque Make tiene reglas implícitas para buildear ejecutables a partir de archivos C. Podés encontrar más información sobre esas reglas en el [manual de Make](http://www.gnu.org/software/make/manual/make.html#Implicit-Rules)
[^sintaxis_assembler]: También podés hacer que GDB muestre la sintaxis de Intel, usada por NASM, MASM y otros assemblers, pero eso se va del alcance del post.
[^humanamente_legible]: **NdeT**: Con _humanamente legible_ quiere decir _**Joaco**-humanamente legible_, pero bue...
[^ancho_registros]: Los procesadores con sets de instrucciones SIMD (como MMX y SSE para x86, y AltiVec para PowerPC) suelen tener algunos registros que son más anchos que la arquitectura de la CPU.
[^process_launch]: Reservamos para un post futuro la discusión sobre formatos de objeto, loaders y linkers.
