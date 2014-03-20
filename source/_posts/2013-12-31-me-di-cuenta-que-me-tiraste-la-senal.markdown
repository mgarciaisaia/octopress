---
layout: post
title: "Me dí cuenta que me tiraste la señal"
date: 2013-12-31 00:27
comments: true
categories: 
---
Uno de los mecanismos de <abbr title="Inter-Process Communication - Comunicación entre procesos">IPC</abbr> que nos provee Linux es el envío de señales. Es un modo de comunicación MUY minimalista: un proceso le envía una determinada señal a otro, y este último simplemente recibe el código asociado a la señal recibida.

Y nada más. No hay parámetros, no hay información del emisor, nada. Sólo un código.<!--more-->

Y, aún así, es un método más que útil: para terminar un proceso en la terminal (cuando apretamos <kbd>CTRL+C</kbd>), para cerrar un programa que se colgó (desde algún administrador de tareas), para pausar la ejecución de un proceso o para cerrar un programa clickeando la `x` de cierre en su ventana - en todos estos casos estamos mandando una señal a un proceso.

Las señales tienen asociadas distintas acciones por defecto, que se ejecutarán cuando el proceso reciba cada señal. Por ejemplo, al recibir `SIGTERM` o `SIGKILL`[^1], el proceso finalizará con un código de salida no-exitoso, mientras que la señal `SIGUSR1` será ignorada (el proceso no hará nada al recibirla).

Lo interesante de las señales es que los procesos pueden especificar sus propias acciones a ejecutar cuando la señal es recibida, en lugar de ejecutar la acción por defecto. Dado que las señales se envían a los procesos[^2], alguno de sus hilos se interrumpirá para pasar a ejecutar una función que nosotros determinemos cuando la señal llegue, tal como si en la línea de código que está ejecutándose al momento de recibir la señal invocara a la función determinada. Al concluir la ejecución de la función, el programa retornará al punto en que estaba antes del arribo de la señal, y todo seguirá su curso normalmente.

Por ser un mecanismo de IPC (es decir, involucra a más de un proceso), no hay otra opción: para que esto funcione, de algún modo **debe** intervenir el sistema operativo. Linux utiliza tres máscaras (bitmaps) para cada proceso, indicando qué señales deberán ignorarse, cuáles están bloqueadas (se encolan hasta ser desbloqueadas) y cuáles serán capturadas para ser atendidas por el proceso en lugar de aplicarles la acción por defecto.

Entonces, si el proceso tiene un manejador de señal definido, al recibir la señal el proceso bloqueará la recepción de esa señal, ejecutará la función correspondiente y, al finalizar, desbloqueará la señal. De este modo, se evita interrumpir la ejecución del manejador de la señal, para evitar conflictos con por sincronización (si el manejador lockea un mutex y se interrumpe con una nueva ejecución de si mismo antes de liberarlo, estaríamos en presencia de un auto-deadlock `:)`).

El problema que inspiró este post viene con el uso de `exec*`. `execve` es una llamada al sistema que reemplaza la imagen de un proceso por una nueva, pasando a ejecutar un nuevo programa. La familia de funciones `exec*` son wrappers de esa syscall.

Lo importante de todo esto son dos cosas: una `execve` exitosa no retorna, y reemplazar la imagen de un proceso no implica cambiar los atributos del mismo. De `man 2 execve`:

    All process attributes are preserved during an execve(), except the following:

Y, adiviná qué: las máscaras de las señales no son parte de las excepciones. Entonces, repasemos: llega la señal, se bloquean la señal recibida, y se ejecuta el handler de la señal. Si ese handler hiciera un `execve` exitoso, la función no retorna: empieza a ejecutar el nuevo proceso. Pero, si el handler no retorna, nunca se desbloquea la señal, por lo que el nuevo proceso no recibirá más la señal especificada, porque serán encoladas eternamente. Bang.

La solución a esto es desbloquear la señal a mano antes de llamar a `execve`. En C, desde dentro del handler, podríamos lograrlo usando este fragmento de código:

{% codeblock lang:c %}
sigset_t set;
sigemptyset(&set);
sigaddset(&set, SIGINT);
sigprocmask(SIG_UNBLOCK, &set, NULL);
{% endcodeblock %}
    

`sigemptyset` inicializa un set de señales vacío, y luego marcamos la señal que nos interese (en este caso, `SIGINT`) en ese set. Por último, la función `sigprocmask` es la que se encargará, con estos parámetros, de desbloquear el set de señales marcados (sólo `SIGINT`). Luego de esto, sí, podremos ejecutar `execve` sin problemas.

**Créditos**: mucha información de este post surgió a partir de [esta respuesta en StackOverflow](http://stackoverflow.com/a/1024778).

[^1]: Se pueden listar todas las señales ejecutando `kill -l`
[^2]: En Linux, que tiene <abbr title="Kernel Level Thread - Hilos de Kernel">KLTs</abbr>, uno puede enviar la señal especificando un <abbr title="Thread ID - Identificador de Hilo">TID</abbr> en lugar de un <abbr title="Process ID - Identificador de Proceso">PID</abbr>, por lo que puede mandarle la señal a un hilo en particular. Poniéndonos más técnicos, el PID coincide con el TID del hilo principal (`main()`), por lo que al enviarle la señal al proceso en realidad se la estamos enviando al hilo principal. Pero la idea genérica de "las señales se envían a procesos" es más feliz.