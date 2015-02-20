---
layout: post
title: "Aprendiendo Assembler para entender C"
date: 2015-02-19 19:10
comments: true
categories:
---

_Este post es mi traducción del post [Understanding C by learning assembly](https://www.hackerschool.com/blog/7-understanding-c-by-learning-assembly) de [David Albert](https://github.com/davidbalbert), del blog de [Hacker School](https://www.hackerschool.com/). La traducción fue hecha con el permiso de Hacker School, y obviamente todos los derechos sobre el post original les pertenecen a ellos.
Cualquier error o sugerencia sobre la traducción es más que bienvenida._

La última vez, [Alan](https://github.com/happy4crazy) mostró cómo usar [GDB como una herramienta para aprender C](https://www.hackerschool.com/blog/5-learning-c-with-gdb). Hoy quiero ir un paso más allá y usar GDB para ayudarnos a entender assembler, también.

Las capas de abstracción son grandes herramientas para construir cosas, pero a veces pueden interponerse en el aprendizaje. Mi objetivo en este post es convencerte de que para aprender C rigurosamente, también necesitamos entender el código assembler que nuestro compilador de C genera. Voy a hacer esto mostrándote cómo desensamblar y leer un programa simple con GDB, y después vamos a usar GDB y nuestro conocimiento de assembler para entender cómo funcionan las variables locales estáticas en C.

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

_(continuará...)_

[^compila_mac]: Lo de que es en OS X no es porque me haga el careta usando Mac, si no porque el post original estaba así, y todavía no tengo la sopa/tiempo/ganas de hacer todo lo mismo pero en Linux con `gcc` :)
[^using_make]: Tal vez notes que usamos Make para buildear `simple.c` sin un makefile. Podemos hacerlo porque Make tiene reglas implícitas para buildear ejecutables a partir de archivos C. Podés encontrar más información sobre esas reglas en el [manual de Make](http://www.gnu.org/software/make/manual/make.html#Implicit-Rules)
[^sintaxis_assembler]: También podés hacer que GDB muestre la sintaxis de Intel, usada por NASM, MASM y otros assemblers, pero eso se va del alcance del post.
[^humanamente_legible]: Con _humanamente legible_ quiere decir _**Joaco**-humanamente legible_, pero bue...
[^ancho_registros]: Los procesadores con sets de instrucciones SIMD (como MMX y SSE para x86, y AltiVec para PowerPC) suelen tener algunos registros que son más anchos que la arquitectura de la CPU.
