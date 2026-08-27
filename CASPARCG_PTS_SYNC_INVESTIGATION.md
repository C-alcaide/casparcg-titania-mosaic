# Investigación técnica: sincronización PTS de múltiples señales en CasparCG 2.5

## 1. Objetivo

Analizar en profundidad el código fuente completo de CasparCG 2.5 para determinar cómo implementar un mosaico/multiviewer en el que varias señales de vídeo —especialmente pares MAIN/BACKUP— se presenten sincronizadas temporalmente.

El requisito no es simplemente arrancar varios producers al mismo tiempo. El objetivo es que, cuando varias señales pertenecen a una misma línea temporal, CasparCG muestre simultáneamente los frames correspondientes al mismo instante de presentación, utilizando PTS u otra referencia temporal adecuada.

**No implementar cambios todavía.**

La primera fase debe ser exclusivamente:

1. auditoría del pipeline actual;
2. validación de hipótesis;
3. identificación exacta del punto donde se pierde o deja de utilizarse la información temporal;
4. propuesta de arquitecturas posibles;
5. comparación de riesgos, complejidad, latencia y mantenibilidad.

---

## 2. Contexto funcional

El sistema pretende aproximarse funcionalmente a un multiviewer profesional tipo EasyTools:

- múltiples entradas de vídeo;
- señales MAIN y BACKUP;
- composición en mosaico;
- overlays, etiquetas y VU meters;
- monitorización en tiempo real;
- posibilidad de detectar o evitar desfases temporales entre señales relacionadas.

Arquitectura actual simplificada:

```text
Señal multicast original
        |
        v
Docker FFmpeg independiente por señal
        |
        v
UDP MPEG-TS
        |
        v
CasparCG FFmpeg producer
        |
        v
Layer de CasparCG
        |
        v
Mixer
        |
        v
Mosaico
```

Cada señal se procesa actualmente de manera independiente.

El problema observado es que dos señales que deberían representar el mismo contenido temporal, por ejemplo MAIN y BACKUP, pueden verse desincronizadas dentro del mosaico.

---

## 3. Evidencia obtenida con ffprobe

Se han capturado PTS/DTS de una señal MAIN y su BACKUP.

Ficheros utilizados:

- `main_7101.csv`
- `bckp_8102.csv`

Ejemplo MAIN:

```text
PTS_time      DTS_time
74487.7048    74487.5448
74487.6248    74487.5848
74487.6648    74487.6248
74487.7848    74487.6648
74487.7448    74487.7048
74487.8248    74487.7448
...
```

Ejemplo BACKUP:

```text
PTS_time      DTS_time
74487.6248    74487.5848
74487.6648    74487.6248
74487.7848    74487.6648
74487.7448    74487.7048
74487.8248    74487.7448
...
```

### Resultado de la comparación

Los CSV contienen 50 registros cada uno.

Durante los 49 registros solapados se cumple:

```text
MAIN[n + 1].PTS == BACKUP[n].PTS
MAIN[n + 1].DTS == BACKUP[n].DTS
```

con diferencia numérica:

```text
Delta PTS = 0
Delta DTS = 0
```

Por tanto, en esta captura concreta MAIN y BACKUP parecen compartir exactamente la misma timeline de timestamps.

La captura de BACKUP simplemente comienza un registro más tarde que MAIN.

---

## 4. Framerate observado

Los DTS avanzan de forma estable en pasos de:

```text
0.040 s
```

lo que corresponde a:

```text
25 fps
```

Los PTS no aparecen monotónicamente en orden de lectura debido al reordenamiento típico producido por B-frames.

Por tanto hay que distinguir claramente:

- orden de decodificación / DTS;
- orden de presentación / PTS;
- orden en el que CasparCG entrega finalmente frames al mixer.

Para sincronización visual, la referencia relevante debería ser el instante de presentación, no simplemente el orden en el que los paquetes o frames fueron recibidos/decodificados.

---

## 5. Hipótesis principal que hay que validar en el código

La hipótesis actual es:

