---
title: "Cómo porteé Rising Dusk a consolas ARM64: joystick, audio y DLLs"
date: 2026-05-17
draft: false
tags: ["linux", "wine", "box64", "gaming", "arm64", "rocknix", "portmaster", "pipewire", "inputplumber"]
description: "El proceso completo de portar Rising Dusk a ARM64 (ROCKNIX, Retroid Pocket Flip 2): cómo resolver el drift del stick con doble capa de deadzone, los artefactos de audio de Wine y por qué están las DLLs de Steam."
showToc: true
---

Rising Dusk es un pequeño juego de plataformas de Studio Stobie disponible en Steam. Decidí portearlo a mi Retroid Pocket Flip 2 (ARM64, ROCKNIX) porque no existía ningún port. La parte de "lanzar el .exe en Wine" fue lo de menos — los problemas reales vinieron del joystick, el audio y las DLLs. Aquí está el desglose técnico completo.

## La cadena de ejecución

El juego es un ejecutable de Windows x86_64. Las consolas ROCKNIX corren Linux sobre ARM64. La cadena completa es:

```
Gamescope → Wine → Box64 → Rising Dusk.exe
```

- **Box64** traduce llamadas x86_64 a ARM64 en tiempo real.
- **Wine** implementa la API de Windows sobre Linux.
- **Gamescope** gestiona el escalado de resolución y el foco del cursor a pantalla completa.

El script de arranque construye el entorno antes de llamar a Gamescope:

```bash
gamescope -f -w "$DISPLAY_WIDTH" -h "$DISPLAY_HEIGHT" -- \
  env WINEPREFIX="$HOME/risingdusk_wine" \
      WINEDEBUG=-all \
      WINEDLLOVERRIDES="steam_api=n,b;steamwrap=n,b" \
      BOX64_EMULATED_LIBS="SDL2" \
      SDL_GAMECONTROLLERCONFIG="$sdl_controllerconfig" \
      SDL_JOYSTICK_ALLOW_BACKGROUND_EVENTS=1 \
      "${AUDIO_ENV[@]}" \
  wine "$EXEC"
```

Cada variable de entorno ahí tiene su razón de ser, que explico en las secciones siguientes.

---

## El problema del joystick: drift y deadzone en dos capas

Este fue el problema que más tiempo me llevó. En el Retroid Pocket Flip 2 el stick izquierdo **no reposa en cero**. En reposo absoluto el eje reporta aproximadamente **−3300** sobre un rango de ±32767, lo que equivale al 10% del rango total. El stick derecho reposa en torno a **+2200** (7%).

Wine no filtra nada de esto. Pasa el valor crudo directamente al juego, que interpreta ese 10% como movimiento constante. El personaje anda solo hacia arriba-izquierda sin tocar nada.

La solución necesitó **dos capas de filtrado**, porque una sola no era suficiente.

### Capa 1: gptokeyb

`gptokeyb` es la herramienta de ROCKNIX para mapear controles. El fichero `risingdusk.gptk` configura el modo de deadzone y el umbral:

```ini
[config]
deadzone_mode = axial
deadzone = 5000

[controls]
start = esc
select = esc

left_analog_up    = up
left_analog_down  = down
left_analog_left  = left
left_analog_right = right
```

`deadzone_mode = axial` aplica el deadzone por eje de forma independiente (no circular), lo que va mejor para movimiento en 8 direcciones. `deadzone = 5000` es el umbral — cualquier valor absoluto por debajo de ese en cualquier eje se ignora. Esto elimina el drift en reposo, pero a costa de perder sensibilidad en el rango bajo del stick.

El problema con solo gptokeyb es que mapea los ejes a teclas de cursor (`up/down/left/right`), no a un eje analógico real. El juego recibe inputs digitales, no analógicos — funcional, pero no ideal.

### Capa 2: InputPlumber (ROCKNIX)

ROCKNIX incluye **InputPlumber**, un daemon que gestiona los dispositivos de entrada a nivel de sistema antes de que Wine o SDL los vean. El perfil `risingdusk_profile.yaml` lo configura específicamente para este juego:

