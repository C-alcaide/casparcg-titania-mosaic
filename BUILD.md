# Origen y build de este árbol

Este repositorio es el árbol fuente real de CasparCG que se compila y despliega en producción
(mosaic1, mosaic2, mosaic4, mosaic5). Documentado aquí porque el árbol se generó originalmente
descargando un `.zip` (sin histórico git de CasparCG) y aplicando parches propios a mano — sin
este fichero, el origen y el porqué de cada cambio no se puede reconstruir solo mirando el árbol.

## Base

- **CasparCG oficial `v2.5.0-stable`**, descargado como fuente sin histórico git desde
  `https://github.com/CasparCG/server/archive/refs/tags/v2.5.0-stable.zip` (por eso este repo
  empieza en un commit "vacío" sin relación con el histórico oficial de CasparCG).
- Confirmado por AMCP (`VERSION` → `2.5.0 N/A Stable`) que coincide con las constantes de
  versión (`CONFIG_VERSION_MAJOR/MINOR/TAG`) del tag `v2.5.0-stable` en el repo `server`.

## Parches aplicados, en orden

1. **`patches/osc-audio-per-layer-on-v2.5.0-stable.patch`** (commit `2d29225`) — expone el pico
   de audio por capa vía OSC (`/channel/N/mixer/layer/L/audio/peak/CH`), necesario para el
   vúmetro por capa. 7 ficheros: `src/core/mixer/audio/audio_mixer.cpp/h`,
   `src/core/mixer/mixer.cpp/h`, `src/core/producer/stage.cpp/h`, `src/core/video_channel.cpp`.
   No toca `av_input.cpp`/`av_producer.cpp` — el `.deb` oficial (Fase 4B de
   `setup-desde-cero.sh`) no trae este desglose por capa.
2. **Fix SIGILL en `src/modules/html/html.cpp`** (commit `74d19ca`) — mitiga un crash recurrente
   de CasparCG (`OnMemoryDump` en `malloc_dump_provider.cc`, dentro de `libcef.so`): el sistema
   de tracing de memoria interno de Chromium ("memory-infra"), disparado por un timer periódico,
   no por ninguna acción de Titania/CasparCG. Causa raíz localizada con `gdb`/coredump real
   (backtrace: `OnMemoryDump()` en `malloc_dump_provider.cc:465`), no una suposición. Mitigación:
   en `OnBeforeCommandLineProcessing`, tras `remote-allow-origins`, se añaden
   `--disable-features=MemoryInfra`, `--disable-component-update` y
   `--disable-background-networking`. Verificado en runtime (vía `/proc/<pid>/cmdline` de los
   subprocesos CEF reales) y estable sin recurrencia desde el 2026-07-31.

## Receta de build (misma que `build-and-deploy.sh` / Fase 4C de `setup-desde-cero.sh`)

```bash
mkdir -p build && cd build
cmake ../src \
  -DCMAKE_BUILD_TYPE=Release \
  -DUSE_SYSTEM_CEF=OFF \
  # ... resto de flags, ver server/tools/linux/instalacion/setup-desde-cero.sh fase4c_build_osc_patch()
cmake --build . --parallel "$(nproc)"
cmake --install . --prefix ../staging
```

Tras el build: copiar `staging/bin/casparcg`, `staging/lib/*.so*` y los recursos de CEF
(`icudtl.dat`, `*.pak`, `v8_context_snapshot.bin`, `libcef.so`, `chrome-sandbox`, `locales/`...)
al directorio final, y `patchelf --set-rpath '$ORIGIN'` sobre el binario para que sea
autocontenido. Detalle completo en `server/tools/linux/instalacion/setup-desde-cero.sh`
(función `fase4c_build_osc_patch`).

## Despliegue real

- **mosaic1, mosaic2, mosaic4, mosaic5**: unificados en `~/mosaic/casparcg/bin/casparcg`
  (`ExecStart=` de `casparcg.service` apunta ahí, no a `/usr/bin/casparcg-server-2.5`) — este
  árbol, con los 2 parches de arriba aplicados. Confirmado por `ExecStart=` real en mosaic2 y
  mosaic4 (2026-08-26).
- **mosaic3**: queda fuera a propósito — sigue en una línea distinta ("fork propio", rama
  `development_gpu` del repo `server`, versión `2.6.0 Dev`), no en este árbol. Migrarlo o no
  quedó pendiente de decisión desde antes de vacaciones (ver memoria
  `mosaic4-casparcg-sigill-crash.md`) — no resuelto todavía.