> Cada `ffmpeg_producer` de CasparCG mantiene su propia cola, decodificador y timeline. Aunque FFmpeg conoce el PTS de los frames, CasparCG puede terminar consumiendo de cada producer el siguiente frame disponible de forma independiente, sin coordinar los PTS entre producers pertenecientes a diferentes layers.

En un tick de salida podría ocurrir conceptualmente:

```text
MAIN producer   -> frame PTS 74487.7848
BACKUP producer -> frame PTS 74487.6248
```

y ambos frames terminar formando parte del mismo frame del mixer.

Eso produciría desfase visual aunque:

- ambos streams tengan PTS correctos;
- ambos tengan exactamente la misma base temporal;
- CasparCG genere correctamente 25 frames/s;
- los producers hayan sido iniciados prácticamente al mismo tiempo.

**Esta hipótesis debe demostrarse o descartarse leyendo el código.**

No asumirla como cierta.

---

## 6. Pregunta esencial

Determinar si en el pipeline actual existe en algún lugar información suficiente para responder:

```text
¿qué PTS original representa este frame?
```

y si esa información llega desde:

```text
FFmpeg decoder
   ->
AVFrame
   ->
Caspar frame
   ->
frame_producer
   ->
layer/stage
   ->
mixer
```

o si se pierde, transforma, normaliza o deja de utilizarse durante el proceso.

---

# 7. Auditoría solicitada del código de CasparCG

Realizar una traza completa desde la recepción del stream hasta la composición.

No limitarse a buscar la palabra `pts`.

## 7.1 FFmpeg input / demuxer

Localizar:

- creación de `AVFormatContext`;
- apertura del input;
- selección de stream;
- lectura de paquetes;
- `AVPacket.pts`;
- `AVPacket.dts`;
- `AVStream.time_base`;
- `start_time`;
- PCR si existe tratamiento explícito;
- parámetros FFmpeg que puedan modificar timestamps.

Determinar qué timestamps recibe realmente el decoder.

## 7.2 Decoder

Analizar:

- `avcodec_send_packet`;
- `avcodec_receive_frame`;
- `AVFrame.pts`;
- `best_effort_timestamp`;
- tratamiento de B-frames;
- reordered PTS;
- time_base;
- conversiones realizadas por CasparCG.

Documentar cuál es el timestamp definitivo del frame decodificado.

## 7.3 Conversión AVFrame -> frame de CasparCG

Localizar dónde se crea el frame interno de CasparCG.

Determinar:

- qué metadata temporal se conserva;
- si el PTS se copia;
- si se convierte a duración/frame number;
- si se descarta;
- si se sustituye por una posición relativa al producer;
- si se normaliza respecto a `start_time`.

Esta parte es especialmente importante.

## 7.4 Buffer interno del producer

Analizar las colas/buffers existentes.

Determinar:

- qué contiene exactamente la cola;
- cómo se ordenan los frames;
- cuántos frames se almacenan;
- qué thread produce;
- qué thread consume;
- qué ocurre ante jitter;
- qué ocurre si el decoder va adelantado;
- qué ocurre si el decoder va retrasado;
- qué ocurre ante frame drops;
- qué ocurre ante discontinuidades de PTS.

## 7.5 `frame_producer`

Trazar el mecanismo mediante el cual CasparCG solicita el siguiente frame.

Responder con código concreto:

- ¿el consumer del producer pide simplemente `next_frame()`?
- ¿existe alguna noción de clock/PTS?
- ¿existe timestamp en la interfaz?
- ¿hay posibilidad de pedir un frame correspondiente a un tiempo concreto?
- ¿un producer sabe qué PTS está mostrando otro producer?

## 7.6 Layer / Stage

Analizar cómo se actualizan simultáneamente distintas layers.

Queremos saber qué significa realmente que dos layers sean procesadas en el mismo tick.

Determinar si:

```text
tick N
   layer 1 -> next frame
   layer 2 -> next frame
   layer 3 -> next frame
```

implica únicamente sincronía de llamada/render o si existe algún mecanismo de sincronización temporal entre contenidos.

## 7.7 Mixer

Analizar:

