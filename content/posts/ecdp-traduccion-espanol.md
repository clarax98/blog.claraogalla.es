---
title: "eCDP al español: traduje el juego de entrenamiento de McDonald's para Nintendo DSi"
date: 2026-05-17
draft: false
tags: ["localización", "traducción", "nintendo-dsi", "open-source", "japonés", "mcdonalds"]
description: "eCrew Development Program es un videojuego educativo de McDonald's Japón para Nintendo DSi. La comunidad lo portó al inglés y yo hice el fork con la traducción al español."
showToc: false
---

**eCrew Development Program** (eCDP), conocido también como *McDonald's Training Game*, es un videojuego educativo creado por McDonald's para **Nintendo DSi**. Se usaba en Japón para enseñar a los empleados los procedimientos del restaurante: preparación de menús, atención al cliente, tiempos de producción. Todo en forma de minijuegos dentro de una DSi.

Es exactamente lo que suena: un juego oficial de McDonald's que los empleados de los restaurantes japoneses usaban en consolas portátiles para formarse.

## La cadena de forks

El juego original es exclusivamente en japonés y nunca salió de Japón. En algún momento la comunidad reverse-engineeró el formato y produjo un fork con la interfaz traducida al inglés. Yo encontré ese fork, lo usé, y decidí hacer otro para tener el juego en español:

```
eCDP original (japonés, McDonald's Japón, Nintendo DSi)
    ↓ fork de la comunidad
Versión en inglés
    ↓ mi fork
eCDP-Spanish
```

## Por qué lo traduje

Principalmente porque me pareció un proyecto curioso y único. No hay muchos juegos educativos corporativos japoneses para DSi con traducción comunitaria, y menos aún en español. Es el tipo de rareza que vale la pena preservar en más idiomas.

También porque localizar algo así tiene su gracia técnica: el vocabulario de McDonald's tiene términos muy específicos que en España y Latinoamérica se conocen de formas distintas ("menú" vs "combo", nombres de productos que varían por país, etc.). Hay que tomar decisiones sobre a qué variedad del español apuntar.

## El proceso

Localizar un juego de DSi no es lo mismo que traducir un documento. Las cadenas de texto están empaquetadas en el formato del juego, hay que extraerlas, traducirlas sin superar la longitud máxima que cabe en pantalla, y volver a empaquetarlas. La DSi tiene una pantalla pequeña y los textos tienen límites de caracteres estrictos.

El fork en inglés ya tenía hecha la parte difícil de la extracción. Mi trabajo fue reemplazar las cadenas en inglés por español respetando esos límites y manteniendo el tono apropiado para un material de formación.

## El repo

[clarax98/eCDP-Spanish](https://github.com/clarax98/eCDP-Spanish) — si te interesa el juego en español, ahí está. Es una rareza, pero una rareza con historia detrás.
