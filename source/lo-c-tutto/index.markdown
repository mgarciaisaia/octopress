---
layout: page
title: "Lo C Tutto"
date: 2013-08-22 00:21
comments: true
sharing: true
footer: true
author: mgarciaisaia
categories: [lo-c-tutto]
---

Lo C Tutto es un tutorial introductorio al lenguaje de progmraación C y varias otras herramientas y conceptos útiles a la hora de desarrollar el trabajo práctico de Sistemas Operativos. La idea es avanzar despacio pero a pasos firmes, explicando todo lo que corresponda, y tomándose el tiempo que haga falta.

Probablemente haya conceptos que me pase por alto, pero en principio la idea es explicar de modo tal que una persona que venga sin muchos conocimientos previos pueda entender qué es lo que se espera que haga en la materia.

Como de costumbre, cualquier error, sugerencia o agregado que quieran hacer, pueden contactarme por mail[^1], [levantar un issue en GitHub](https://github.com/mgarciaisaia/tutorial-c/issues) o [enviarme un pull request](https://github.com/mgarciaisaia/tutorial-c/fork).

Dejo el índice (el más nuevo primero):

<ul>
{% for post in site.categories['lo-c-tutto'] %}
<li><a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
</ul>


[^1]: mgarciaisaia [ arroba ] gmail [ punto ] com