- cuándo recibe los frames;
- si recibe metadata temporal;
- si conoce PTS;
- si podría realizar selección temporal;
- si el mixer es un lugar apropiado o inapropiado para solucionar el problema.

No asumir que el mixer sea el punto correcto.

---

# 8. Diferenciar tres conceptos

El análisis debe separar explícitamente:

## A. Sincronización de ejecución

Dos producers son llamados en el mismo tick.

```text
Producer A -> next
Producer B -> next
```

Esto NO garantiza necesariamente que sus frames correspondan al mismo instante temporal.

## B. Sincronización por PTS

Ambos muestran frames cuyo presentation timestamp corresponde al mismo instante:

```text
A -> PTS X
B -> PTS X
```

o, si no existe coincidencia exacta, al frame más adecuado dentro de una tolerancia conocida.

## C. Sincronización de contenido

Dos rutas diferentes pueden tener:

- timestamps similares o iguales;
- diferentes procesos de encoding;
- frame drops;
- cambios de cadence;
- distinta selección de frames;
- delay añadido upstream.

Hay que determinar hasta qué punto:

```text
mismo PTS
```

garantiza realmente:

```text
misma imagen / mismo instante del contenido
```

en nuestro pipeline concreto.

No asumir ninguna de las dos posibilidades.

---

# 9. Hipótesis de `SyncGroup`

Evaluar una arquitectura en la que varios producers puedan pertenecer a un grupo de sincronización.

Ejemplo conceptual:

```text
SYNC GROUP 1
    layer 1
    layer 2
    layer 3
    layer 4
```

Cada producer conservaría frames indexados temporalmente:

```text
Producer A
  [PTS X-2]
  [PTS X-1]
  [PTS X]
  [PTS X+1]

Producer B
  [PTS X-2]
  [PTS X-1]
  [PTS X]
```

Un coordinador determinaría qué instante puede mostrarse con seguridad en todas las entradas.

Ejemplo:

```text
latest A = X+1
latest B = X
latest C = X+2
latest D = X

safe presentation point = X
```

y solicitaría:

```text
A -> frame X
B -> frame X
C -> frame X
D -> frame X
```

Analizar si este concepto encaja con la arquitectura de CasparCG o supondría luchar contra ella.

---

# 10. Posibles localizaciones del sincronizador

Comparar al menos estas arquitecturas.

## Opción A — dentro de cada `ffmpeg_producer`

Modificar producers para permitir selección/retención por PTS.

Problema potencial:

Cada producer sigue sin conocer directamente el estado de los demás.

## Opción B — `SyncManager` compartido

Crear una entidad común que agrupe producers.

```text
Producer A -> PTS queue --+
Producer B -> PTS queue --+
Producer C -> PTS queue --+--> SyncManager --> Stage/Mixer
Producer D -> PTS queue --+
```

Evaluar:

- ownership;
- threads;
- locking;
- performance;
- integración con lifecycle de producers/layers;
- cambio dinámico de streams;
- pérdida de una señal;
- reconexión.

## Opción C — nuevo producer multi-input

Crear un producer específico:

```text
sync_producer
    input A
    input B
    input C
    input D
```

que gestione múltiples inputs dentro de un único contexto de sincronización.

Evaluar si esto simplifica considerablemente el problema.

## Opción D — sincronización antes de CasparCG

FFmpeg/GStreamer externo:

```text
UDP A --+
UDP B --+
UDP C --+--> FrameSync/xstack --> mosaico --> CasparCG
UDP D --+
```

Ventaja:

- sincronización temporal fuera de Caspar.

Desventajas:

- Caspar pierde independencia de cada layer;
- menor flexibilidad para layouts;
- overlays/señales dejan de ser elementos independientes;
- puede reducir utilidad de Caspar como compositor.

Usar esta opción principalmente como referencia/POC salvo que resulte claramente superior.

## Opción E — mantener CasparCG sin modificar y añadir delays

Evaluarla, pero se considera inicialmente una solución inferior si el delay entre señales es variable.

---

# 11. Buffer de sincronización

