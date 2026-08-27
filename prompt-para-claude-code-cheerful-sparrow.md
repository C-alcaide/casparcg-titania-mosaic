# Plan — Completar investigación técnica: sincronización PTS CasparCG (MAIN/BACKUP)

## Contexto

`C:\OVERON\casparcg-titania-mosaic\docs\CASPARCG_PTS_SYNC_INVESTIGATION.md` pide una auditoría
exhaustiva, con evidencia de código, de cómo trata CasparCG 2.5 el PTS de múltiples señales
(especialmente pares MAIN/BACKUP) al componerlas en un mosaico — y exige validar o refutar la
hipótesis de que dos producers/layers "tirados en el mismo tick" pueden mostrar contenido de
instantes distintos, sin asumir nada por conocimiento general de FFmpeg/CasparCG. Los dos CSV de
ffprobe (`main_7101.csv`, `bckp_8102.csv`) muestran evidencia real: 49 filas solapadas donde
`MAIN[n+1].PTS == BACKUP[n].PTS` exacto (delta 0), es decir, BACKUP es la misma timeline que MAIN
leída una fila más tarde.

Regla explícita del usuario: **no tocar código todavía**. El entregable de esta fase es un único
documento nuevo, `CASPARCG_PTS_SYNC_ANALYSIS_RESULT.md`, con la estructura A–G y las 24 preguntas
obligatorias que exige el documento fuente (§27–28).

Esta sesión ya ha ejecutado una investigación profunda (2 agentes Explore sobre el C++ de
CasparCG + 1 agente Plan de contraste/verificación + lecturas directas del lado C#/Docker/AMCP de
este repo + verificación manual de los hallazgos más críticos). Lo que sigue resume lo ya
confirmado con evidencia y define el trabajo que falta antes de poder escribir el documento final.

## Restricción dura para toda la fase de ejecución