```yaml
- name: Left Stick Deadzone
  source_event:
    gamepad:
      axis:
        name: LeftStick
  target_events:
  - gamepad:
      axis:
        name: LeftStick
        deadzone: 0.15   # 15% — cubre el reposo en ~-3300 (10% del rango)

- name: Right Stick Deadzone
  source_event:
    gamepad:
      axis:
        name: RightStick
  target_events:
  - gamepad:
      axis:
        name: RightStick
        deadzone: 0.10   # 10% — cubre el reposo en ~2200 (7%)
```

InputPlumber aplica la deadzone **antes** de que el evento llegue al sistema, así Wine y SDL ya reciben valores limpios. El 15% para el stick izquierdo cubre el drift de reposo de −3300 con margen.

El script carga y descarga el perfil automáticamente:

```bash
# Al arrancar
inputplumber device "$IPDEV" profile load "$GAMEDIR/risingdusk_profile.yaml"

# Al salir
inputplumber device "$IPDEV" profile load \
  /usr/share/inputplumber/profiles/default.yaml
```

En CFWs que no tienen InputPlumber (JELOS, KNULLI), el bloque se salta sin error — la variable `$IPDEV` simplemente no se usa.

---

## El problema del audio: artefactos, buffer y PipeWire

El audio fue el otro quebradero de cabeza. La cadena en ROCKNIX es:

```
Rising Dusk.exe → Wine (wineaudio) → SDL2 → PipeWire/ALSA → hardware
```

Con el buffer por defecto de SDL aparecían **artefactos audibles a volumen bajo**: una especie de crujido o chasquido periódico. No es un bug del juego — es el pipeline de Wine/Box64 introduciendo latencia variable que el buffer no absorbe.

La solución es forzar un buffer grande y coherente en toda la cadena:

```bash
SDL_AUDIODRIVER=pipewire
SDL_AUDIO_SAMPLES=4096
PIPEWIRE_LATENCY=4096/44100
```

`SDL_AUDIO_SAMPLES=4096` le dice a SDL que use un buffer de 4096 frames en vez del default (512 o 1024). `PIPEWIRE_LATENCY=4096/44100` le dice a PipeWire que use exactamente la misma ventana (~93 ms). Si los dos valores no coinciden, PipeWire tiene que hacer resampling o buffering adicional, y ahí es donde aparecen los artefactos.

93 ms de latencia de audio es mucho en un juego AAA, pero para un platformer indie como Rising Dusk es imperceptible.

### Fallback a ALSA

En ROCKNIX siempre hay PipeWire, pero en JELOS y algunas builds de KNULLI puede no estarlo. El script detecta el socket de PipeWire antes de arrancar:

```bash
_pw_sock="${PIPEWIRE_RUNTIME_DIR:-/run/pipewire}/pipewire-0"
if [ -S "$_pw_sock" ]; then
  AUDIO_ENV=(
    SDL_AUDIODRIVER=pipewire
    "PIPEWIRE_RUNTIME_DIR=${PIPEWIRE_RUNTIME_DIR:-/run/pipewire}"
    SDL_AUDIO_SAMPLES=4096
    PIPEWIRE_LATENCY=4096/44100
  )
else
  AUDIO_ENV=(
    SDL_AUDIODRIVER=alsa
    SDL_AUDIO_SAMPLES=4096
  )
fi
```

En ALSA sin PipeWire los artefactos a volumen muy bajo siguen existiendo pero son menos frecuentes. No tiene solución limpia — es una limitación del pipeline Wine/Box64/ALSA.

---

## Las DLLs: por qué están ahí y qué hace cada una

Esta es la pregunta que más me hicieron en el hilo de PortMaster Discord. El directorio del juego incluye `steam_api.dll` y `steam_api64.dll` que no son las DLLs originales de Steam.

### `steam_api.dll` / `steam_api64.dll` — Goldberg Steam Emulator