Si se sincroniza por PTS, estudiar cuánto buffering sería necesario.

Conceptualmente:

```text
Input A -----> [buffer] ---+
Input B ---------> [buffer] ---+
Input C -------> [buffer] ---+--> synchronized output
Input D ------------> [buffer] ---+
```

La latencia de salida deberá ser suficiente para absorber:

- diferencia de llegada entre streams;
- jitter;
- reordenamiento;
- decoding;
- B-frames;
- variaciones de red;
- pausas momentáneas.

Analizar estrategias como:

```text
targetPTS = minimum(latestPTS de todos los miembros)
```

pero NO asumir que ésta sea la estrategia definitiva.

Compararla con:

- target clock;
- master producer;
- watermark temporal;
- ventana de tolerancia;
- nearest frame;
- previous frame;
- wait/hold;
- duplicate last frame;
- drop;
- discontinuity reset.

---

# 12. Qué hacer cuando falta un frame

Definir comportamiento para:

```text
A -> PTS X disponible
B -> PTS X disponible
C -> PTS X NO disponible
D -> PTS X disponible
```

Opciones a evaluar:

1. esperar;
2. repetir último frame de C;
3. usar el frame temporalmente más próximo;
4. saltar X;
5. declarar discontinuidad;
6. sacar señal de `SyncGroup`;
7. timeout configurable.

El diseño debe evitar que una señal muerta bloquee indefinidamente todo el mosaico.

---

# 13. Tolerancia temporal

No diseñar el sistema suponiendo obligatoriamente igualdad exacta de PTS.

Aunque la muestra actual MAIN/BACKUP coincide exactamente, el producto debería considerar:

```text
abs(PTS_A - PTS_B) <= tolerance
```

Evaluar tolerancias expresadas en:

- unidades del `time_base`;
- microsegundos;
- milisegundos;
- fracciones de frame.

Para 25 fps:

```text
1 frame = 40 ms
```

Determinar cuál sería la política adecuada.

---

# 14. Time bases

Validar cuidadosamente:

```text
AVStream.time_base
```

y cualquier conversión de PTS.

Dos streams podrían expresar el mismo instante utilizando distintos time bases.

Nunca comparar directamente dos enteros PTS sin normalizarlos antes a una escala común.

Evaluar uso de:

```cpp
av_rescale_q(...)
```

o mecanismo equivalente ya empleado por CasparCG.

---

# 15. Discontinuidades y rollover

Analizar comportamiento ante:

- discontinuity indicators;
- cambio brusco de PTS;
- reconexión UDP;
- reinicio de encoder;
- reinicio del Docker FFmpeg;
- cambio de fuente;
- PTS wrap;
- streams que empiezan en puntos temporales distintos.

El sincronizador debe disponer de un mecanismo para detectar que la timeline anterior ya no es válida y rearmarse.

---

# 16. Audio

Aunque inicialmente el objetivo visual es vídeo, revisar cómo afectaría la solución al audio.

Determinar:

- si cada layer conserva audio independiente;
- si el audio se sincroniza con el frame de vídeo;
- si la retención de vídeo exige retención equivalente de audio;
- si el mosaico utiliza realmente audio de todas las señales o solo VU meters.

No implementar una solución de vídeo que rompa A/V sync internamente.

---

# 17. Docker FFmpeg anterior a CasparCG

Existe actualmente un Docker FFmpeg por señal antes de CasparCG.

Hay que comprobar también ese pipeline.

Preguntas:

- ¿preserva PTS/DTS de la señal original?
- ¿usa `-copyts`?
- ¿usa `-start_at_zero`?
- ¿usa `-reset_timestamps`?
- ¿usa `setpts`?
- ¿regenera timestamps?
- ¿recodifica?
- ¿produce B-frames?
- ¿qué GOP utiliza?
- ¿hay `vsync` / `fps_mode` que pueda duplicar o descartar frames?
- ¿la salida MPEG-TS conserva una timeline común MAIN/BACKUP?
- ¿el muxer MPEG-TS modifica los timestamps?
- ¿se fuerza CBR?
- ¿se modifica PCR?

