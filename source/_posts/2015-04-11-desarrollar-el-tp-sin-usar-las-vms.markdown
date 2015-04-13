---
layout: post
title: "Desarrollar el TP sin usar las VMs"
date: 2015-04-11 14:43
comments: true
categories: 
---

Una pregunta frecuente durante los inicios de cuatrimestre es "¿Puedo desarrollar el TP instalando nativamente Linux en mi máquina?". Hay quienes quieren hacerlo porque ya usan Linux y les es más cómodo usar las configuraciones que ya le hicieron a la máquina, mientras que a otros la VM les funciona lento, y correr nativamente las aplicaciones agiliza un poco las cosas.<!-- more -->

El trabajo práctico se programa en C sobre un Linux de 32 bits, sin usar más bibliotecas de terceros que las que vienen en el propio sistema operativo, y la evaluación del TP se hace en el laboratorio de Medrano, en las computadoras del laboratorio de Medrano, corriendo en la VM Ubuntu Server que la cátedra entrega. _El trabajo práctico tiene que funcionar correctamente en la VM Server en las computadoras del laboratorio de Medrano_.

C es un lenguaje multiplataforma, en cuanto a que uno puede escribir un programa y compilarlo en distintas máquinas, obteniendo cada vez un programa ejecutable para esa plataforma en particular. Pero que el lenguaje sea multiplataforma no significa que todo código programado en C también lo sea.

Las bibliotecas (como todo proyecto de software) evolucionan. Si bien en general los desarrolladores buscan mantener las interfaces (APIs) y comportamientos de las funciones, de modo de mantener la compatibilidad con versiones anteriores de la misma biblioteca, a veces aparecen cambios (accidentales o intencionales) que modifican el comportamiento de alguna función.

Si un proceso del trabajo práctico dependía de que esa función ejecute de un modo determinado (y no del otro), ese trabajo práctico podría fallar por un cambio en una biblioteca de tercero. En general, _nadie quiere que le pase eso_ :) Y asegurarse de que todas las bibliotecas de las que dependemos se mantengan en la misma versión que en las de la máquina destinto no es _tan fácil_ como podría esperarse.

Además de eso, eclipse (el IDE que recomendado por la cátedra para desarrollar el TP) escribe rutas absolutas en varias partes de su configuración. En particular, en los Makefiles (los archivos que describen cómo compilar los proyectos) eclipse pone las rutas absolutas de los archivos que usa para compilar. Si los diferentes integrantes del grupo guardan los proyectos en rutas diferentes, versionar el Makefile va a ser un problema. En las VMs, todos tienen creado el mismo usuario. Si te hacés tu propia instalación, es más fácil olvidarse de esto.

Teniendo en cuenta todo esto, son libres de desarrollar _como quieran_. Nuestro más sabio consejo es que, lo hagan de la forma que lo hagan, **no dejen de probar el TP en la VM Server**, y, de ser posible, _en el laboratorio mismo_.

Para cerrar, el consejo un poco más genérico que damos siempre: el desarrollo del TP, siguiendo todas las recomendaciones y consejos que les damos, ya tiene suficiente complejidad como para tener un cuatrimestre movido. Si pueden evitar sumarse más complejidad, se van a estar haciendo un favor.
