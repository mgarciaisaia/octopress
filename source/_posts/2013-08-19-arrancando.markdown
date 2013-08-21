---
layout: post
title: "Arrancando"
date: 2013-08-19 21:59
comments: true
categories: 
author: mgarciaisaia
---

Vamos a hacer un programa en C, por lo que empezamos con una función:

{% codeblock ok.c %}
int main (void) {
	return 0;
}
{% endcodeblock %}

Este es (aproximadamente) el programa más chico que podamos hacer en C. `main()` es la función que se ejecuta al ejecutar un programa C. En este caso, el prototipo de la función es `int main(void)`: nuestro programa no recibirá parámetros (`void`), y devolverá a quién lo ejecute un entero signado (`int`).<!--more-->

Nuestro programa tiene una única instrucción: devolver (`return`) 0, un código de salida que, por convención, indica que el programa ejecutó correctamente.
Compilemos nuestro programa:
{% codeblock lang:bash %}
$ gcc ok.c -o ok
$ ls
ok.c ok
{% endcodeblock %}

`gcc` es el compilador más usado de C. Parte de la GNU Compiler Collection, `gcc` es el compilador específico de C (el proyecto se llamaba GNU C Compiler, pero cambiaron el nombre por soportar también C++, Java y tantos otros lenguajes). Como casi todo comando en Linux, Unix y derivados, podemos leer su manual haciendo `man gcc`.

`gcc` recibe como parámetro (entre tantos otros) el archivo fuente a compilar (`ok.c`), y el parámetro `-o NOMBRE` indica qué nombre queremos darle al binario resultante (`ok`). De no indicarlo, `gcc` elije uno _hermoso_: `a.out`.


Ejecutemos:
{% codeblock lang:bash %}
$ ./ok
$ echo $?
0
{% endcodeblock %}

Hay un 0, así que debemos estar _no-tan-mal_, como mínimo. ¿Qué pasó acá?

En UNIX, la forma de ejecutar un programa es escribiendo como orden la ruta completa al mismo y, luego, separados por espacios, todos sus parámetros. Por ejemplo:

{% codeblock lang:bash %}
$ /bin/ps --version
procps-ng version 3.3.3
{% endcodeblock %}

Ejecutamos el programa `/bin/ps` con el parámetro `--version`. `ps` nos contesta la versión que tenemos instalada. En nuestro caso anterior, ejecutamos `./ok`. `.` y `..` son dos enlaces especiales que hay en todo directorio: `.` enlaza al directorio actual (el propio directorio que contiene a `.`), y `..` enlaza al directorio padre del actual (osea, al directorio que contiene al directorio que contiene a `..`). Entonces, al escribir `.` ya estamos referenciando toda la ruta al directorio actual. Si estamos ubicados en `/home/utnso`, `.` y `/home/utnso` se refieren al mismo directorio. Agregándole `/ok` queda `./ok`, que equivale a `/home/utnso/ok`, la ruta completa a nuestro programa. Hell yeah, ruta completa => ¡ejecutamos el programa!

"Che, pero... ¡No me mostró nada! ¿Dónde está mi 0?"

Bueno, sí, es cierto. No muestra nada porque no le pedimos que muestre nada: nuestro programa sólo devuelve un 0, y nuestra consola sólo ejecuta las instrucciones que le damos. Entonces, pidámosle que muestre el resultado: `echo $?`.

"¡Que te recontra!" Bueno, sí. [El amigo Bourne] (http://en.wikipedia.org/wiki/Stephen_R._Bourne) había faltado a la clase de nombres descriptivos. `echo` es un comando de las consolas que imprime en pantalla[^1] lo que sea que le pasemos por parámetro. Por ejemplo, `echo Hola mundo` imprime `Hola mundo`. `bash` (el lenguaje que interpreta nuestra consola) posee variables, y para dereferenciarlas (leerlas) hay que anteponerle un `$` al nombre de la variable. Por ejemplo (`#` es el caracter de comentario):

{% codeblock lang:bash %}
$ nombre = "Mundo" # asigno "Mundo" a la variable nombre, creandola si no existe
$ echo $nombre # imprimo el contenido de la variable llamada nombre
Mundo
$ echo "Hola $nombre"
Hola Mundo
{% endcodeblock %}

En particular, nosotros le habíamos pedido mostrar una variable: `$?`. `?` es una variable manejada automáticamente por Bash. Cada vez que ejecuta una instrucción, Bash almacena en `?` el código de salida del programa ejecutado. Por eso, al pedirle que imprima la variable `?` (`echo $?`), Bash nos mostró el 0 que nuestro programa había devuelto.

[^1]: En realidad, imprime en la salida estándar, ya lo veremos más adelante. Por ahora, creeme que es la pantalla.