La evidencia CSV indica que, al menos en la muestra analizada, MAIN y BACKUP terminan con timestamps idénticos en los frames solapados.

Pero hay que verificar si eso se mantiene durante períodos largos.

---

# 18. GOP: hipótesis que hay que evaluar

Se había planteado aumentar el GOP del Docker FFmpeg como posible solución al desfase.

No asumir que ayude.

Analizar técnicamente sus consecuencias.

Un GOP mayor puede afectar:

- frecuencia de I-frames;
- dependencia entre frames;
- latencia del encoder;
- recuperación tras packet loss;
- tiempo de adquisición después de conectar;
- seek/resync;
- cantidad y estructura de B-frames.

Pero, en principio, **no debería solucionar el problema fundamental de dos CasparCG producers independientes que muestran PTS diferentes en el mismo tick**.

Incluso podría empeorar recuperación y latencia.

Validar esta afirmación según la configuración real de FFmpeg y CasparCG.

También evaluar si para un multiviewer de baja latencia tendría más sentido:

- GOP más corto;
- limitar B-frames;
- `bf=0`;
- all-intra en algún escenario;
- mantener GOP actual.

No recomendar cambios hasta medir su efecto.

---

# 19. Medición adicional necesaria: arrival time

Los CSV actuales demuestran coincidencia de timestamps, pero NO indican cuándo llegó físicamente cada frame al servidor.

Necesitamos poder medir:

```text
PTS          arrival MAIN       arrival BACKUP     delta
X            T1                 T2                 T2-T1
X+1          T3                 T4                 T4-T3
...
```

Proponer la mejor manera de instrumentarlo.

Puede ser:

- en CasparCG;
- FFmpeg;
- ffprobe;
- libavformat/libavcodec;
- captura de paquetes;
- timestamp monotónico en el momento de decode;
- otro método.

Queremos obtener:

- delay medio MAIN/BACKUP;
- mínimo;
- máximo;
- p95;
- p99;
- jitter;
- estabilidad a lo largo de minutos/horas.

Esta métrica permitirá determinar el tamaño real del buffer de sincronización.

---

# 20. Latencia vs sincronización

Documentar explícitamente el trade-off:

```text
más buffer
    ->
más capacidad para sincronizar
    ->
más latencia
```

El sistema es un multiviewer/monitor, no necesariamente una salida de emisión final.

Por tanto cierta latencia añadida puede ser aceptable si garantiza estabilidad visual.

Proponer valores razonables solo después de disponer de mediciones.

---

# 21. Rendimiento

El sistema puede contener muchas señales.

Analizar impacto de:

- guardar múltiples frames decoded en RAM;
- frames HD/UHD;
- GPU frames si existen;
- copies;
- locks;
- allocations;
- producers simultáneos;
- 25/50 fps.

Evitar una arquitectura que requiera copias masivas innecesarias.

Investigar si CasparCG ya usa:

- reference-counted frames;
- shared pointers;
- immutable frames;
- pools;
- GPU textures;
- mecanismos reutilizables.

---

# 22. Threading

Esta parte debe analizarse con especial cuidado.

Crear un diagrama de threads mostrando:

```text
network/demux thread
decoder
producer worker
stage
mixer
consumer/output
```

y localizar:

- mutex existentes;
- atomics;
- executors;
- strands;
- queues;
- futures;
- bloqueos potenciales.

El sincronizador no puede bloquear el hilo de render esperando indefinidamente a una entrada lenta.

---

# 23. Configuración / API

Si se recomienda introducir `SyncGroup`, proponer cómo podría configurarse sin romper compatibilidad.

Por ejemplo, solo como idea conceptual:

```text
SYNC 1-1 GROUP 1
SYNC 1-2 GROUP 1
```

o configuración XML/JSON equivalente.

NO diseñar primero la sintaxis.

Diseñar primero la arquitectura interna y después proponer la API mínima.

La funcionalidad actual de CasparCG debe permanecer intacta para layers que no pertenezcan a grupos sincronizados.

---

# 24. Logging y observabilidad

