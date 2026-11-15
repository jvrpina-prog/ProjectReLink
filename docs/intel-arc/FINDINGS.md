# Intel Arc A380 — hallazgos y plan de fork

Objetivo: correr OoT remake (Switch 2, `0400486024DEC000`) bien en **Intel Arc A380 (6GB)** + i5-8500 (6c/6t, 2017) + 16GB DDR4, Windows.

Estado al 2026-11-15: **¡corre y es jugable!** Se ve el mundo (Kokiri Forest, Link, Navi, cinemática del Deku), **60 fps en la mayoría de zonas**, sin crash. Falta: FPS en zonas densas y artefactos gráficos.

## Config que funciona hoy (relink_env.txt, al lado de ReLink.exe)

```
RX_VK_NOOPT=1
RX_VK_SKIP_PROG=ffb53ed102dd9908,53509044401d78d2
RX_VK_HANG_DUMP=relink_hang.txt
```

- **`RX_VK_NOOPT=1`** — compila pipelines sin el optimizer de `igvk64.dll`. Sin esto: el driver falla compilando ~10 programas del mundo → esos draws se saltan → **pantalla negra / solo HUD**. Con esto: el mundo se dibuja. Es la llave de entrada.
- **`RX_VK_SKIP_PROG=ffb53ed102dd9908,...`** — salta el programa que causaba **device-lost** (confirmado en los markers del log; es el MISMO que crasheaba en Hookshot). `53509044401d78d2` es el skip por default de OoT que no hay que perder al sobreescribir la lista.

El entorno NVN del A380 (del log): Vulkan 1.4, driver `0x1963d7`, mesh shaders **sí**, pipeline libraries **sí**, `VK_EXT_shader_atomic_float` sí. **`VK_EXT_device_fault` NO** → `RX_VK_FAULT_INFO` es inútil; para ubicar crashes hay que usar checkpoints/markers NV.

## Problema 1: artefactos gráficos (plantas fantasma, titileo)

**Causa raíz:** varios shaders de **cómputo fallan al traducir** (`dispatches of program X are left out (it did not translate)` / "rejected compute" en el log). Son mayormente las **redes de procesamiento de imagen / upscaler** del juego, que usan **cooperative matrix**. El A380 no tiene el tipo necesario — log: *"the cooperative matrices lack 16x8x16 f16"* → caen al stub de reemplazo o dejan salida incompleta.

Es el **mismo problema del upscaler** que hundía a los emuladores, reaparecido en el recomp.

**Dónde mirar en el código:**
- `gpu/vk/vk_draw.cpp` ~7379 — el log de "rejected compute" (cuando `!cp || cp->failed`).
- `gpu/vk/vk_coopmat.cpp` — manejo de cooperative matrix; `RX_VK_MMA_NO_COOP` lo apaga (cae a software).
- `upscale_stub` / `RX_VK_FORCE_UPSCALE_STUB` — la ruta de reemplazo; puede dar "neon colours" (issues 425/508).

**A probar en el fork:** (a) por qué esos compute no traducen con MMA en software (¿bug en la ruta software del cooperative matrix?); (b) mejorar el stub para que no deje fantasmas; (c) identificar cuál de los ~10 programas fallidos produce el titileo del follaje.

## Problema 2: FPS en zonas densas (CPU-bound, NO GPU)

**Confirmado por log:** `GPU-bound wait 0.0 ms` siempre. La GPU termina al instante. Bajar resolución / meter FSR **NO ayuda** (no era la GPU el cuello).

Patrón:
- Escenas normales (2-48 draws) → **60 fps**.
- Kokiri con follaje (**800-1100 draws/frame**) → **7-8 fps**.

El i5-8500 no procesa 1000+ draw calls/frame. Kokiri Forest es de las zonas más densas del juego (peor caso). Palancas a explorar: occlusion culling (`RX_VK_OCC_QUERIES` — ¿culla algo en Intel? en el log no bajaba draws), aliviar el sistema de captura de memoria/mirrors por submit, reducir draws. Nota: una CPU más moderna subiría el piso; la A380 no es el cuello con este juego.

## Plan del fork

1. **Baquear defaults de Intel** (vendorID `0x8086`): activar NOOPT + la skip-list automáticamente, para que el usuario no necesite el relink_env.txt. (Patch simple, bajo riesgo — primer commit a probar.)
2. **Arreglar la traducción de los compute con cooperative-matrix** (el upscaler) → mata los artefactos. (El trabajo pesado.)
3. **Explorar reducción de draws / occlusion en Intel** para las zonas densas.
4. CI que compile el renderer (cazar errores de build antes de gastar un ciclo en la PC).

## Cómo se buildea (recordatorio)

El juego se arma en la PC del usuario desde el fuente del fork:
`python -I nx2/make_game.py --nup <...> --out <...>`
Necesita: Python + `pip install lz4 numpy capstone "unicorn>=2.1,<2.2" zstandard`, **llvm-mingw** (ucrt-x86_64), Vulkan headers + FFmpeg en el CPM cache (`RX_CPM`), y los prebuilts de `third_party` + perfil de `work` (vienen con un ReLink instalado, en `%LOCALAPPDATA%\ReLink\versions\<v>\ReLink\tools`).
