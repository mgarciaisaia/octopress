---
layout: post
title: "She bangs, she bangs..."
date: 2014-03-20 19:33
comments: true
categories:
---
Si alguna vez editaron un script UNIX, seguramente habrán visto que arrancan de manera similar: `#!/bin/sh`. Por algún motivo, esa línea mágica se llama [Shebang](0)[^1].

{% codeblock mi_script lang:bash %}
#!/bin/sh
echo "¡Hola, mundo!"
{% endcodeblock %}

Llamamos `mi_script` a este archivo.

Esa línea mágica es la responsable de que podamos correr un script ejecutando `./mi_script` en la consola.<!--more--> Como dijimos en [el primer post](tutorial-c/blog/2013/08/19/arrancando), para ejecutar un programa en UNIX, el programa tiene que tener permisos de ejecución[^4], y luego tenemos que tipear su ruta completa en la consola:

{% codeblock lang:bash %}
$ chmod +x mi_script
$ ./mi_script
¡Hola, mundo!
{% endcodeblock %}

Al hacerlo, el sistema operativo carga el programa en memoria y lo ejecuta.

El problema es que un script es texto. Los programas compilados tienen las instrucciones que nuestro procesador entiende, entonces es relativamente trivial ejecutarlo[^2]. Un script tiene texto. Y, adivinaste: el procesador no ejecuta texto. Entonces, ¿qué es lo que ocurre? ¿Por qué y cómo funciona la ejecución de scripts?

La clave es el shebang. Cuando el sistema operativo detecta que lo que pretendemos ejecutar es un script (digamos, no es un programa en algún formato binario que el sistema operativo comprenda), busca en la primer línea para encontrar este shebang. Esa ruta que se encuentra después del `#!` (en nuestro caso anterior, `/bin/sh`) es la que va a ser ejecutada, pasándole como primer parámetro la ruta de nuestro script. Sí: en `/bin/sh` tenés que tener un programa ejecutable para que la ejecución del script funcione. Si, además, querés que se ejecute como corresponde, ese `/bin/sh` debería entender que cuando se lo ejecuta con un parámetro, ese parámetro representa una ruta a un archivo que contiene código en un lenguaje que ese programa sepa interpretar.

