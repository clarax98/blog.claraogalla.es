---
title: "Mi setup: CachyOS + KDE Plasma 6 + Ghostty con instalación automatizada"
date: 2026-05-17
draft: false
tags: ["linux", "cachyos", "arch", "kde", "dotfiles", "zsh", "ghostty", "starship", "wayland"]
description: "Cómo tengo configurado mi entorno de escritorio en CachyOS con KDE Plasma 6, Ghostty, zsh, Starship (Catppuccin Mocha) y un script de instalación que lo despliega todo solo."
showToc: true
---

Llevo tiempo refinando mi entorno de trabajo en Linux. Lo que empezó como copiar configs de aquí y allá acabó convirtiéndose en un repo de dotfiles con un script de instalación que lo despliega todo desde cero en un sistema limpio. Aquí explico qué uso y por qué.

## El sistema base: CachyOS

Uso **CachyOS**, una distribución basada en Arch con el kernel CachyOS que aplica parches de rendimiento (principalmente BORE scheduler y optimizaciones de compilación para x86_64-v3). La diferencia en el día a día es pequeña pero real — aplicaciones que antes tardaban en arrancar lo notan.

Al ser Arch-compatible, `pacman` y AUR funcionan igual. Nada exótico.

## El entorno de escritorio: KDE Plasma 6 en Wayland

KDE Plasma 6 con **KWin en Wayland**. Llevo usando Wayland de forma estable desde hace tiempo sin echar de menos X11 en ningún aspecto práctico. Las ventajas son claras: mejor manejo de pantallas de alta densidad, sin tearing, y aislamiento de aplicaciones.

El estilo de ventanas es **Breeze** nativo — sin temas de terceros para Qt, porque suelen romperse con cada actualización de Plasma. Para iconos uso **candy-icons** y para cursores **Sweet-cursors**, ambos instalados automáticamente por el script.

El launcher es **Andromeda Launcher** (widget de Plasma, preinstalado en CachyOS) mapeado a la tecla Super.

## La terminal: Ghostty

Cambié a **Ghostty** y no he vuelto atrás. Es nativa en Wayland, rápida, y tiene soporte de splits sin necesitar tmux:

| Atajo | Acción |
|-------|--------|
| `Ctrl+Shift+T` | Nueva tab |
| `Ctrl+Shift+O` | Split horizontal |
| `Ctrl+Shift+E` | Split vertical |
| `Alt+←↑↓→` | Navegar entre splits |
| `Ctrl+Shift+F` | Buscar en buffer |

Tema: **Catppuccin Mocha**. Fuente: **JetBrainsMono Nerd Font**.

## La shell: zsh sin Oh My Zsh

Uso zsh pero **sin Oh My Zsh**. OMZ añade demasiada magia invisible y ralentiza el arranque. Los plugins que realmente necesito vienen directamente de pacman:

- `zsh-autosuggestions` — sugerencias en gris mientras escribes
- `zsh-syntax-highlighting` — comandos válidos en verde, errores en rojo

Se cargan directamente en `.zshrc` con `source /usr/share/...`. Arranque instantáneo.

## El prompt: Starship (Catppuccin Mocha)

**Starship** es el mejor prompt que he probado. Muestra lo que necesitas en el contexto en que estás: rama git, lenguaje del proyecto, estado de la batería, hora. La configuración usa el tema **Catppuccin Mocha** para que combine con Ghostty.

## Herramientas CLI que uso a diario

| Herramienta | Reemplaza a | Por qué |
|-------------|-------------|---------|
| `eza` | `ls` | Iconos Nerd Font, git status en columnas |
| `bat` | `cat` | Syntax highlighting, números de línea |
| `ripgrep` | `grep` | 10-100x más rápido, ignora `.git` por defecto |
| `zoxide` | `cd` | Recuerda los directorios que más usas, saltas con `z nombre` |
| `atuin` | historial bash | Búsqueda fuzzy del historial, sincronizable entre máquinas |
| `fzf` | — | Búsqueda fuzzy interactiva para todo |
| `fastfetch` | neofetch | Info del sistema al abrir terminal, más rápido |
| Microsoft Edit | nano/vim | Editor de terminal moderno con atajos familiares |

Los aliases están en `.zshrc` para que `ls`, `cat`, `grep` y `cd` apunten automáticamente a las versiones mejoradas.

## El script de instalación

La parte que más tiempo me ahorró fue escribir `install.sh`. Lo que hace en orden:

1. Comprueba qué paquetes de pacman faltan y los instala
2. Descarga candy-icons y Sweet-cursors desde GitHub si no están
3. Instala Oh My Zsh si no existe (para los plugins de completado)
4. Crea symlinks de plugins zsh desde pacman
5. Cambia el shell de login a zsh con `chsh`
6. Descarga Microsoft Edit desde GitHub releases
7. Activa el estilo Breeze en KDE
8. Configura variables de entorno Wayland
9. Hace backup `.bak` de configs existentes
10. Enlaza todas las configs con **symlinks** — editar el repo es editar la config activa

La clave son los symlinks. En vez de copiar archivos, el script hace:

```bash
ln -sf "$DOTFILES_DIR/ghostty/config" "$HOME/.config/ghostty/config"
ln -sf "$DOTFILES_DIR/starship/starship.toml" "$HOME/.config/starship.toml"
# etc.
```

Así cualquier cambio en el repo se refleja inmediatamente en el sistema sin volver a ejecutar nada.

## Variables de entorno Wayland

Un archivo `wayland.conf` en `~/.config/environment.d/` que KDE carga automáticamente:

```ini
NIXOS_OZONE_WL=1
QT_QPA_PLATFORM=wayland
SDL_VIDEODRIVER=wayland
```

Esto fuerza a aplicaciones Electron y SDL a usar el backend Wayland nativo en vez de XWayland cuando sea posible.

## El repo

Todo está en [clarax98/clara-dotfiles](https://github.com/clarax98/clara-dotfiles). Un `git clone` + `./install.sh` lo despliega todo.
