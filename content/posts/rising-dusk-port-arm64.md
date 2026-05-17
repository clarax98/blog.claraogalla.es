---
title: "Cómo porteé Rising Dusk a consolas ARM64 con Wine y Box64"
date: 2026-05-17
draft: false
tags: ["linux", "wine", "box64", "gaming", "arm64", "rocknix", "portmaster"]
description: "El proceso completo de portar un juego de Windows a consolas handheld ARM64 (Retroid Pocket Flip 2) usando Wine, Box64 y Gamescope."
showToc: true
---

Rising Dusk es un pequeño juego de plataformas de Studio Stobie disponible en Steam. Decidí portearlo a mi Retroid Pocket Flip 2 (ARM64, ROCKNIX) porque no existía ningún port y el proceso me parecía un reto interesante. Aquí cuento cómo lo hice.

## El problema: Windows .exe en ARM64 Linux

El juego es un ejecutable de Windows de 32/64 bits. Las consolas ROCKNIX corren Linux sobre ARM64. Para ejecutar software de Windows en Linux existe **Wine**, pero Wine por sí solo no entiende la arquitectura ARM64 — ahí entra **Box64**.

- **Box64**: traduce llamadas x86_64 a ARM64 en tiempo real.
- **Wine**: capa de compatibilidad que implementa la API de Windows sobre Linux.
- **Gamescope**: compositor de Valve que gestiona el escalado y la ventana del juego.

Juntos forman la cadena: `Gamescope → Wine → Box64 → Rising Dusk.exe`

## El script de arranque

El `RisingDusk.sh` que va en `ports/` del dispositivo hace esto:

```bash
#!/bin/bash
export WINEPREFIX="$HOME/.wine_risingdusk"
export WINEARCH=win64
export BOX64_LOG=0

gamescope -w 1280 -h 720 -r 60 --force-grab-cursor -- \
  wine "$GAMEDIR/Rising Dusk.exe"
```

El prefijo de Wine separado evita contaminar el prefijo global con las DLLs del juego.

## El problema de steam_api.dll

Rising Dusk llama a la API de Steam al arrancar. En la consola no hay Steam, así que el juego carga `steam_api.dll` y falla al no encontrar el cliente.

La solución: **Goldberg Steam Emulator**. Es una reimplementación limpia de la API de Steam bajo licencia MIT. Sus DLLs reemplazan las originales y devuelven valores válidos para las funciones que el juego llama (`SteamAPI_Init`, `SteamAPI_RunCallbacks`, etc.), sin necesidad de tener Steam instalado.

> Las DLLs de Goldberg **no son código de Valve** — son una implementación propia compatible con la misma interfaz. El juego en sí hay que comprarlo en Steam.

## Controles: gptokeyb

ROCKNIX incluye `gptokeyb` para mapear el mando a teclas y ratón. Rising Dusk acepta mando directamente a través de Wine/SDL, pero el fichero `.gptk` permite ajustar el comportamiento:

```ini
[default]
start = escape
select = escape
```

## Resultado

El juego corre fluido a 60fps en el Retroid Pocket Flip 2. El audio funciona correctamente a través del pipeline de Wine/Box64. Las partidas guardadas se guardan en el prefijo de Wine.

El port está disponible en GitHub: [clarax98/rising-dusk-port](https://github.com/clarax98/rising-dusk-port)

## Lo que aprendí

- Box64 + Wine es una combinación sorprendentemente funcional para juegos pequeños/indie en ARM64.
- Separar el prefijo de Wine por juego evita muchos problemas de DLLs.
- Gamescope añade estabilidad al manejo del foco y el escalado en dispositivos handheld.
- Los emuladores de API como Goldberg son herramientas legítimas y bien documentadas para entornos sin conectividad.
