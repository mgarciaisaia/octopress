---
layout: post
title: "Between a rock and a hard link"
date: 2015-02-07 11:47
comments: true
categories:
---

{% blockquote %}
¿Qué es un hard link? ¿Qué es un soft link? ¿Qué diferencia hay entre ellos?
{% endblockquote %}

Me parece que la clave para entender los enlaces en los filesystems basados en inodos[^1] es entender _qué es un archivo_.<!--more-->

Un archivo está compuesto por dos partes: su contenido (_los bloques de datos_) y su metadata (_el inodo_).

Cuando creo un archivo llamado `miarchivo.txt` cuyo único contenido es `hola` (con, por ejemplo, el comando `echo hola > miarchivo.txt`), en mi FS se reserva un bloque de datos en el que se escriben los 5 bytes `hola\n`[^2], y además se reserva un inodo que se marca como ocupado, en el que se especifica que el contenido del archivo mide 5 bytes, y se establece en el primer puntero del inodo que el primer bloque de datos es el que acabamos de reservar. Se ponen más datos, obviamente, pero por ahora no nos importan.

Y con eso tenemos un archivo, pero nos falta algo importante: referenciarlo. Ese archivo **no tiene nombre**. _El inodo no guarda el nombre del archivo_. Y yo nunca le pido al sistema operativo que me abra el archivo del inodo 5236: yo le pido rutas. Y una ruta está formada por una lista de directorios, y el nombre del archivo al final.

El nombre existe en el directorio.

Un directorio es un archivo cuyo contenido es una tabla. Cada entrada de esa tabla (las _entradas de directorios_) relaciona un nombre de archivo con un número de inodo. Al querer abrir un archivo (digamos, `miarchivo.txt`), el FS busca en esa tabla la entrada con ese nombre, y mira cuál es el número de inodo que le corresponde (digamos, `402`), y hace todas las operaciones usando ese inodo. Siguiendo el ejemplo, si quisieramos leer todo el archivo `miarchivo.txt` (`cat miarchivo.txt`), el FS encuentra la entrada de directorio de ese archivo, lee que le corresponde el inodo 402, abre el inodo 402, mira que mide 5 bytes, busca cuál es el primer bloque de datos, lee los primeros 5 bytes de ese bloque de datos y se los devuelve a `cat`, que se encarga de imprimirlos por pantalla.

{% codeblock lang:bash %}
$ cat miarchivo.txt
hola
$
{% endcodeblock %}

Ahora, ¿qué me impide tener otra entrada de directorio que apunte al inodo 402? En el inodo no hay ninguna referencia del estilo `directorioPadre` ni nada por el estilo. Tranquilamente podría crear una nueva entrada en ese directorio (o en otro, es indistinto[^3]) que apunte al mismo inodo. Asumamos que creamos una entrada para el nombre `passwords.txt`, que también apunta al inodo 402.

¿Qué va a ocurrir cuando quiera leer el archivo `passwords.txt` haciendo `cat passwords.txt`? Esto:

{% blockquote %}el FS encuentra la entrada de directorio de ese archivo, lee que le corresponde el inodo 402, abre el inodo 402, mira que mide 5 bytes, busca cuál es el primer bloque de datos, lee los primeros 5 bytes de ese bloque de datos y se los devuelve a `cat`, que se encarga de imprimirlos por pantalla.{% endblockquote %}

{% codeblock lang:bash %}
$ cat passwords.txt
hola
$
{% endcodeblock %}

Exactamente lo mismo que hace un rato, porque, a partir de que encontró que el inodo es el 402, el resto va todo igual: el inodo es lo que representa a nuestro archivo en el FS.

Y si quisiera agregarle contenido a `miarchivo.txt`, ¿qué pasaría? Ejecuto `echo clave123 >> miarchivo.txt`, y entonces el FS busca la entrada correspondiente a `miarchivo.txt`, encuentra que corresponde al inodo 402, va al inodo 402, ve que mide 5 bytes y que si le agrega los 9 bytes correspondientes a `clave123\n` sigue entrando en el primer bloque de datos, entonces abre el primer bloque de datos, y a partir del sexto byte escribe ese contenido. Por último, anota en el inodo que el archivo ahora pesa 14 bytes, cierra el archivo, y todos felices. Si ahora hacemos `cat miarchivo.txt`, encuentra la entrada de directorio[^4], abre el inodo 402, abre el bloque de datos, lee los 14 bytes, se los devuelve a `cat`, y `cat` los imprime por pantalla.

{% codeblock lang:bash %}
$ cat miarchivo.txt
hola
clave123
$
{% endcodeblock %}

¿Y si ahora leo el contenido de `passwords.txt`? Bueno, _leo la entrada de directorio de `passwords.txt`, veo que es el inodo 402, abro el inodo, veo que mide 14 bytes, abro el primer bloque de datos, leo los 14 bytes, y se los devuelvo a `cat` para que los imprima en pantalla_.

{% codeblock lang:bash %}
$ cat passwords.txt
hola
clave123
$
{% endcodeblock %}

**Las dos entradas de directorio referencian al mismo archivo**. Lo que _escribo en un archivo se ve reflejado en el otro_, pero porque **es mentira que sean dos archivos**: _son el mismo_. Son _dos referencias al mismo objeto_.

TODO: no existe "el archivo" vs "sus hard links": todos son iguales. contador de hard links (reference counting wikipedia), comando para crearlos (`ln miarchivo.txt passwords.txt`), funcionan sólo intra-FS. Ver que las dos entradas apuntan al mismo inodo (`ls -i`, `ls -i1`, `ls -li`)

TODO: soft link: es un archivo cuyo contenido es la ruta de otro archivo, más un flag que indica que es un softlink. Tiene un inodo propio, aparte del del archivo apuntado. Es cross-FS. El apuntado no se entera de que existe el soft. Si borro el archivo original, el link queda _roto_ (esa ruta ya deja de apuntar a un archivo válido). Comando para crearlo (`ln -s`). Leer el target de un softlink (`readlink softlink`).


[^1]: Existen los enlaces en FAT, pero funcionan a otro nivel, me disgustan, me caen mal, y voy a ignorarlos rotundamente :)
[^2]: Por defecto, `echo` imprime un salto de línea, y por eso el `\n` al final del contenido. Se puede evitar el `\n` final haciendo `echo -n`.
[^3]: Sólo importa que sea dentro del mismo filesystem, ya veremos por qué.
[^4]: A esta altura ya te habrás imaginado que tiene bastante sentido aplicar algúna técnica de caché para las entradas de directorios, porque _se usan bastante seguido_, je :)