Una implementación futura debería permitir mostrar algo parecido a:

```text
SYNC GROUP 1
A pts=74487.7848 arrival=...
B pts=74487.7848 arrival=...
delta=...
buffer=...
drops=...
holds=...
state=SYNCED
```

Diseñar métricas que permitan diagnosticar el sistema en producción.

Proponer al menos:

- current PTS;
- current normalized PTS;
- latest decoded PTS;
- presented PTS;
- buffer depth;
- inter-input delta;
- frame drops;
- duplicated/held frames;
- discontinuities;
- resync count;
- sync state.

---

# 25. Experimentos antes de modificar arquitectura

Diseñar pruebas pequeñas que permitan validar hipótesis.

## Experimento 1 — Instrumentar PTS dentro de CasparCG

Registrar por cada producer:

```text
decode timestamp
original PTS
normalized PTS
frame delivered timestamp
Caspar tick/frame number
layer
```

Objetivo:

demostrar qué PTS está mostrando realmente cada layer en cada tick.

## Experimento 2 — MAIN/BACKUP

Para dos señales conocidas:

```text
tick Caspar | MAIN PTS | BACKUP PTS | delta
```

Si aparece algo como:

```text
1001 | X+5 | X | 200 ms
1002 | X+6 | X+1 | 200 ms
```

habremos demostrado el origen del desfase.

## Experimento 3 — arrival delta

Medir cuándo llega el mismo PTS a cada producer.

## Experimento 4 — FFmpeg FrameSync/xstack externo

Como referencia, probar ambas señales en un único filter graph de FFmpeg y observar si quedan visualmente sincronizadas.

No se plantea necesariamente como arquitectura final.

## Experimento 5 — GOP/B-frames

Probar de forma controlada:

- configuración actual;
- GOP menor;
- GOP mayor;
- B-frames reducidos;
- B-frames deshabilitados.

Medir:

- latencia;
- arrival delta;
- estabilidad;
- recuperación;
- PTS/DTS;
- CPU/GPU.

No basarse en percepción visual únicamente.

---

# 26. Qué NO hacer todavía

No:

- modificar código;
- crear `SyncManager`;
- cambiar interfaces públicas;
- cambiar GOP;
- eliminar B-frames;
- cambiar timestamps de FFmpeg;
- introducir delays arbitrarios;
- reescribir producers.

Primero demostrar dónde está el problema.

---

# 27. Entregables esperados de esta investigación

Crear un informe técnico en Markdown con esta estructura.

## A. Pipeline real de CasparCG

Diagrama real basado en código:

```text
UDP
 -> ...
 -> ...
 -> AVPacket
 -> AVFrame
 -> ...
 -> frame_producer
 -> layer
 -> stage
 -> mixer
 -> consumer
```

Para cada transición indicar:

- clase;
- fichero;
- función;
- thread;
- metadata temporal disponible.

## B. Tratamiento exacto de PTS/DTS

Tabla:

| Etapa | PTS disponible | DTS disponible | Time base | Se modifica | Se conserva hasta siguiente etapa |
|---|---|---|---|---|---|

## C. Causa probable del desfase

Distinguir:

- hechos demostrados;
- hipótesis;
- cuestiones pendientes.

## D. Evidencia de código

Toda conclusión importante debe incluir:

```text
ruta/fichero.cpp
función
líneas aproximadas
explicación
```

No responder únicamente con conocimiento general de FFmpeg/CasparCG.

## E. Opciones de solución

Comparar al menos:

1. instrumentación sin cambio funcional;
2. delay por layer;
3. PTS-aware producer;
4. `SyncManager` / `SyncGroup`;
5. producer multi-input;
6. sincronización externa con FFmpeg/GStreamer;
7. cualquier alternativa mejor descubierta en el código.

Para cada opción:

| Criterio | Resultado |
|---|---|
| complejidad | |
| riesgo | |
| latencia añadida | |
| cambio API | |
| rendimiento | |
| compatibilidad | |
| capacidad real de sync por PTS | |
| tolerancia a pérdida de señal | |
| mantenibilidad | |

