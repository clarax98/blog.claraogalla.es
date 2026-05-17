---
title: "GaLink: cómo convertí una pistola arcade USB en un mando Xbox 360"
date: 2026-05-17
draft: false
tags: ["python", "usb", "gaming", "windows", "hardware", "pyusb", "vgamepad"]
description: "GaLink es un adaptador software que traduce la entrada de una pistola Gaime USB a un mando virtual Xbox 360, con calibración de 5 puntos y overlay en pantalla."
showToc: true
---

Las pistolas de luz arcade tienen un problema en PC moderno: los juegos esperan un mando o un ratón, no un dispositivo USB propietario con su propio protocolo. GaLink nació para resolver exactamente eso — traducir la entrada de una pistola Gaime USB a algo que cualquier juego de Windows entienda.

## El problema

Las pistolas Gaime se conectan por USB pero no se identifican como un ratón ni como un mando estándar. El firmware habla su propio protocolo en paquetes de 64 bytes con coordenadas en formato little-endian. Windows no sabe qué hacer con eso.

El objetivo era que juegos como House of the Dead o Time Crisis recibiesen la pistola como si fuera un mando Xbox 360 normal — sin parches, sin drivers especiales en el juego.

## La arquitectura

GaLink se divide en dos capas principales:

**`MenuPrincipal`** — la ventana de configuración donde se gestionan perfiles por juego, el idioma, y se lanza el sistema.

**`GaLinkSystem`** — la ventana de juego a pantalla completa. Aquí ocurre todo lo interesante: lectura USB en hilos separados, render del puntero, calibración y conversión de entrada.

```
Pistola USB
    ↓  pyusb (64 bytes, little-endian)
GaLinkSystem
    ↓  calibración de 5 puntos
Coordenadas en pantalla
    ↓  vgamepad
Mando virtual Xbox 360
    ↓
Juego
```

## El driver USB: Zadig + pyusb

Windows no permite acceso directo a dispositivos USB sin un driver compatible. La solución es reemplazar el driver original de la pistola con **WinUSB** usando [Zadig](https://zadig.akeo.ie/), lo que permite que `pyusb` hable directamente con el hardware.

```python
import usb.core

pistola = usb.core.find(idVendor=VID, idProduct=PID)
pistola.set_configuration()
pistola.claim_interface(0)
```

Los datos llegan en paquetes de 64 bytes. Las coordenadas X e Y vienen en little-endian y hay que interpretarlas antes de poder usarlas.

## El mando virtual: vgamepad + ViGEmBus

Para emular un mando Xbox 360 se usa `vgamepad`, que se apoya en el driver **ViGEmBus** de Valve. Una vez instalado el driver, crear un mando virtual es trivial:

```python
import vgamepad as vg

gamepad = vg.VX360Gamepad()

# Apuntar mueve el joystick izquierdo
gamepad.left_joystick_float(x_value_float=x, y_value_float=y)

# Disparar es el botón A
gamepad.press_button(button=vg.XUSB_BUTTON.XUSB_GAMEPAD_A)
gamepad.update()
```

El loop principal corre a 10 ms (100 Hz) para mantener latencia baja.

## La calibración de 5 puntos

La calibración es la parte más importante del proyecto. Las pistolas de luz leen posición relativa en su sensor, no coordenadas absolutas de pantalla — hay que mapear el espacio del sensor al espacio de la pantalla.

Al iniciar, aparecen 5 puntos en pantalla en este orden:

1. Arriba izquierda
2. Arriba derecha
3. Abajo derecha
4. Abajo izquierda
5. Centro

Disparas a cada punto y GaLink registra la lectura del sensor en ese momento. Con esos 5 pares de valores construye la transformación afín que convierte cualquier lectura del sensor en coordenadas de pantalla.

## El overlay

Mientras juegas hay un overlay transparente a pantalla completa que muestra el puntero en tiempo real. Está hecho con `tkinter` con `overrideredirect(True)` y transparencia de ventana — la ventana existe pero no se ve, solo el puntero encima del juego.

## Problemas que encontré

**Movimiento inestable** — las pistolas IR son sensibles a la iluminación ambiente. Con otra pantalla encendida cerca o luz solar directa el sensor se satura. El suavizado de movimiento y la zona muerta configurable ayudan mucho.

**Ejes invertidos** — algunos juegos invierten los ejes del joystick. Añadí una opción en la interfaz para invertirlos sin tocar código.

**Reconexión USB** — si la pistola se desconecta y reconecta, el sistema tiene que reabrir los recursos USB sin dejar botones atascados en el mando virtual. Hubo que manejar los timeouts USB explícitamente.

**ViGEmBus** — muchos usuarios instalan `vgamepad` pero se olvidan del driver del kernel. Sin ViGEmBus el mando virtual no aparece. Lo documenté bien en el README para evitar confusiones.

## Resultado

GaLink funciona con cualquier pistola Gaime y cualquier juego que acepte mando Xbox 360. Los perfiles por juego permiten guardar configuraciones distintas de sensibilidad, zona muerta y mapeo de botones.

El proyecto está en GitHub: [clarax98/GaLink](https://github.com/clarax98/GaLink)
