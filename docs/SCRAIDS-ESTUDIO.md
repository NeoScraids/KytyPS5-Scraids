# KytyPS5-Scraids: estudio de la base Jetsku (U59) y hoja de ruta

Documento de trabajo del fork. Explica **cómo** la base [Jetsku/KytyPS5](https://github.com/Jetsku/KytyPS5)
consigue tanto más rendimiento que el KytyPS5 oficial, dónde vive cada mecanismo y qué falta para la
meta: **juego fluido a 60 FPS en AMD (RX 9070 XT + i5-12400)**.

Rutas relativas a `src/graphics/` salvo que se indique. Merge-base con el oficial: `aa3b2cd5`.
Fuentes ampliadas: `docs/CHANGES-U59.md`, `docs/PIPELINE-STUTTER.md`, `docs/release-notes-history.md`,
`docs/CHANGE-CATALOG.md` (inventario de cada commit).

## 0. Qué costó

- ~640 commits propios y ~120.000 líneas en `src/` en **18 días** (2026-09-26 → 2026-10-13), más 52
  archivos de tests. ~80 % de esos commits los firman agentes de IA (Claude Code, Claude, Codex) dirigidos
  por Jetsku. Los 853 commits de `nmzik` son del KytyPS5 oficial (historia reescrita en el fork).
- El valor no está solo en el código: está en las mediciones A/B, las rondas de "AMD test" con jugadores
  y los experimentos descartados (p. ej. una regresión 65 → 61 FPS que revirtieron).

## 1. Medición inicial (oficial vs base Jetsku) en la RX 9070 XT

| | KytyPS5 oficial | Base Jetsku (2.ª partida, log activo) |
|---|---|---|
| ASTRO's PLAYROOM tras la intro | ~23 FPS, tirones, audio desfasado | 58,3 FPS medio, mediana 60 |
| Zona CPU Tower | `ErrorDeviceLost` + TDR (3 de 3 partidas) | Pasa sin errores |
| Hilo más cargado | CP al ~90 %: 70 % compilando pipelines en el driver AMD, 20 % traduciendo shaders | Trabajo repartido en 4 hilos |
| GPU 3D | ~40 % (esperando a la CPU) | ~62 % |

## 2. Configuración: `u59-preset.json`

**Casi todo lo nuevo está apagado en el código** y se enciende con variables `KYTY_*` que el launcher
lee de `u59-preset.json` (junto a `launcher.exe`). Sin ese archivo se mide otro emulador.
`build-scraids.bat` lo copia a `_Build/scraids/install`. Código: `src/common/u59Preset.*`.
Varias flags entran en el fingerprint de codegen: cambiarlas invalida `programs.bin`.

## 3. Cómo funciona (por subsistema)

### 3.1 Del juego a la GPU: cadena de 4 hilos (antes, uno solo)

```
hilo del juego ─► [cola] ─► CpSequencer (parsea PM4, dueño del estado de registros)
                              │ anillo de ops (SPSC) + snapshots → workers de DrawPrep
                              ▼
                  Resolver / hilo GPU (Exec*, cachés, CommandScheduler)
                              │ CommandStream (anillo SPSC de 16 MB, paquetes por valor)
                              ▼
                  CP recorder (vkCmd*, vkEnd)
                              │ QueueSubmissionBroker (≤256 en cola)
                              ▼
                  hilo "Vulkan queue submission" ─ vkQueueSubmit (≤64 por llamada) ─► GPU
```

- **Idea central: reservar el orden antes de diferir.** El tick/secuencia se asigna en el productor; cada
  etapa es una FIFO que copia valores (nunca punteros). Solo se re-sincroniza en drenajes explícitos
  (`Drain`, `WaitRecorded`, `DrainThroughLocked`).
- **Envío en cola y agrupado** (`host_gpu/queueSubmission.cpp`; `KYTY_SUBMISSION_MODE=queued`,
  `KYTY_SUBMISSION_COALESCE=1`): fusiona records con ticks consecutivos en un `VkSubmitInfo`;
  `preserve_completion` impide mover una señal que alguien espera. +5–12 % medido por Jetsku.
- **Grabador** (`host_gpu/renderer/commandStream.*`, `commandRecorder.*`; `KYTY_CP_RECORDER=1`): el CP
  codifica cada `vkCmd*` en un paquete; otro hilo lo reproduce. Si el CP necesita el command buffer
  directamente, abre una "direct window" que drena el anillo (`renderer/context.cpp:223-279`).
- **Secuenciador** (`guest_gpu/command_processor/cpOps.*`, `graphicsRun.cpp:4744`; `KYTY_CP_SEQ=1`):
  separa parseo de ejecución. El parseo solo se detiene en lo que de verdad lo decide: `WaitRegMem`,
  `WaitFlipDone`, predicación/condiciones, memoria escrita por la GPU (`GpuTouched`) y escrituras
  pendientes del CP (`CpWriteRange`). `KYTY_CP_SEQ_PREFETCH` parsea por delante en esperas de su propia
  etiqueta y adopta los draws solo si el hash coincide.
- **Ritmo de cuadros** (defaults del código): flush de EOP por lotes (`KYTY_EOP_FLUSH_BATCH=8`), flush
  anticipado si la GPU está ociosa, `WAIT_FLIP_DONE` suspende solo esa cola, `sceAgcSuspendPoint`
  acotado a 2 frames en vuelo, frame fence por frame, slicing de gráficos cada 128 draws.

### 3.2 Preparación de dibujos en paralelo

- **Especular en workers, certificar en el commit** (`renderer/drawPrep/drawPrep.cpp`): el CP copia un
  `RegisterSnapshot` a un anillo de 32; 6 workers (2 siempre activos) resuelven programas, recursos y un
  *binding plan* (`bindingPlan.h`) **sin efectos laterales** y graban qué memoria leyeron (*read-set*).
  En el commit, `CertificateHolds`/`Validate` comprueban con un log de coherencia que esa memoria no
  cambió; si cambió, se rehace por la ruta serial. Correcto por construcción.
- **Draw runs** (`drawPrep/drawRun.h`; `KYTY_DRAW_RUN*`): draws seguidos con la misma clave se hacen
  como delta (sin re-adquirir targets, texturas, pipeline ni estado dinámico).
- **SRT** (tablas de descriptores anidadas en memoria del juego): se cachea el **plan** de lectura
  (recetas compiladas, "tapes" aritméticas, lecturas por tramos; `shader/recompiler/ir/passes/SrtWalker.cpp`),
  nunca los valores; cada byte se vuelve a leer por la puerta de "página limpia".

### 3.3 Shaders y pipelines (tirones)

Antes: cada shader nuevo se traducía y cada pipeline se creaba de forma síncrona en el CP.

- **Caché de programas en disco** (`*.programs.bin`; `KYTY_PROGRAM_CACHE=1`): guarda SPIR-V + metadatos por
  fuente/permutación con claves exactas (todo el código del shader). Identidad = `CodegenFingerprint()` +
  GPU + driver: **cada build nueva la invalida** (una partida en frío).
- **Diario + precompilación al arrancar** (`*.shaders.journal`; `KYTY_SHADER_PRECOMPILE=1`): guarda las
  *entradas* de traducción (sobrevive a cambios del traductor) y al arrancar las reproduce en 2 hilos de
  baja prioridad.
- **Fast-first** (`renderer/pipeline/pipelineFastFirst.h`; `KYTY_PIPELINE_FAST_FIRST=1`): crea el pipeline
  **sin optimizar** al instante y compila el optimizado en 2 hilos; lo cambia subiendo
  `m_pipeline_generation`; el viejo se destruye a los 60 s.
- **Known pipelines / pipeline journal** (`KYTY_PIPELINE_KNOWN`, `KYTY_PIPELINE_JOURNAL`): al arrancar
  reconstruye en 4 hilos los pipelines de la sesión anterior dentro de la caché del driver.
- **DriverCacheSaver**: guarda la caché del driver en un hilo propio (cada ≥60 s, ≥32 nuevos).
- **Menos variantes**: `NormalizePipelineKey` (anula estado irrelevante), estado dinámico (cull, front
  face, depth bounds), layouts internados, IDs SPIR-V deterministas.
- **GPL** (`pipelineLibrary.h`; `KYTY_PIPELINE_LIBRARY`): **apagado**. No soporta mesh/teselación
  (muy usados en PS5), crea las librerías en el CP y no es bit-exacto. Tu driver soporta GPL + fast link.

### 3.4 Memoria y coherencia

- Log de rangos sucios con *epochs* en vez de escanear todo (`host_gpu/memoryTracker.cpp`,
  `cache/bufferCache.cpp`; `KYTY_BDA_INCREMENTAL_SYNC=1`). Invariante: una escritura de la CPU siempre
  invalida la copia de la GPU.
- Re-protección de páginas diferida al hilo DMA (`host_gpu/pageManager.cpp`; `KYTY_DEFER_PROTECTS=all`).
- Lecturas del juego que esperan solo al tick que produjo esos bytes (no drenan la GPU), segunda cola,
  DMA de 64 MiB para uploads grandes, pool de imágenes nativas, residencia por mip.

### 3.5 Por qué se colgaba tu GPU y cómo se arregló (AMD sí, NVIDIA no)

| Arreglo | Causa raíz | Código / flag |
|---|---|---|
| wave32 clusters | AMD ejecuta VS/PS/mesh en subgrupos de 64; dos waves de 32 del juego compartían uno y los bucles "waterfall" leían lanes ajenas y no terminaban → TDR | `shader/recompiler/backend/spirv/spirvEmitterHelpers.cpp:139-190`; `KYTY_WAVE32_CLUSTERS=auto` |
| Barrera LDS en `S_WAITCNT` | Barrera de alcance workgroup dentro de ramas a las que otras waves no llegan → deadlock en AMD | `frontend/translate/Control.cpp:216-244`; `KYTY_LDS_WAITCNT_BARRIER=subgroup` |
| Atómicos en helper lanes | Bucles CAS emulados que en helper lanes nunca convergen en AMD | `spirvEmitterInternal.h:749-768`; `KYTY_PS_HELPER_ATOMICS_SKIP` |
| LDS en buffer de VRAM | AMD permite 32 KiB de memoria compartida (NVIDIA 48) | `CodegenOptions.cpp:174-192`; `KYTY_LDS_DEVICE_BUFFER=auto` |
| Arrays por lane reducidos | Array de 8192 dwords por lane → ~2 MiB de scratch por wave en AMD | `host_gpu/vulkanCommon.cpp:150-195`; `KYTY_FUNCTION_ARRAY_SHRINK` |
| Buffers de solo lectura | Rendimiento: cargas escalares en AMD (3,45 → 2,76 ms en una iluminación) | `KYTY_READONLY_BUFFERS` |
| PR #25/#26/#27 | Crash del compilador AMD en el shader de tiling; placeholder de depth con formato de color; VS que no escribía un slot que el PS leía (basura en AMD) | `host_gpu/shaders/gpu_tiler_standard64_3d.inc`, `renderer/pipeline/descriptors.cpp:544,580`, `shader/shaderPixelParameter.cpp` |

Nuestro intento previo en el oficial (pedir subgrupo de 32 al driver) no bastaba: el driver AMD no
permite exigir 32 en vertex shaders y el resto de causas seguía presente.

### 3.6 Diagnóstico

- Breadcrumbs de GPU (`host_gpu/gpuBreadcrumbs.*`): tras un device lost el log dice la operación
  "likely hang".
- `KYTY_HANG_TRACE=1` + `KYTY_HANG_TRACE_FRAME_TIMES=1` → `_HangTrace/<fecha>/frames.csv`,
  `compiles.csv` (tiempos por cuadro y esperas por compilación); `KYTY_PIPELINE_STATS=1` → línea de
  pipelines cada 10 s.
- **No dejar `printf` a archivo para jugar**: escribió ~140 KB/s (100 MB en minutos) y genera tirones.

## 4. Hallazgos para AMD (candidatos del preset Scraids)

1. **Fast-first forzado en AMD.** El preset usa `KYTY_PIPELINE_FAST_FIRST=1`, pero el propio código
   (`pipelineFastFirst.h:48-51`) avisa de que fuera de NVIDIA el código sin optimizar puede ir mucho más
   lento (y lo señala como sospechoso de un device lost en una RX 6800S). Además, con fast-first activo
   el **prefetch de pipelines queda apagado** (`pipelineCache.cpp:5624`) aunque el preset lo pida.
   Prueba A/B: `KYTY_PIPELINE_FAST_FIRST=nvidia` + `KYTY_PIPELINE_PREFETCH_THREADS=4`.
2. **Sparse residency de texturas** (`KYTY_TEXTURE_SPARSE_RESIDENCY=1`, `KYTY_BDA_PAGETABLE_SPARSE=1`):
   el kit de pruebas AMD de Jetsku la desactiva en sus escenarios. Prueba A/B con `0`.
3. **Crear pipelines fuera de `m_mutex`** (`pipelineCache.cpp:7197-7272`): hoy el build rápido se hace con
   el lock tomado y serializa a los workers de preparación.
4. **GPL en AMD con librerías precompiladas** desde los diarios (o `VK_EXT_shader_object`), añadiendo
   mesh a pre-raster.
5. **Re-entrenar PGO** con nuestro juego y máquina cuando cambiemos código (el perfil actual es de
   RTX 3090 + 7950X3D).

## 5. Método de trabajo

- Un cambio por prueba; medir FPS medio, % de tiempo ≥58 FPS, bajones (<45) y crecimiento de caché.
- Comparar siempre con la misma caché (2.ª partida o más) y sin log a archivo.
- Los cambios de flags se prueban primero en `_Build/scraids/install/u59-preset.json`; si ganan, pasan a
  `tools/u59-preset.json` (preset AMD de Scraids) con su medición en el commit.

## 6. Cómo compilar

`C:\Users\Admin\kyty-scripts\build-scraids.bat`: LLVM 22.1.3 (`C:\LLVM-22.1.3`, versión del perfil
PGO), código servido desde la unión `C:\kyty-src19` (ruta con la que se grabó el perfil), IPO + PGO,
origen `Fork`. Resultado en `_Build/scraids/install`.