En general, esto ocurre: por convención, en los sistemas UNIX `/bin/sh` es una [Bourne Shell](http://es.wikipedia.org/wiki/Bourne_Shell), o alguna otra shell con el modo de compatibilidad activado. ¿Qué es una Bourne Shell? Un programa capaz de interpretar scripts escritos en ese lenguaje. Dado que los programas que se ponen en los shebangs tienen como objetivo _interpretar_ scripts, se los llamó (muy originalmente) _intérpretes_.

¿Existen otros intérpretes? Claro que sí. [Python](https://www.python.org/) es un lenguaje interpretado, y uno puede hacer scripts en Python y ejecutarlos en la consola, como [este](https://github.com/chapuni/llvm-project/blob/bbf3f7262bd18fe1b140ba0db9ada8defa2c839f/klee/scripts/objdump). Lo mismo pasa con Ruby, perl, y tantos otros lenguajes.

Ahora, como somos curiosos, podríamos ponernos a jugar con esto y abusar un poquito. ¿Qué pasa si usamos otro programa como intérprete? ¿Podemos usar cualquier programa? Podemos, sí, pero los resultados van a cambiar.

Editemos nuestro script. Modifiquemos únicamente el shebang, poniéndole como intérprete `/bin/cat`. `cat` es un programa que imprime _en pantalla_ el contenido de la ruta que le pasamos por parámetro. ¿Qué va a pasar cuando volvamos a ejecutar nuestro script?

{% codeblock lang:bash %}
$ ./mi_script
#!/bin/cat
echo "¡Hola, mundo!"
{% endcodeblock %}

Claro que sí. Al ejecutar nuestro script, el sistema operativo ejecutó `/bin/cat` pasándole la ruta a `mi_script` como parámetro. `cat`, como siempre, abrió la ruta que le pasaron por parámetro y mostró su contenido.

Sigamos jugando. Usemos otro programa: `echo`. Cambiemos el intérprete por `/bin/echo`, y ejecutemos:

{% codeblock lang:bash %}
$ ./mi_script
./mi_script
{% endcodeblock %}

Así es. `echo` imprime lo que sea que le pasemos como parámetro. Si lo que le pasamos es la ruta de nuestro script, como pasa con todas las ejecuciones de los scripts, `echo` va a imprimir eso mismo en la consola.

Otro intérprete "loco" que podemos poner es `rm`: `#!/bin/rm`. Ejecutemos:

{% codeblock lang:bash %}
$ ./mi_script
$ ls mi_script
ls: mi_script: No such file or directory
{% endcodeblock %}

Así es, lo borramos.

Entonces, hagamos una última prueba para jugar. Escribamos un programa que liste todos los parámetros con los que fue ejecutado:

{% codeblock interprete.c lang:c %}
#include <stdio.h>

int main(int argc, char **argv) {
  int index;
  for(index = 0; index < argc; index++) {
    printf("  Parametro %d: %s\n", index, argv[index]);
  }
  return 0;
}
{% endcodeblock %}

Compilemos y ejecutemos:

{% codeblock lang:bash %}
$ gcc interprete.c -o interprete
$ ./interprete
  Parametro 0: ./interprete
$ ./interprete primerParametro segundo 3ro
  Parametro 0: ./interprete
  Parametro 1: primerParametro
  Parametro 2: segundo
  Parametro 3: 3ro
{% endcodeblock %}

El primer parámetro de cualquier programa C es el propio programa. Ahora bien, pongámoslo como intérprete de nuestro script:

{% codeblock mi_script lang:bash %}
#!./interprete
echo "Este codigo nunca se ejecutara"
{% endcodeblock %}

Y ejecutemos:

{% codeblock lang:bash %}
$ ./mi_script
  Parametro 0: ./interprete
  Parametro 1: ./mi_script
$ ./mi_script primerParametro
  Parametro 0: ./interprete
  Parametro 1: ./mi_script
  Parametro 2: primerParametro
{% endcodeblock %}

Así es: se ejecuta el programa que pusimos en la shebang, y luego se le pasa todo lo que tipeemos en la consola. Y no sólo podemos poner un programa en el shebang, sino también parámetros:

{% codeblock mi_script lang:bash %}
#!./interprete unParametroFijo
echo "Este codigo nunca se ejecutara"
{% endcodeblock %}

Y ejecutemos:

{% codeblock lang:bash %}
$ ./mi_script
  Parametro 0: ./interprete
  Parametro 1: unParametroFijo
  Parametro 2: ./mi_script
$ ./mi_script otroParametro
  Parametro 0: ./interprete
  Parametro 1: unParametroFijo
  Parametro 2: ./mi_script
  Parametro 3: otroParametro
{% endcodeblock %}

Creo que no queda mucho más para agregar. _Espero que mis respuestas os haya iluminados :)_

<hr>

**EDIT:** Recordé que **sí** había algo más para agregar.

Muchas veces ví que podía escribir un script sin ponerle un shebang, y que de todos modos funcionaba. ¿Por qué? Bueno, porque de algún modo se defaultea. En [Stackoverflow](http://stackoverflow.com/a/9945113/641451) dicen que se ejecuta con la misma shell que estás usando en ese momento, pero esto no es comportamiento estándar: la documentación de la syscall [`execve`](http://linux.die.net/man/2/execve) dice que un script **tiene que** comenzar con un shebang válido[^3]. Conviene no dar nada por sentado y especificar el intérprete que queremos usar, asegurándonos así la compatibilidad y buen funcionamiento.

[0]: http://en.wikipedia.org/wiki/Shebang_(Unix)
[^1]: El caracter `#` en inglés se llama "hash" (y supongo que se lo llamará, también, "sha"), y el `!` es el "bang".
[^2]: _trivial_ es un decir. Ejecutar un programa no tiene nada de trivial, pero digamos que _la idea de cómo ejecutarlo_ es relativamente sencilla, al menos respecto a interpretar un script.
[^3]: Y, abajo de todo, en la documentación, muestra el código de un programa llamado `myecho` _MUUUUUUUY_ similar a nuestro `interprete.c` :)
[^4]: ¡Gracias, Victor, por la corrección!
