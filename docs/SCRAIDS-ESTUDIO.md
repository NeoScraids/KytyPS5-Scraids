# KytyPS5-Scraids: estudio de la base Jetsku (U59) y hoja de ruta

Documento de trabajo del fork. Resume qué trae la base
[Jetsku/KytyPS5](https://github.com/Jetsku/KytyPS5) respecto al KytyPS5 oficial, dónde vive cada
mecanismo en el código y qué falta para la meta: **juego fluido a 60 FPS en AMD (RX 9070 XT)**.

Fuentes: `docs/CHANGES-U59.md`, `docs/PIPELINE-STUTTER.md`, `docs/EXPERIMENTAL.md`,
`docs/HANG-WATCHDOG.md`, `docs/release-notes*.md`, `docs/CHANGE-CATALOG.md` (inventario completo).

## 1. De dónde venimos

| Rama | Base | Notas |
|---|---|---|
| `amd-perf` | KytyPS5 oficial | Primeros experimentos. ASTRO's PLAYROOM cuelga la GPU en AMD (`ErrorDeviceLost`, TDR). |
| `scraids` | Jetsku `main` (int19.1) + nuestro branding | Base actual. La build int16.1-amd4 de Jetsku pasa la zona que colgaba. |

Jetsku partió del oficial el 2026-10-08 y añadió ~1.761 commits (377 archivos, +113k líneas).
No es "oficial + parche": es otra línea del emulador. El oficial tiene commits propios que U59 no
incluye; traerlos exige revisión manual (ver `CHANGE-CATALOG.md`).

## 2. Cómo se configura: `u59-preset.json`

Casi todo lo nuevo se activa con variables de entorno `KYTY_*`. El launcher lee
`u59-preset.json` (junto a `launcher.exe`) y las pasa al emulador. **Sin ese archivo el emulador
corre con los valores por defecto del código, más lentos.** `build-scraids.bat` lo copia a
`_Build/scraids/install`. Código: `src/common/u59Preset.*`.

## 3. Mecanismos por subsistema (qué hace y dónde está)

### 3.1 Envío de trabajo a la GPU (CPU → driver)
- **Envío en cola y agrupado** (`queueSubmission.*`; `KYTY_SUBMISSION_MODE=queued`,
  `KYTY_SUBMISSION_COALESCE=1`): saca `vkQueueSubmit` del hilo que dibuja y junta envíos pequeños.
  Medido +5–12 % en sus pruebas.
- **Grabador de comandos** (`commandStream.*`, `commandRecorder.*`; `KYTY_CP_RECORDER=1`): otro
  hilo graba los comandos Vulkan.
- **Secuenciador del CP** (`cpOps.*`, `cpSequencer.*`; `KYTY_CP_SEQ=1`): separa el parseo de PM4
  de su ejecución.

Esto ataca justo lo que medimos en el oficial: un único hilo de GPU saturado.

### 3.2 Preparación de dibujos
- Preparación paralela con certificados de coherencia (`KYTY_DRAW_PREP_BINDINGS=1`), planes de
  binding, memos de estado/descriptores, recetas SRT compiladas (`KYTY_SRT_*`).
- "Draw runs" (`KYTY_DRAW_RUN*`): procesa secuencias de dibujos de una vez.

### 3.3 Shaders y pipelines (los tirones)
- **Caché de programas en disco** (`_PipelineCache/<juego>.programs.bin`, `KYTY_PROGRAM_CACHE=1`):
  guarda la traducción de shaders, no solo la caché del driver. Elimina el ~20 % que gastaba
  nuestro recompilador.
- **Guardado periódico de la caché del driver** en un hilo propio (`DriverCacheSaver`).
- **Precompilación** (`KYTY_SHADER_PRECOMPILE=1`, `shaderPrecompile.*`): en el siguiente arranque
  re-compila en segundo plano lo que el juego usó.
- **Prefetch de pipelines** (`KYTY_PIPELINE_PREFETCH=1`, `KYTY_PIPELINE_PREFETCH_PROGRAMS=1`): 1–4
  hilos compilan el pipeline antes de que el dibujo lo necesite (`docs/PIPELINE-STUTTER.md`).
- **Fast-first** (`KYTY_PIPELINE_FAST_FIRST=1`, `pipelineFastFirst.h`): primero un pipeline sin
  optimizar (rápido de crear), luego el optimizado en segundo plano.
- **Diario/known pipelines** (`KYTY_PIPELINE_JOURNAL=1`, `KYTY_PIPELINE_KNOWN=1`).
- **GPL** (`KYTY_PIPELINE_LIBRARY=1`): experimento, apagado por defecto.

Límite reconocido por Jetsku: un shader totalmente nuevo sigue necesitando traducción y creación;
la primera partida nunca es 100 % libre de tirones sin saltarse dibujos.

### 3.4 Memoria, texturas y transferencias
Seguimiento de páginas sucias por rangos, epochs, lecturas sin drenar la GPU, cola DMA, pool de
imágenes nativas (`KYTY_NATIVE_IMAGE_POOL=1`), residencia por mip, DCC en GPU, fix del agua.

### 3.5 Corrección de shaders en AMD (cuelgues)
- **wave32 clusters** (`KYTY_WAVE32_CLUSTERS`): cada wave32 del juego se limita a sus 32 lanes
  cuando el host usa subgrupos de 64 (AMD).
- **Barrera LDS en `S_WAITCNT`** (`KYTY_LDS_WAITCNT_BARRIER=subgroup`).
- **LDS grande en buffer de GPU** (`KYTY_LDS_DEVICE_BUFFER`), arrays por lane reducidos
  (`KYTY_FUNCTION_ARRAY_SHRINK=1`), helper lanes sin bucles atómicos
  (`KYTY_PS_HELPER_ATOMICS_SKIP`).
- **Shaders que crashean el compilador AMD** sin optimizar + guardia (`KYTY_PIPELINE_NO_OPT_SHADERS`,
  `KYTY_PIPELINE_OPT_FAULT_GUARD`, `_PipelineCache/<juego>.noopt.txt`).
- Código: `src/graphics/shader/recompiler/CodegenOptions.*`, `frontend/translate/Control.cpp`,
  `backend/spirv/spirvEmitterFlow.cpp`.

### 3.6 Diagnóstico de cuelgues (clave para no colgar la PC a ciegas)
- **Breadcrumbs de GPU** (`gpuBreadcrumbs.*`, `diagnosticCheckpoints.cpp`): tras un device lost,
  el log imprime la operación "likely hang" (la que la GPU no terminó).
- **Hang watchdog** (`KYTY_HANG_WATCHDOG=auto`, `docs/HANG-WATCHDOG.md`).
- Salida fatal con `TerminateProcess` para que un driver no deje el proceso zombie.

## 4. Hoja de ruta hacia 60 FPS estables

1. **Línea base medible** en la RX 9070 XT con el preset: FPS medio, p95/p99 de frame time
   (`KYTY_HANG_TRACE_FRAME_TIMES=1` → `frames.csv`), tirones por compilación (`compiles.csv`).
2. **Estabilidad primero**: si hay `ErrorDeviceLost`, usar el "likely hang" de los breadcrumbs
   para arreglar el shader exacto, no a ciegas.
3. **Tirones**: probar `KYTY_PIPELINE_PREFETCH_THREADS` (2→4 en un i5-12400 de 12 hilos) y medir
   GPL (`KYTY_PIPELINE_LIBRARY=1`) en AMD, que soporta fast-link.
4. **Ritmo de cuadros**: presentación Mailbox/limitador a 60, medir p99.
5. **Re-entrenar PGO** con nuestra máquina/juego cuando cambiemos código (el perfil actual es de
   Jetsku, RTX 3090 + 7950X3D).
6. **Traer del oficial** solo lo revisado (catálogo de 76/143 commits upstream).

## 5. Cómo compilar

`C:\Users\Admin\kyty-scripts\build-scraids.bat`: LLVM 22.1.3 (`C:\LLVM-22.1.3`, la versión del
perfil PGO), código servido desde la unión `C:\kyty-src19` (ruta con la que se grabó el perfil),
IPO + PGO, origen `Fork`. Resultado en `_Build/scraids/install`.