## F. Recomendación

Elegir:

- opción recomendada;
- segunda opción;
- opciones descartadas;
- razones.

## G. Plan incremental

Proponer fases pequeñas.

Ejemplo:

```text
FASE 0 - instrumentación
FASE 1 - prueba con 2 producers
FASE 2 - buffer PTS experimental
FASE 3 - SyncGroup mínimo
FASE 4 - gestión de errores
FASE 5 - N inputs
FASE 6 - optimización
```

Cada fase debe ser:

- comprobable;
- reversible;
- pequeña;
- con criterios de aceptación.

---

# 28. Preguntas que el informe debe responder obligatoriamente

1. ¿Dónde entra el PTS original en CasparCG?
2. ¿Qué timestamp usa finalmente el frame decodificado?
3. ¿Se conserva el PTS original dentro del frame de CasparCG?
4. ¿En qué punto se pierde?
5. ¿Cada `ffmpeg_producer` corre temporalmente de forma independiente?
6. ¿Qué hace exactamente `next_frame()`?
7. ¿Dos layers procesadas en el mismo tick pueden mostrar PTS diferentes?
8. ¿Puede demostrarse con instrumentación mínima?
9. ¿Cuál es el lugar arquitectónicamente correcto para coordinar producers?
10. ¿Existe ya alguna abstracción de CasparCG reutilizable?
11. ¿Puede realizarse sin modificar el mixer?
12. ¿Puede realizarse sin romper producers no sincronizados?
13. ¿Qué buffer mínimo sería necesario?
14. ¿Cómo evitar que una señal caída bloquee las demás?
15. ¿Qué hacer ante discontinuidades?
16. ¿Cómo afecta al audio?
17. ¿Cómo afecta a GPU/RAM/CPU?
18. ¿Aumentar el GOP ayudaría realmente?
19. ¿Los B-frames son parte del problema o solo explican el orden PTS/DTS?
20. ¿Cómo medir el arrival time del mismo PTS?
21. ¿Cómo demostrar visual y numéricamente que la solución funciona?
22. ¿Mismo PTS garantiza mismo instante de contenido en nuestro pipeline real?
23. Si no lo garantiza siempre, ¿qué mecanismo adicional sería necesario?
24. ¿Es CasparCG el lugar correcto para solucionar esto o debería resolverse upstream?

---

# 29. Principio de diseño

La solución ideal debería permitir conceptualmente:

```text
NORMAL LAYERS
layer 1 -> comportamiento CasparCG tradicional
layer 2 -> comportamiento CasparCG tradicional

SYNC GROUP A
layer 10 -> signal MAIN
layer 11 -> signal BACKUP
layer 12 -> signal C
layer 13 -> signal D
```

sin alterar el comportamiento de las demás layers.

El sistema debería añadir sincronización solo cuando se solicita explícitamente.

---

# 30. Resultado final deseado

Queremos pasar de un modelo conceptual:

```text
tick
  producer A -> siguiente frame disponible
  producer B -> siguiente frame disponible
  producer C -> siguiente frame disponible
```

a uno que, para grupos sincronizados, sea equivalente a:

```text
target presentation instant = T

producer A -> frame correspondiente a T
producer B -> frame correspondiente a T
producer C -> frame correspondiente a T
```

con reglas explícitas para:

- tolerancia;
- ausencia de frame;
- jitter;
- buffer;
- discontinuidades;
- caída de señal;
- resync.

La implementación concreta debe surgir del análisis del código real de CasparCG, no de esta descripción conceptual.

---

# 31. Regla de trabajo para esta tarea

**No escribas código todavía.**

Primero:

1. recorre el repositorio;
2. localiza las clases y funciones reales;
3. traza el flujo completo;
4. valida/refuta las hipótesis;
5. diseña experimentos;
6. entrega el informe técnico;
7. espera una decisión antes de implementar.

Si encuentras que alguna premisa de este documento es incorrecta, indícalo claramente y demuestra por qué con referencias al código.

La prioridad es entender correctamente CasparCG antes de modificarlo.
