---
title: "eCDP al español: cómo localicé un simulador de tripulación de vuelo japonés"
date: 2026-05-17
draft: false
tags: ["localización", "traducción", "open-source", "japonés", "simulación"]
description: "eCrew Development Program es un simulador de tripulación de vuelo originalmente en japonés. La comunidad lo portó al inglés, y yo hice el fork con la traducción al español."
showToc: false
---

**eCrew Development Program (eCDP)** es un software de simulación enfocado en la formación de tripulación de vuelo. El proyecto original está en japonés. En algún punto la comunidad hizo un fork y lo tradujo al inglés. Yo encontré ese fork, lo usé, y decidí hacer otro fork con la traducción al español.

## Por qué lo traduje

El software existía en inglés pero no en español, y hay mucha gente hispanohablante interesada en aviación y simulación que se beneficiaría de tenerlo en su idioma. Es el tipo de contribución que no requiere saber programar a fondo — requiere entender bien el dominio (en este caso, terminología de aviación) y tener cuidado con el contexto de cada cadena de texto.

## El proceso de localización

La localización de software no es solo traducir palabra por palabra. Hay que tener en cuenta:

**Terminología técnica** — la aviación tiene vocabulario muy específico. "Cabin crew", "boarding", "pushback", "turnaround"... muchos términos tienen una traducción oficial en español aeronáutico que no es la traducción literal. Usé terminología estándar de AESA e IATA en español.

**Longitud de las cadenas** — el español tiende a ser más largo que el inglés. Una cadena que en inglés cabe en un botón puede no caber en español. Hay que buscar formulaciones equivalentes más cortas sin perder el significado.

**Contexto de la interfaz** — una misma palabra puede traducirse distinto según dónde aparece. "Cancel" en un diálogo de confirmación es "Cancelar", pero en el contexto de cancelación de un servicio puede ser "Dar de baja". Sin ver la interfaz en contexto es fácil equivocarse.

## La cadena de forks

```
Original japonés (eCDP)
    ↓ fork de la comunidad
Versión en inglés
    ↓ mi fork
eCDP-Spanish
```

Mantener un fork de un fork tiene sus complicaciones — si el fork en inglés se actualiza hay que hacer merge y revisar qué cadenas nuevas aparecieron. No es un trabajo puntual sino de mantenimiento.

## El repo

[clarax98/eCDP-Spanish](https://github.com/clarax98/eCDP-Spanish) — si usas eCDP y prefieres tenerlo en español, ahí está.