Rising Dusk llama a la API de Steam al arrancar. En la consola no hay Steam, así que el juego carga `steam_api.dll` y falla inmediatamente.

La solución es **Goldberg Steam Emulator**: una reimplementación limpia de la API de Steam bajo licencia MIT. Sus DLLs exponen las mismas funciones que las de Valve (`SteamAPI_Init`, `SteamAPI_RunCallbacks`, `SteamAPI_Shutdown`, etc.) pero sin necesitar el cliente de Steam. El juego las carga, las inicializa sin error y arranca.

> Las DLLs de Goldberg **no contienen código de Valve**. Son una reimplementación propia compatible con la misma interfaz binaria. El juego original hay que comprarlo en Steam — sin los archivos del juego, el port no hace nada.

La variable `WINEDLLOVERRIDES` es crítica aquí:

```bash
WINEDLLOVERRIDES="steam_api=n,b;steamwrap=n,b"
```

`n,b` significa "native first, then builtin" — Wine primero busca la DLL nativa (la que está en el directorio del juego, es decir Goldberg), y si falla intenta la builtin de Wine. Sin este override, Wine podría ignorar la DLL local e intentar cargar la de Steam del sistema, que en una consola no existe.

### `steamwrap.ndll` — biblioteca de Haxe/Lime

`steamwrap.ndll` es una biblioteca nativa de **Haxe/Lime**, el framework con el que está hecho Rising Dusk. Es el puente entre el código Haxe del juego y la API de Steam — llama a funciones de `steam_api.dll` para logros, guardado en la nube, etc.

Esta DLL **no** la proporciona el port — viene del juego original y hay que copiarla junto con `Rising Dusk.exe`. Box64 la traduce de x86_64 a ARM64 como cualquier otro binario de Windows.

### `BOX64_EMULATED_LIBS="SDL2"`

Esta variable le dice a Box64 que, en vez de usar la librería SDL2 del sistema ARM64 nativa (que puede tener una versión distinta o flags de compilación distintas), emule la SDL2 de Windows x86_64. Esto evita incompatibilidades de versión entre la SDL2 que compiló el juego y la SDL2 que hay en ROCKNIX.

---

## El prefijo de Wine: ext4 obligatorio

```bash
export WINEPREFIX="$HOME/risingdusk_wine"
```

El prefijo está en `$HOME` y no en la tarjeta SD a propósito. En ROCKNIX, `$HOME` es `/storage`, que está en la memoria interna formateada en **ext4**. Las tarjetas SD suelen estar en exFAT o FAT32, y Wine necesita un sistema de ficheros que soporte **symlinks** — exFAT no los soporta. Poner el prefijo en la SD en FAT32 hace que Wine falle silenciosamente al inicializarse.

En el primer arranque, si no existe el prefijo, el script lo crea:

```bash
if [ ! -f "$WINEPREFIX/system.reg" ]; then
  pm_message "Setting up Wine prefix (first run, takes ~30 s)..."
  wineboot -i 2>/dev/null
fi
```

Los arranques posteriores son inmediatos porque el prefijo ya existe.

---

## Lo que aprendí

- El drift de joystick en ARM handhelds requiere doble capa de filtrado: a nivel de sistema (InputPlumber) y a nivel de herramienta de mapeo (gptokeyb). Una capa sola no es suficiente.
- El audio de Wine/Box64 necesita que el buffer de SDL y la latencia de PipeWire estén sincronizados exactamente — si no coinciden, aparecen artefactos aunque el juego funcione.
- `WINEDLLOVERRIDES` es imprescindible cuando el juego trae sus propias DLLs — sin él, Wine puede ignorarlas.
- El prefijo de Wine **tiene** que estar en ext4. En exFAT/FAT32 falla sin mensaje de error claro, lo que hace el diagnóstico innecesariamente difícil.
- Box64 + Wine funciona sorprendentemente bien para juegos indie pequeños en ARM64, pero cada juego tiene sus propias quirks.

El port está en [clarax98/rising-dusk-port](https://github.com/clarax98/rising-dusk-port).