- Cero cambios de código — ni en `casparcg-titania-mosaic` (C++), ni en `core2-mosaic` (C#), ni en
  `mosaic` (osc-bridge.js). Todo lo que en este plan se describe como "instrumentación propuesta"
  se documenta como propuesta dentro del `.md`, no se implementa.
- Único artefacto a crear: `C:\OVERON\casparcg-titania-mosaic\docs\CASPARCG_PTS_SYNC_ANALYSIS_RESULT.md`.
- Cada conclusión relevante del documento final debe llevar fichero + función/clase + líneas
  aproximadas + explicación (exigencia explícita del documento fuente, §27.D).
- Separar explícitamente hechos demostrados / hipótesis / cuestiones pendientes (§27.C).

---

## 1. Ya confirmado con evidencia de código (no repetir investigación, solo verificar puntualmente si hace falta)

### Dónde entra y dónde se pierde el PTS (preguntas 1–4, 6)

| Etapa | Fichero:línea | Qué pasa |
|---|---|---|
| Demux | `av_input.cpp:54` (`av_read_frame`) | `AVPacket` con pts/dts crudos, cola `tbb::concurrent_bounded_queue` cap. 256 (`av_input.h:55`) |
| `AVStream.time_base` | `av_producer.cpp:179` (Decoder ctor), `537-538/565-566` (Filter) | Se lee y se usa para `av_rescale_q` |
| Decode | `av_producer.cpp:240` | `av_frame->pts = av_frame->best_effort_timestamp;` — reordenamiento B-frame delegado 100% a libavcodec, sin buffer propio |
| Normalización | `av_producer.cpp:954,961,970` | `frame.pts = av_rescale_q(video->pts, tb, TIME_BASE_Q) - start_time` (`start_time` = `input_->start_time`, **por producer**, independiente entre MAIN y BACKUP) |
| Struct interno | `av_producer.cpp:64-73` (`struct Frame`) | Aquí SÍ vive `.pts/.start_time/.duration/.frame_count` — sobrevive hasta justo antes de cruzar a `core::` |
| **Se pierde aquí** | `av_util.cpp:46-111` (`make_frame()`) | Grep de todo el fichero por `"pts"`: **0 resultados**. Solo copia píxeles/audio. Llamada desde `av_producer.cpp:974-975` |
| Frame CasparCG | `core/frame/frame.h:22-113` | `mutable_frame`/`const_frame`: grep de `pts\|PTS\|timestamp` en **todo** `src/core`: **0 resultados** |

**Conclusión de hecho, no hipótesis**: el PTS existe de forma completa y correcta hasta el límite
exacto de `make_frame()` / cruce a `core::frame_producer`, y a partir de ahí desaparece por
completo del dato que ve Layer/Stage/Mixer.

### `next_frame()`, interfaz `frame_producer`, "mismo tick" (preguntas 5, 7, 9–12)

- `frame_producer::receive(video_field field, int nb_samples)` (`frame_producer.h:59,80`) — **sin
  parámetro de tiempo/PTS/reloj de ningún tipo.** Confirmado por los dos agentes Explore de forma
  independiente.
- `stage::impl::operator()` (`stage.cpp:119-232`) corre entero dentro de UN `caspar::executor` (un
  solo hilo, `stage.cpp:58`), con un `for` secuencial sobre layers (164-200). "Mismo tick" =
  "misma iteración de este bucle, mismo `frame_counter_`" — cero comparación temporal entre
  layers.
- `layer::impl::receive()` (`layer.cpp:106-149`) no pasa tiempo a `foreground_->receive()`; si el
  producer no tiene nada listo, sustituye en silencio `last_frame(field)` (línea 128) — repite el
  frame anterior, sin avisar.
- Mixer (`mixer.cpp:64-145`, `image_mixer.cpp:294-322`) solo recibe píxeles+geometría+audio vía
  visitor — **no puede** estructuralmente decidir por PTS, porque ese dato ya no existe cuando
  llega. Responde P11 (sí, es posible resolverlo sin tocar el mixer — de hecho hay que resolverlo
  *antes* del mixer, no dentro).
- No existe ningún concepto de sync/genlock/SyncGroup entre producers en todo el árbol (`grep
  sync|genlock|framesync|SyncGroup|align` sobre `modules/ffmpeg` + `core`: nada relevante).
  `has_synchronization_clock()` (`frame_consumer.h:59`) es paceo de **salida/hardware** (Decklink/
  Bluefish), no tiene relación con alinear producers/layers entre sí.

### Corrección importante — `frame_number()` SÍ deriva de PTS en la clase real usada (pregunta 10)

Verificado personalmente en `ffmpeg_producer.cpp:119-122`:
```cpp
std::uint32_t frame_number() const override
{
    return static_cast<std::uint32_t>(producer_->time() - producer_->start());
}
```
`producer_->time()` = `av_rescale_q(frame_time_, TIME_BASE_Q, format_tb_)` (`av_producer.cpp:1113`),
y `frame_time_` se fija en `next_frame()` desde el `.pts` normalizado del `Frame` (`av_producer.cpp:1080`).
Es decir: **para todo input UDP MPEG-TS de este pipeline, `frame_number()` sí refleja PTS real**,
vía override de la clase concreta (no la base `frame_producer.h:90`, que es un contador ingenuo).
Esto hace **consistente con el código** la explicación del commit `888f374c` ("frame_number() sin
-copyts no refleja el PTS real") — no hay que presentarla como sospechosa en el informe final.

### Hallazgo mayor — ya existe un canal de instrumentación en producción, probado, con PTS real (pregunta 8, Experimento 1/2)

Cadena completa trazada y verificada:
```
av_producer.cpp:1080  frame_time_ = buffer_[0].pts        (por producer, cada tick)
av_producer.cpp:1007  state_["file/time"] = time()/fps     (PTS-derivado, ver arriba)
ffmpeg_producer.cpp:205  state() override → producer_->state()
layer.cpp:132         state_["foreground"] = foreground_->state()
stage.cpp:222         state["layer"][id] = layer.state()
video_channel.cpp:179/189  state["stage"]=...; tick_(state_)   (cada tick, todo el canal)
shell/server.cpp:292  osc::client::send(...)
```
Dirección OSC resultante: `/channel/N/stage/layer/L/foreground/file/time` — coincide byte a byte
con el regex ya en producción `osc-bridge.js:344`
(`/^\/channel\/(\d+)\/stage\/layer\/(\d+)\/foreground\/file\/time$/`), y su comportamiento está
**confirmado empíricamente en producción** (`Context/mosaic-freeze-watch-senal-congelada.md:46-52`,
sesión 2026-08-24 en mosaic2: cambia cada tick con señal sana, se congela exacto bajo `PAUSE`).

**Implicación práctica clave para el informe**: el Experimento 2 del documento fuente
(`tick | MAIN PTS | BACKUP PTS | delta`) puede ejecutarse **hoy, en producción, sin tocar
CasparCG** — solo extendiendo `osc-bridge.js` (mismo patrón que freeze-watch) para comparar
`file/time` de un par Main/Backup cada tick en vez de cada 5s. Esto se documenta como propuesta
("Tier 0"), no se implementa esta fase.

**Matiz que debe quedar en el informe** (evidencia de código, no opinión): en EOF de señal en vivo,
`av_producer.cpp:1051-1058` avanza `frame_time_` artificialmente (`frame_time_ += frame_duration_`)
aunque se repita el mismo frame — mismo progreso de PTS no implica mismo contenido visible en ese
caso concreto. Responde con evidencia a la pregunta 22 del documento.

### GOP / B-frames — la premisa del documento ya no aplica al estado actual (preguntas 18–19)

`MosaicSignalService.cs` (`BuildFfmpegCommands`, ~960-1089), producción, ambas ramas GPU/CPU:
`-g 25 -keyint_min 25 -x264-params scenecut=0` (GOP fijo ~1s) y **`-bf 0` ya activo** — el
documento fuente pregunta "¿tendría valor probar bf=0?" y la respuesta es que ya es el estado de
producción. Nada en `av_producer.cpp`/`ffmpeg_producer.cpp` condiciona ningún comportamiento al
tamaño de GOP. Conclusión para el informe: aumentar GOP no puede ayudar a esto (no hay ninguna ruta
de código que lo relacione con sincronización entre producers) y no hace falta experimentarlo.

### Fixes recientes en este repo (C#) — ya resolvieron la desalineación de *arranque* sin tocar CasparCG

Cuatro commits, en orden (`git log`/`git show` ya revisados en detalle):
1. `5a42ff10` — invierte el orden: docker run (Main+Backup) → espera → `PLAY` conjunto en un solo
   `BEGIN/COMMIT` AMCP. Causa raíz documentada por el propio autor: sin esto, cada layer arranca a
   reproducir en cuanto le llega su primer frame, en un instante arbitrario, y esa ventaja queda
   fija el resto de la sesión porque CasparCG no realinea layers ya arrancados — 100% consistente
   con "no existe coordinación entre producers en ningún punto" confirmado arriba.
2. `888f374c` — añade `-copyts -avoid_negative_ts make_zero` al Docker ffmpeg. Reduce el desfase de
   arranque de ~25-50 frames a ~1 frame (verificado a mano en mosaic2).
3. `e7355d9b` — sube `DOCKER_WARMUP_DELAY_MS` de 1000 a 4000, mismo motivo que (1).
4. `eb44def0` — espera confirmación real de `docker rm` (no un `Sleep(1s)` a ciegas) antes de
   recrear el contenedor, cerrando una carrera adicional.

**Riesgo arquitectónico a documentar (derivado del código, no medido aún)**: como no existe ningún
mecanismo de corrección de deriva en tiempo de ejecución (todo lo anterior solo alinea el
*arranque*), y un underrun de cualquiera de los dos productores repite en silencio el último frame
(`layer.cpp:126-129`), si MAIN y BACKUP sufren alguna vez un stall/drop diferencial después de
arrancar, quedarán desalineados permanentemente hasta el próximo restart, sin que nada lo detecte.
Esto es precisamente lo que el Tier 0 de instrumentación (arriba) permitiría medir.

### Hallazgos colaterales a mencionar en el informe (fuera del núcleo PTS, pero relevantes/riesgo)

- Existen **tres implementaciones distintas** de construcción de comandos ffmpeg en este repo C#:
  `CasparCgFfmpegService.BuildFfmpegCommand(WithVumeter)` (código muerto/roto, con un literal
  `"ffmpegInputUrl"` pasado como URL), `MosaicRollingRestartService.BuildFfmpegCommands` (copia
  propia, **sin** `-copyts` — no recibió el fix de `888f374c`, pero ese servicio entero está
  confirmado sin invocar todavía, ver su propio comentario de cabecera) y
  `MosaicSignalService.BuildFfmpegCommands` (la real, con el fix). Mencionar como riesgo latente
  (footgun si `MosaicRollingRestartService` se llega a conectar sin portar el fix) — no corregir
  en esta fase, es fuera de alcance (solo investigación).
- `Context/mosaic-freeze-watch-senal-congelada.md:12-13` afirma la existencia de un `stall_timer`
  ("parche propio") que reconecta tras 5s sin frames. Grep de todo `casparcg-titania-mosaic/src`
  por `stall_timer`: **0 resultados**. El único patch local confirmado
  (`patches/osc-audio-per-layer-on-v2.5.0-stable.patch`, commit `2d29225`) declara explícitamente
  no tocar `av_input`/`av_producer`. **No repetir la afirmación del stall_timer como hecho en el
  informe final** sin verificación independiente — dejarla como "cuestión pendiente" si sale a
  relucir. (Aviso aparte para el usuario: puede ser una entrada desactualizada/incorrecta en esa
  nota de contexto — no se toca ese documento en esta tarea, pero merece revisión suya.)
- Los puertos `7101`/`8102` de los CSV no coinciden con `LocalPortMain`/`LocalPortBckp`
  (`20000+Id`/`30000+Id`, `MosaicInput.cs:48,51`) ni con ningún patrón localizable en los repos.
  Lectura más probable: son la señal de **origen** (pre-Docker), coherente con que `888f374c` y
  `eb44def0` describen haber comprobado por separado "el origen" y "los dos UDP locales" con
  ffprobe. Etiquetar como inferido en el informe, no como confirmado — o preguntar directamente al
  usuario si se quiere cerrar del todo.

---

## 2. Trabajo que falta antes de poder escribir el documento final

Lecturas puntuales — con alcance ya muy acotado por lo anterior, no es exploración abierta:

**Con peso en el informe (bloquean preguntas obligatorias concretas):**
1. `src/core/mixer/mixer.cpp` + `src/accelerator/ogl/image/image_mixer.cpp` — lectura completa
   (hoy la conclusión "el mixer no puede decidir por PTS" está probada por eliminación vía grep en
   `core/frame`, no leyendo el propio mixer). Cierra P11/§7.7 con rigor.
2. `src/core/mixer/audio/audio_mixer.cpp` + ruta `VU_LAYER` en `MosaicSignalService.cs` — cierra
   P16 (¿el mosaico usa audio real de cada señal o solo vúmetro?).
3. `src/protocol/amcp/**` — confirmar si `file/time` (u otro campo temporal) se expone también por
   `INFO` AMCP síncrono, y si existe algún comando de "seek a PTS absoluto". Cierra P8/P10 del lado
   de herramientas, y refuerza la comparación de arquitecturas (opción G).
4. `src/accelerator/ogl/**` (pooling de texturas/buffers GPU) — cierra P17 (impacto GPU/RAM/CPU),
   no leído todavía en detalle.
5. Intentar cerrar la afirmación del `stall_timer` (arriba) — un grep adicional ya descartó que
   viva en `casparcg-titania-mosaic/src`; comprobar si existe en otro sitio (parche no mergeado,
   config, doc de otro repo) o marcarlo definitivamente como pendiente/posible error de una nota
   anterior.

**Sin bloquear, mejoran el informe si hay margen:**
6. `transition_producer.cpp`/`sting_producer.cpp` completos — documentar formalmente el patrón
   "producer que envuelve producers" como precedente reutilizable (mismo layer, sin PTS).
7. `osc-bridge.js` (lectura completa, no solo grep) — para especificar la propuesta Tier 0 con
   precisión de código real, no solo de patrón.

---

## 3. Estructura y contenido previsto de `CASPARCG_PTS_SYNC_ANALYSIS_RESULT.md`

Sigue literalmente la estructura pedida en el documento fuente (§27 A–G), ya con el contenido
sustancial definido por lo confirmado arriba:

- **A. Pipeline real** — diagrama basado en código (demux → decode → make_frame → frame_producer →
  layer → stage → mixer → output), con clase/fichero/función/hilo/metadata temporal disponible en
  cada transición (ya trazado punto 1).
- **B. Tabla PTS/DTS** — por etapa: disponible/no, time base, se modifica, se conserva a la
  siguiente etapa (tabla del punto 1, ampliada).
- **C. Causa probable del desfase** — separar explícitamente: hechos demostrados (tabla arriba +
  diagrama de threads), hipótesis (p. ej. si la deriva post-arranque ocurre en la práctica — no
  medido), pendientes (stall_timer, puertos CSV).
- **D. Evidencia de código** — todas las citas fichero:función:líneas ya recopiladas.
- **E. Comparación de arquitecturas** — tabla A–G ya elaborada (ver §4 de este plan), con los
  criterios exigidos por el documento (complejidad, riesgo, latencia, cambio API, rendimiento,
  compatibilidad, capacidad real de sync por PTS, tolerancia a pérdida de señal, mantenibilidad).
- **F. Recomendación** — ver §4.
- **G. Plan incremental** — FASE 0 (instrumentación Tier 0/Tier 1, ver §5) → FASE 1 (medir 2
  producers reales, MAIN/BACKUP) → FASE 2 (si hay deriva sostenida: diseño mínimo de
  coordinador/SyncManager, opt-in) → FASE 3 (gestión de errores/discontinuidades) → FASE 4 (N
  inputs) → FASE 5 (optimización). Cada fase con criterio de aceptación explícito.
- Tabla final con las **24 preguntas obligatorias** (§28), estado y evidencia — ya construida y
  verificada en esta sesión, lista para incorporar casi literal.

### 4. Comparación de arquitecturas (contenido ya elaborado, para incorporar)

Recomendación: **A primero siempre** (instrumentación, no negociable — el propio documento lo
exige antes de tocar nada). Con esos datos:
- Si el desfase resulta pequeño y estable → **Opción G** (realineación vía AMCP/OSC externo,
  mismo patrón operativo que el freeze-watch ya probado) es suficiente y más barata que C.
- Si el desfase deriva o es significativo y sostenido → **Opción C (SyncManager)** como segunda
  fase — única que preserva Main/Backup como layers independientes (de lo que dependen
  `StopSignal`, warning overlay, freeze-watch) añadiendo capacidad real de sync.
- **Descartadas con motivo**: E (sync externo FFmpeg/GStreamer — el propio documento la relega a
  POC, y rompe la independencia operativa por-slot ya construida en el C#); F (delay fijo — el
  TODO de discontinuidades sigue sin resolver en `av_producer.cpp:102`, nada garantiza que el
  offset sea constante); B en solitario (el propio documento ya señala que un producer seguiría
  sin conocer el estado de los demás — es un bloque de construcción de C/D, no una opción propia).

### 5. Propuesta de instrumentación mínima (contenido ya elaborado, para incorporar)

- **Tier 0** (recomendado empezar aquí, coste ~cero, sin recompilar CasparCG): extender
  `osc-bridge.js` para diffear `file/time` de un par Main/Backup cada tick en vez de cada 5s —
  ejecuta el Experimento 2 del documento con datos de producción reales.
- **Tier 1** (solo si Tier 0 no da resolución suficiente): puntos de log exactos ya identificados —
  `av_producer.cpp:240` (PTS crudo + decode time), `av_producer.cpp:1079-1081` (PTS normalizado +
  delivery time, justo antes de `buffer_.pop_front()`), `video_channel.cpp:132` (tick de canal),
  identificación de layer vía `state_["file/path"]` (`av_producer.cpp:782`, ya contiene el puerto
  UDP local, que ya es único por Main/Backup vía `LocalPortMain`/`LocalPortBckp`) — sin nuevos
  parámetros. Cumple la restricción del documento de no cambiar interfaces públicas.

---

## Verificación

- Antes de dar el informe por bueno: contrastar la tabla de 24 preguntas contra el documento
  fuente (todas deben tener estado explícito: respondida/parcial/pendiente, ninguna en blanco).
- Abrir al azar 6-8 citas fichero:línea del informe final y confirmar que el código citado dice
  literalmente lo que el informe afirma (ya se ha hecho para la más crítica —
  `ffmpeg_producer.cpp:119-122` — durante esta sesión).
- Confirmar que el informe separa con claridad hechos/hipótesis/pendientes en la sección C, y que
  no repite la afirmación del `stall_timer` como hecho.
- `git status` en `casparcg-titania-mosaic`, `core2-mosaic` y `mosaic` debe seguir sin cambios de
  código — solo el `.md` nuevo en `casparcg-titania-mosaic/docs/`.
