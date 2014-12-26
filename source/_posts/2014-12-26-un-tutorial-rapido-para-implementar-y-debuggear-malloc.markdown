---
layout: post
title: "Un tutorial rápido para implementar y debuggear malloc, free, calloc y realloc"
date: 2014-12-26 00:29
comments: true
categories:
---

_Este post es mi traducción del post [A Quick Tutorial on Implementing and Debugging Malloc, Free, Calloc, and Realloc](http://danluu.com/malloc-tutorial/) de [Dan Luu](https://twitter.com/danluu). Cualquier error o sugerencia sobre la traducción es bienvenida, y si querés que haga de intermediario para sugerirle cambios de contenido a él, también vale._


¡Implementemos nuestro propio [`malloc`](http://man7.org/linux/man-pages/man3/malloc.3.html) y veamos cómo funciona con programas pre-existentes!

Este tutorial asume que sabés qué es un puntero, y que `*ptr` dereferencia un puntero, y que `ptr->foo` significa `(*ptr).foo`, que `malloc` se usa para [reservar (_allocar_) espacio dinámicamente](http://duartes.org/gustavo/blog/post/how-the-kernel-manages-your-memory/), y que te es familiar el concepto de lista enlazada. Si decidís seguir este tutorial sin tener un claro conocimiento de C, contame qué partes convendría explicar un poco más. Si querés mirar todo el código de una, está disponible [acá](https://github.com/danluu/malloc-tutorial/blob/master/malloc.c).<!--more-->

Presentaciones aparte, la firma de `malloc` es:

{% codeblock lang:c %}
void *malloc(size_t size);
{% endcodeblock %}

Recibe un número de bytes y devuelve un puntero a un bloque de memoria de ese tamaño.

Hay varias formas de implementar esto. Nosotros vamos a elegir arbitrariamente utilizar la llamada al sistema [`sbrk`](http://man7.org/linux/man-pages/man2/sbrk.2.html). El sistema operativo reserva espacios de stack (_pila_) y heap (_montículo_) para los procesos, y `sbrk` nos permite manipular el heap. `sbrk(0)` devuelve un puntero al tope del heap. `sbrk(foo)` incrementa el tamaño del heap en `foo` bytes y devuelve un puntero al tope previo.

![Diagrama del esquema de memoria de Linux, cortesía de Gustavo Duarte](http://danluu.com/images/malloc-tutorial/heap.png)

Si queremos implementar un `malloc` realmente simple, podemos hacer algo como:

{% codeblock lang:c %}
#include <assert.h>
#include <string.h>
#include <sys/types.h>
#include <unistd.h>

void *malloc(size_t size) {
  void *p = sbrk(0);
  void *request = sbrk(size);
  if (request == (void*) -1) {
    return NULL; // sbrk failed.
  } else {
    assert(p == request); // Not thread safe.
    return p;
  }
}
{% endcodeblock %}

Cuando un programa le pide espacio a `malloc`, éste se lo pide a `sbrk` para incrementar el tamaño del heap, y devuelve un puntero al inicio de la nueva región en el heap. Esta implementación falla en un tecnisismo, dado que `malloc(0)` debería devolver `NULL` u otro puntero que se le pueda pasar a `free` sin romper todo, pero básicamente funciona.

Pero, hablando de `free`... ¿Cómo funciona? Su prototipo es

{% codeblock lang:c %}
void free(void *ptr);
{% endcodeblock %}

Cuando le pasan a `free` un puntero que había sido devuelto por `malloc` debe liberar ese espacio. Pero si nos dan un puntero a algo reservado por nuestro `malloc`, no tenemos idea de cuál es el tamaño de ese bloque. ¿Dónde almacenamos eso? Si tuviéramos un `malloc` funcionando, podríamos `malloc`ear algo de espacio y guardarlo ahí, pero vamos a meternos en muchos problemas si necesitamos llamar a `malloc` para reservar espacio cada vez que llamamos a `malloc` para reservar espacio :)

Un truco común para evitar este problema es guardar meta-información sobre la región de memoria en el espacio previo al puntero que devolvemos. Supongamos que el tope del heap actual está en `0x1000` y pedimos `0x400` bytes. Nuestro `malloc` actual va a pedir `0x400` bytes a `sbrk` y devolver un puntero a `0x1000`. Si, en cambio, guardáramos, digamos, `0x10` bytes para almacenar información sobre el propio bloque, nuestro `malloc` pediría `0x410` bytes a `sbrk` y devolvería un puntero a `0x1010`, escondiendo nuestro bloque de `0x10` bytes de meta-información del código que está llamando a `malloc`.

Eso nos permite liberar un bloque, pero... ¿y ahora? Esa región de heap que nos da el SO tiene que ser contigua, entonces no podemos devolverle al SO un pedacito de memoria que esté en el medio. Incluso si quisiéramos copiar todos los datos que están después de la región liberada hacia adelante para rellenar el hueco, cosa de poder liberar el pedazo final del heap, no tenemos forma de actualizar todos los punteros que haya hacia esa zona de memoria a lo largo de todo nuestro programa.

En cambio, podemos marcar que esos bloques fueron liberados sin devolvérselos al SO, y en invocaciones futuras de `malloc` podríamos reusar esos bloques. Pero para hacer eso necesitamos poder acceder a la meta-información de cada bloque. Hay varias formas de hacerlo, nosotros vamos a elegir arbitrariamente hacerlo con una lista simplemente enlazada por cuestión de simplicidad.

Entonces, para cada bloque, queremos tener algo como:

{% codeblock lang:c %}
struct block_meta {
  size_t size;
  struct block_meta *next;
  int free;
  int magic; // For debugging only. TODO: remove this in non-debug mode.
};

#define META_SIZE sizeof(struct block_meta)
{% endcodeblock %}

Necesitamos saber el tamaño del bloque, si está o no libre, y cuál es el bloque siguiente. Hay también un número mágico ahí (el campo `magic`) para ayudarnos a debuggear, pero no es realmente necesario. Le vamos a asignar valores arbitrarios que nos van a permitir saber qué parte del código fue la última en modificar esa estructura.

También necesitamos un inicio para nuestra lista enlazada:

{% codeblock lang:c %}
void *global_base = NULL;
{% endcodeblock %}

Para nuestro `malloc`, queremos reusar el espacio libre si fuera posible, _alloc_ando espacio sólo cuando no podemos reusar el que ya tenemos. Dado que tenemos esta estructura de lista enlazada, buscar un bloque libre y devolverlo es casi trivial. Cuando tenemos un pedido de algún tamaño, iteramos nuestra lista buscando algún bloque libre con tamaño suficiente.

{% codeblock lang:c %}
struct block_meta *find_free_block(struct block_meta **last, size_t size) {
  struct block_meta *current = global_base;
  while (current && !(current->free && current->size >= size)) {
    *last = current;
    current = current->next;
  }
  return current;
}
{% endcodeblock %}

Si no encontramos un bloque que nos sirva, le tenemos que pedir ese espacio al SO usando `sbrk` y lo agregamos al final de nuestra lista.

{% codeblock lang:c %}
struct block_meta *request_space(struct block_meta* last, size_t size) {
  struct block_meta *block;
  block = sbrk(0);
  void *request = sbrk(size + META_SIZE);
  assert((void*)block == request); // Not thread safe.
  if (request == (void*) -1) {
    return NULL; // sbrk failed.
  }

  if (last) { // NULL on first request.
    last->next = block;
  }
  block->size = size;
  block->next = NULL;
  block->free = 0;
  block->magic = 0x12345678;
  return block;
}
{% endcodeblock%}

Como en nuestra implementación original, pedimos espacio usando `sbrk`, pero ahora le pedimos un poquito más de espacio para poder almacenar nuestra estructura, y además inicializamos los campos correctamente.

Ahora que tenemos las funciones auxiliares para ver si tenemos espacio libre y para pedirlo, nuestro `malloc` es simple. Si nuestro puntero base global es `NULL`, tenemos que pedir espacio y apuntar nuestro puntero base a ese nuevo bloque. Si no es `NULL`, buscamos si podemos reusar el espacio existente. Si podemos, lo hacemos, y si no pedimos espacio nuevo y lo usamos.

{% codeblock lang:c %}
void *malloc(size_t size) {
  struct block_meta *block;
  // TODO: align size?

  if (size <= 0) {
    return NULL;
  }

  if (!global_base) { // First call.
    block = request_space(NULL, size);
    if (!block) {
      return NULL;
    }
    global_base = block;
  } else {
    struct block_meta *last = global_base;
    block = find_free_block(&last, size);
    if (!block) { // Failed to find free block.
      block = request_space(last, size);
      if (!block) {
        return NULL;
      }
    } else {      // Found free block
      // TODO: consider splitting block here.
      block->free = 0;
      block->magic = 0x77777777;
    }
  }

  return(block+1);
}
{% endcodeblock %}

Para quienes no se lleven mucho con C, devolvemos `block+1` porque queremos devolver un puntero a la región siguiente a `block_meta`. Dado que `block` es un puntero del tipo `struct block_meta`, hacerle `+1` incrementa la dirección en `sizeof(struct block_meta)` bytes.

Si sólo quisiéramos un `malloc` sin un `free`, podríamos haber usado nuestro `malloc` original, mucho más simple. Entonces, ¡escribamos nuestro `free`! Lo principal que `free` tiene que hacer es setear el campo `->free`.

Como vamos a necesitar obtener la dirección de nuestro struct en varios lugares de nuestro código, definamos una función para ello:

{% codeblock lang:c %}
struct block_meta *get_block_ptr(void *ptr) {
  return (struct block_meta*)ptr - 1;
}
{% endcodeblock%}

Ahora que lo tenemos, este es `free`:

{% codeblock lang:c %}
void free(void *ptr) {
  if (!ptr) {
    return;
  }

  // TODO: consider merging blocks once splitting blocks is implemented.
  struct block_meta* block_ptr = get_block_ptr(ptr);
  assert(block_ptr->free == 0);
  assert(block_ptr->magic == 0x77777777 || block_ptr->magic == 0x12345678);
  block_ptr->free = 1;
  block_ptr->magic = 0x55555555;
}
{% endcodeblock %}

Además de asignar `->free`, es válido llamar a `free` con un puntero `NULL`, y por eso chequeamos por `NULL`. Como `free` no debería llamarse con cualquier dirección arbitraria o bloques que ya hayan sido liberados, `assert`eamos que esas cosas no pasen.

En realidad no es necesario `assert`ear nada, pero suele ayudar a debuggear. De hecho, cuando escribí este código tuve un bug que hubiera resultado en corromper silenciosamente los datos si los asserts no hubieran estado ahí. En cambio, el código falló por los `assert`s, y debuggear el problema se volvió trivial.

Ahora que tenemos `malloc` y `free`, ¡podemos escribir programas usando nuestro propio gestor de memoria! Pero antes de meter nuestro gestor en programas ya existentes, necesitamos implementar un par de funciones comunes más: `realloc` y `calloc`. `calloc` es simplemente un `malloc` que inicializa la memoria en 0, así que arranquemos por `realloc`. `realloc` debería ajustar el tamaño de un bloque de memoria que obtuvimos a través de `malloc`, `calloc` o `realloc`.

El prototipo de `realloc` es:

{% codeblock lang:c %}
void *realloc(void *ptr, size_t size)
{% endcodeblock %}

Si le pasamos un puntero a `NULL`, debería funcionar exactamente igual que `malloc`. Si le pasamos un puntero previamente `malloc`eado, debería liberar espacio si el tamaño es menor al previo, y asignar más espacio y copiar los datos existentes si el tamaño es mayor que el anterior.

Los programas pueden seguir funcionando si no hiciéramos lo de achicar el bloque o liberarlo, pero sí necesitamos reservar más espacio cuando el tamaño se incrementa, así que empecemos con eso:

{% codeblock lang:c %}
void *realloc(void *ptr, size_t size) {
  if (!ptr) {
    // NULL ptr. realloc should act like malloc.
    return malloc(size);
  }

  struct block_meta* block_ptr = get_block_ptr(ptr);
  if (block_ptr->size >= size) {
    // We have enough space. Could free some once we implement split.
    return ptr;
  }

  // Need to really realloc. Malloc new space and free old space.
  // Then copy old data to new space.
  void *new_ptr;
  new_ptr = malloc(size);
  if (!new_ptr) {
    return NULL; // TODO: set errno on failure.
  }
  memcpy(new_ptr, ptr, block_ptr->size);
  free(ptr);
  return new_ptr;
}
{% endcodeblock %}

Y ahora `calloc`, que simplemente inicializa la memoria antes de devolver el puntero:

{% codeblock lang:c %}
void *calloc(size_t nelem, size_t elsize) {
  size_t size = nelem * elsize; // TODO: check for overflow.
  void *ptr = malloc(size);
  memset(ptr, 0, size);
  return ptr;
}
{% endcodeblock %}

Notemos que no chequea overflows en `nelem * elsize`, requerido por la especificación. Todo este código es simplemente lo mínimo necesario para que las cosas más o menos funcionen.

Y ahora que tenemos algo que más o menos funciona, usémoslo con los programas ya existentes (¡sin siquiera recompilarlos!).

Primero que nada, necesitamos compilar nuestro código. En Linux, algo como:

{% codeblock lang:bash %}
clang -O0 -g -W -Wall -Wextra -shared -fPIC malloc.c -o malloc.so
{% endcodeblock %}

debería funcionar.

`-g` agrega los símbolos de debug, para poder mirar nuestro código con `gdb` o `lldb`. `-O0` va a ayudar a debuggear, evitando que en las optimizaciones se eliminen variables. `-W -Wall -Wextra` agrega warnings adicionales. `-shared -fPIC` nos permite linkear nuestro código dinámicamente, que es lo que nos permite [usar nuestro código con binarios ya existentes](http://jvns.ca/blog/2014/11/27/ld-preload-is-super-fun-and-easy/).

En Mac usaríamos algo así:

{% codeblock lang:bash %}
clang -O0 -g -W -Wall -Wextra -dynamiclib malloc.c -o malloc.dylib
{% endcodeblock %}

Notar que `sbrk` fue deprecado en las versiones recientes de OS X. Apple usa una definición poco ortodoxa de _deprecado_ - algunas llamadas al sistema deprecadas directamente fallan. No probé esto en una Mac, así que puede que la implementación tenga fallas, o directamente no funcione en Mac.

Ahora, para que un binario use nuestro `malloc` en Linux, tenemos que setear la variable de entorno `LD_PRELOAD`. Si estás usando `bash`, podés hacerlo así:

{% codeblock lang:bash %}
export LD_PRELOAD=/path/absoluto/a/malloc.so
{% endcodeblock %}

Si estás en Mac, el comando es:

{% codeblock lang:bash %}
export DYLD_INSERT_LIBRARIES=/path/absoluto/a/malloc.so
{% endcodeblock %}

Si todo funciona, podés correr cualquier binario arbitrario y debería funcionar normalmente (excepto que será un poquito más lento).

{% codeblock lang:bash %}
$ ls
Makefile  malloc.c  malloc.so  README.md  test  test-0  test-1  test-2  test-3  test-4
{% endcodeblock %}

Si hubiera un bug, vas a ver algo así:

{% codeblock lang:bash %}
$ ls
Segmentation fault (core dumped)
{% endcodeblock %}

### Debuggeando

¡Hablemos de debugear! Si te es familiar el uso del debugger para poner breakpoints, inspeccionar la memoria y ejecutar paso a paso el código, podés saltear esta sección e ir directamente a [los ejercicios](#ejercicios).

Esta sección asume que podés instalar `gdb` en tu sistema. Si estás en Mac, probablemente quieras usar `lldb` y traducir los comandos apropiadamente. Como no tengo idea de qué bugs podés encontrarte, voy a introducir algunos bugs en el código para mostrarte cómo encontrarlos y resolverlos.

Primero, necesito poder correr `gdb` sin que genere un segmentation fault. Si `ls` _segfaultea_ y tratamos de correr `gdb ls`, muy probablemente `gdb` vaya a segfaultear también. Podríamos escribir un wrapper para hacer esto, pero `gdb` también lo soporta. Si iniciamos `gdb` y después corremos `set environment LD_PRELOAD=./malloc.so` antes de correr el programa, `LD_PRELOAD` va a funcionar normalmente.

{% codeblock lang:bash %}
$ gdb /bin/ls
(gdb) set environment LD_PRELOAD=./malloc.so
(gdb) run
Program received signal SIGSEGV, Segmentation fault.
0x00007ffff7bd7dbd in free (ptr=0x0) at malloc.c:113
113       assert(block_ptr->free == 0);
{% endcodeblock %}

Como esperábamos, tenemos un segfault. Podemos usar `list` para ver el código alrededor del error:

{% codeblock lang:bash %}
(gdb) list
108     }
109
110     void free(void *ptr) {
111       // TODO: consider merging blocks once splitting blocks is implemented.
112       struct block_meta* block_ptr = get_block_ptr(ptr);
113       assert(block_ptr->free == 0);
114       assert(block_ptr->magic == 0x77777777 || block_ptr->magic == 0x12345678);
115       block_ptr->free = 1;
116       block_ptr->magic = 0x55555555;  
117     }
{% endcodeblock %}

Y podemos usar `p` (_print_) para ver qué está pasando con las variables:

{% codeblock lang:bash %}
(gdb) p ptr
$6 = (void *) 0x0
(gdb) p block_ptr
$7 = (struct block_meta *) 0xffffffffffffffe8
{% endcodeblock %}

`ptr` vale `0`, o sea, `NULL`, y esa es la causa del problema: nos olvidamos de chequear por `NULL`.

Ahora que encontramos eso, probemos con un bug un poco más complicado. Digamos que decidimos reemplazar nuestra estructura por:

{% codeblock lang:c %}
struct block_meta {
  size_t size;
  struct block_meta *next;
  int free;
  int magic;    // For debugging only. TODO: remove this in non-debug mode.
  char data[1];
};
{% endcodeblock %}

Y devolver `block->data` en lugar de `block+1` en `malloc`, sin más cambios. Se parece bastante a lo que veníamos haciendo, sólo que ahora definimos un campo más que apunta al final de la estructura, y retornamos un puntero ahí.

Pero esto es lo que pasa si usamos nuestro nuevo `malloc`:

{% codeblock lang:bash %}
$ /bin/ls
Segmentation fault (core dumped)
$ gdb /bin/ls
(gdb) set environment LD_PRELOAD=./malloc.so
(gdb) run

Program received signal SIGSEGV, Segmentation fault.
_IO_vfprintf_internal (s=s@entry=0x7fffff7ff5f0, format=format@entry=0x7ffff7567370 "%s%s%s:%u: %s%sAssertion     `%s' failed.\n%n", ap=ap@entry=0x7fffff7ff718) at vfprintf.c:1332
1332    vfprintf.c: No such file or directory.
1327    in vfprintf.c
{% endcodeblock %}

Este no es tan lindo como el error anterior - podemos ver que uno de nuestros `assert`s falló, pero `gdb` nos deja tirados dentro de una función de `printf`, llamada cuando un `assert` falla. Pero esa función usa nuestro `malloc` buggeado ¡y revienta!

Lo que podemos hacer es inspeccionar `ap` para ver qué era lo que `assert` trataba de imprimir:

{% codeblock lang:bash %}
(gdb) p *ap
$4 = {gp_offset = 16, fp_offset = 48, overflow_arg_area = 0x7fffff7ff7f0, reg_save_area = 0x7fffff7ff730}
{% endcodeblock %}

Eso funcionaría bien. Podemos jugar un poquito hasta encontrar qué es lo que pretendía imprimir, y encontrar el error de ese modo. Otras formas hubieran sido implementar nuestro propio `assert` o usar los _hooks_ correctos para prevenir que `assert` use nuestro `malloc`.

Pero, en este caso, nosotros sabemos que hay sólo unos pocos `assert`s en nuestro código: el de `malloc`, garantizando que no estemos usandolo en un programa multihilo, y los dos de `free` chequeando que no estemos liberando algo que no debiéramos. Miremos primero `free`, poniendo un _breakpoint_:

{% codeblock lang:bash %}
$ gdb /bin/ls
(gdb) set environment LD_PRELOAD=./malloc.so
(gdb) break free
Breakpoint 1 at 0x400530
(gdb) run /bin/ls

Breakpoint 1, free (ptr=0x61c270) at malloc.c:112
112       if (!ptr) {
{% endcodeblock %}

Aún no asignamos `block_ptr`, pero si hacemos `s` un par de veces para avanzar hasta después de que fuera asignado, podemos ver que su valor es:

{% codeblock lang:bash %}
(gdb) s
(gdb) s
(gdb) s
free (ptr=0x61c270) at malloc.c:118
118       assert(block_ptr->free == 0);
(gdb) p/x *block_ptr
$11 = {size = 0, next = 0x78, free = 0, magic = 0, data = ""}
{% endcodeblock %}

Estoy usando `p/x` en vez de `p` para poder verlo en hexadecimal. El campo `magic` está en 0, que debería ser imposible para una estructura válida para liberar. ¿Puede que `get_block_ptr` esté devolviendo un offset incorrecto? Tenemos `ptr` disponible, así que podemos inspeccionar distintos offsets. Dado que es un `void *`, vamos a tener que castearlo para que `gdb` sepa cómo interpretar los resultados:

{% codeblock lang:bash %}
(gdb) p sizeof(struct block_meta)
$12 = 32
(gdb) p/x *(struct block_meta*)(ptr-32)
$13 = {size = 0x0, next = 0x78, free = 0x0, magic = 0x0, data = {0x0}}
(gdb) p/x *(struct block_meta*)(ptr-28)
$14 = {size = 0x7800000000, next = 0x0, free = 0x0, magic = 0x0, data = {0x78}}
(gdb) p/x *(struct block_meta*)(ptr-24)
$15 = {size = 0x78, next = 0x0, free = 0x0, magic = 0x12345678, data = {0x6e}}
{% endcodeblock %}

Si miramos un poco la dirección que estamos usando, podemos ver que el offset correcto es 24 y no 32. Lo que está pasando es que la estructura tiene _padding_, es decir, está siendo alineada al tamaño de palabra del procesador. Entonces, `sizeof(struct block_meta)` es 32, incluso aunque el último campo válido esté en 24. Si queremos remover ese espacio extra, tenemos que arreglar `get_block_ptr`.

¡Y eso fue todo el debugging!

### <a name="ejercicios">Ejercicios</a>


Personalmente, estas cosas no me quedan hasta que no hago un par de ejercicios, así que voy a dejar algunos acá para quienes les interese.

1. Se espera que `malloc` devuelva un puntero _convenientemente alineado a cualquier tipo nativo_. ¿Hace eso el nuestro? Si lo hace, ¿por qué? Si no, corregilo. Notá que "cualquier tipo nativo" es, básicamente, 8 bytes en C, dado que los tipos de SSE/AVX no son nativos.
2. Nuestro `malloc` desperdicia un montón de espacio si reusamos un bloque sin necesitar _tanto_ tamaño. Implementá una función que lo divida en bloques que ocupen el espacio mínimo necesario.
3. Después de hacer `2`, si llamamos a `malloc` y `free` muchas veces con tamaños aleatorios, terminaremos con un montón de bloques pequeños que sólo se pueden reusar si pedimos cantidades pequeñas de memoria. Implementá un mecanismo que una bloques libres adyacentes para que varios bloques consecutivos se unan en uno solo.
4. ¡Encontrá bugs en el código existente! No lo testeé demasiado, así que _tienen que_ haber bugs, por más que esto más o menos funcione.

## Parte 2 en adelante

A continuación vamos a ver cómo hacer que esto sea más rápido y que sea _thread safe_.

### Recursos

Leí [este tutorial](http://www.inf.udec.cl/~leo/Malloc_tutorial.pdf) de Marwan Burelle antes de sentarme a escribir mi propia implementación, así que se parece bastante. Las principales diferencias son que mi versión es más simple, pero más vulnerable a la fragmentación de memoria. En términos de exposición, mi estilo es bastante más informal. Si querés algo más formal, el Dr. Burelle es tu camino a seguir.

Para saber más sobre cómo Linux maneja la memoria, mirá [este post](http://duartes.org/gustavo/blog/post/anatomy-of-a-program-in-memory/) de Gustavo Duarte.

Para saber más sobre cómo funcionan las implementaciones reales de `malloc`, [dlmalloc](http://g.oswego.edu/dl/html/malloc.html) y [tcmalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html) son dos grandes lecturas. No leí el código de [jemalloc](http://www.canonware.com/jemalloc/), y escuché que es un poco más difícil de entender, pero es una de las implementaciónes de [malloc] de alta performance más usadas que hay.

Para ayuda debuggeando, [Address Sanitizer](https://code.google.com/p/address-sanitizer/wiki/AddressSanitizer) la rompe. Y si querés escribir una versión thread-safe, [Thread Sanitizer](https://code.google.com/p/data-race-test/wiki/ThreadSanitizer) también es una gran herramienta.

### Agradecimientos

Gracias a Gustavo Duarte por permitirme usar una de sus imágenes para ilustrar `sbrk`, a Ian Whitlock y Danielle Sucher por encontrar algunos _typos_, a Nathan Kurz por sus sugerencias sobre los recursos adicionales, y a "tedu" por encontrar un bug. Por favor, [avisame](https://twitter.com/danluu) si encontrás otros bugs en este post (tanto en el texto como en el código).
