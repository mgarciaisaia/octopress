---
layout: post
title: "Variables"
date: 2013-08-20 01:21
comments: true
categories: 
author: mgarciaisaia
---

Bien. Escribimos, compilamos y corrimos nuestro primer programa. Pero es como bastante aburrido, ¿no? Vamos a ponerle onda: declaremos una variable (¡iupi! (¿?))

{% codeblock ok.c (v2) %}
int main(void) {
	int exit_status = 0;
	return exit_status;
}
{% endcodeblock %}

Guau. Me la jugué :) Anoche no dormí porque me quedé debuggeando un error en este programa.<!--more-->

¿Qué cambió? Bueno, entre las llaves hay dos instrucciones ahora. En principio, donde antes decía `return 0;`, ahora dice `return exit_status;`.

"Así que seguramente `exit_status` sea una variable mágica de C, como `$?` en bash" Pendorcho. `exit_status` es una variable, sí. Osea, es un identificador de un _cacho'e memoria_. Puedo guardar _cosas_ ahí, y luego leerlas. Pero antes necesito declararla, para decir: a) que existe; y b) qué tipo de cosas va a manejar esa variable. Y eso es lo que hicimos antes: `int exit_status = 0;`.

Para declarar una variable, especificamos su tipo de dato, seguido por su nombre. En nuestro caso, `int exit_status` crea una variable de tipo `int` llamada `exit_status`. Declaraciones válidas son `int hola;` o `int hola, chau;`, por ejemplo: la primera declara una variable `hola` de tipo `int`, mientras que la segunda crea `hola` y `chau`, dos variables de tipo `int`, ambas totalmente independientes entre sí.

Pero con declarar la variable no alcanza: si queremos devolverla o leerla, primero tenemos que darle un valor ("inicializarla", para los amigos). En C, las asignaciones son del estilo `variable = expresion;`, donde, en nuestro caso original, `expresion` es un triste `0` constante. Y ahí tenemos nuestra primera línea: `int exit_status = 0;`.

"Che, y, entonces, si no es una variable mágica de C, ¿por qué se llama `exit_status` y no, por ejemplo, `a`, `bleh` o `code`?" Bueno, porque nosotros **sí** fuimos a la clase de nombres bonitos y representativos :) Y si esa variable representa nuestro estado de salida, así la llamaremos[^1]. Podríamos haberla llamado `a`, `bleh`, `code` o `__a256723b`, pero preferimos reservarnos los nombres horribles para las PPT :)

Buen, a ver qué hace este programa:

{% codeblock lang:bash %}
$ gcc ok.c -o ok
$ ./ok
$ echo $?
0
{% endcodeblock %}

Compilamos y ejecutamos, y vemos que sigue sin mostrar nada. Hacemos el `echo` y vemos nuestro hermoso 0.

"Che, para mí que éste nos está chamuyando y el 0 está hardcodeado por ahí"

OK, cambiémoslo:

{% codeblock ok.c (v2.1) lang:c %}
int main(void) {
	int exit_status = 1;
	return exit_status;
}
{% endcodeblock %}

Y probemos:

{% codeblock lang:bash %}
$ ./ok
$ echo $?
0
{% endcodeblock %}

"¡Ajá! ¡Te dije que nos mentía!"

¡Momento, cerebrito! No recompilaste, chámpion ;-)

{% codeblock lang:bash %}
$ gcc ok.c -o ok
$ ./ok
$ echo $?
1
{% endcodeblock %}

_¡Touché!_

"OK, ganaste. Ahora, si necesito recompilarpara cambiar el valor de una variable, muy variable no me parece. Y podría cambiar el 0 por un 1 en la primer versión de `ok.c`, y no tengo que andar haciendo tanta parafernalia. ¿Por qué se llaman _variables_?"

Buen, sí, justamente, porque podés cambiarles el valor durante una misma ejecución del programa. Así como inicializamos `exit_status` en 0 o en 1, podríamos después de esa incialización _asignarle_ un nuevo valor. Desde que se ejecute esa instrucción en adelante, cada vez que se lea el contenido de la variable obtendremos el nuevo valor, como si nunca hubiera tenido un valor distinto:

{% codeblock ok.c (v2.2) lang:c %}
int main(void) {
	int exit_status = 0;
	exit_status = 1;
	return exit_status;
}
{% endcodeblock %}

{% codeblock lang:bash %}
$ gcc ok.c -o ok
$ ./ok
$ echo $?
1
{% endcodeblock %}

`=` es el operador de asignación. El resultado de evaluar lo que esté a su derecha (ya veremos alternativas, pero por ahora quedémonos con que los números evalúan a sí mismos) se almacena en el espacio de memoria referido a la izquierda.
Y, ¿qué pasó con el 0? Se perdió. El 0 sigue existiendo y valiendo 0, como siempre. Sólo que el contenido de la variable `exit_status` se sobreescribe con `1`: la asignación es _destructiva_.

Sigamos jugando con esto:

{% codeblock ok.c (v2.3) lang:c %}
int main(void) {
	int exit_status = 0;
	int a_number = 1;
	exit_status = a_number;
	a_number = 3;
	return exit_status;
}
{% endcodeblock %}

{% codeblock lang:bash %}
$ gcc ok.c -o ok
$ ./ok
$ echo $?
1
{% endcodeblock %}

"¡Eh! ¡¿Qué onda?! Si `exit_status` es igual a `a_number`, y a `a_number` le asigno `3`, ¿por qué el estado de salida es `1`?"

Bueno, porque te olvidaste lo que dije de la asignación: en lo que está a la izquierda del `=` guardo el resultado de evaluar lo que está a _la otra izquierda_ del mismo (comunmente conocida como "derecha"). Y nada más que eso: las variables no se ligan, ni quedan relacionadas, ni nada. Las variables se evalúan a su contenido del momento en que se ejecuta la instrucción, por lo que al hacer `exit_status = a_number;` estamos diciendo "en `exit_status` guardame lo que `a_number` valga en ese momento". Como `a_number` venía valiendo 1, `exit_status` pasa a valer 1 también. Que después modifiquemos `a_number` es otra canción, y no tiene ninguna relación con esa asignación que ya se hizo: lo hecho, hecho está, y si al evaluar la variable ésta valía 1, los posibles valores que tenga después no importan, porque ya se realizó la asignación.

[^1]: Hay miles de debates sobre escribir código en inglés vs español. No prometo ser constante durante la guía, pero tiendo a codificar en inglés. Comercialmente hablando, ~todos deseamos que nos compre Google, tercerizar el desarrollo a India, o pedir ayuda con los problemas en [StackOverflow](http://stackoverflow.com/). Pero, si escribir en inglés hará que no entiendas lo que representa cada cosa (cargándote la clase de nombres descriptivos), dale con el español. El piso mínimo que deberías tener es que todo el equipo (sea del TP, o de un trabajo _de posta_) use la misma convención. Y atenti con el tema de caracteres acentuados y eso, que a ¡¡50!! años de inventarse [el código ASCII](http://es.wikipedia.org/wiki/ASCII) seguimos encontrándonos caracteres mochos por ahí, y, en el peor de los casos, puede hacernos fallar la compilción por problemas de encoding. Nadie quiere eso.
