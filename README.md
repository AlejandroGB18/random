# Investigación: benchmark comparativo de trackers MOT para video de dron con cámara en movimiento

> **Fecha:** 2026-11-17 · **Alcance:** cómo medir, con ground truth, los 4 trackers de `VD - Trackers To Real Implementation/` (propio, UCMC, ByteTrack, BoT-SORT) y si BoxMOT sirve como infraestructura.
>
> **Versiones verificadas en código fuente** (clonadas en modo solo lectura, sin ejecutar):
>
> | Repo | Commit / versión | Fecha del commit |
> |---|---|---|
> | BoxMOT `master` | [`fab3fe0`][bm] (pyproject `version = "26.0.0"`) | 2026-11-12 |
> | BoxMOT último release | `v26.0.0` → [`75b91fb`][bm-v26] | 2026-10-27 |
> | TrackEval (original) | [`12c8791`][te] | 2022-11-29 |
> | Roboflow `trackers` | [`84569dc`][rf] (v2.7.0) | 2026-11-16 |
> | py-motmetrics | [`e5ae766`][pmm] (v1.4.0 en PyPI) | 2026-07-17 |
> | VisDrone2018-MOT-toolkit | [`2dc45b4`][vdt] | 2019-07-01 |
> | UCMCTrack | [`7851a49`][ucmc-repo] | — |
> | BoT-SORT (oficial) | [`2519854`][botsort-repo] | 2022-10-30 |
> | Ultralytics | 8.4.192 (paquete instalado en esta máquina) | — |
>
> Convención: **[V]** = verificado leyendo la fuente primaria citada; **NO VERIFICADO** = no se pudo comprobar; **[Propuesta]** = recomendación propia, no viene de la literatura.

---

## Índice

0. [Resumen ejecutivo](#0-resumen-ejecutivo)
1. [¿Existe realmente "ground truth"?](#1-existe-realmente-ground-truth)
2. [Nuestro caso: ego-movimiento del dron vs. lo que se mueve en la escena](#2-nuestro-caso-ego-movimiento-del-dron-vs-lo-que-se-mueve-en-la-escena)
3. [Las métricas en detalle](#3-las-métricas-en-detalle)
4. [El flujo completo del benchmark](#4-el-flujo-completo-del-benchmark)
5. [Integrar nuestros 4 trackers en BoxMOT](#5-integrar-nuestros-4-trackers-en-boxmot)
6. [¿Es BoxMOT la mejor opción?](#6-es-boxmot-la-mejor-opción)
7. [Decisiones abiertas](#7-decisiones-abiertas)
8. [Fuentes](#8-fuentes)

---

## 0. Resumen ejecutivo

| # | Pregunta | Respuesta corta |
|---|---|---|
| 1 | ¿Existe GT? | **Sí en los benchmarks públicos, no en VIGIA.** El GT es una anotación humana por frame: caja + ID persistente + clase + banderas (ignorar, visibilidad, oclusión). VisDrone y UAVDT la hicieron anotadores expertos; UAVDT con >10 expertos durante dos meses usando VATIC ([UAVDT, arXiv 1804.00518][uavdt]). El evaluador empareja predicción↔GT por IoU (0.5 en CLEAR e IDF1; barrido α = 0.05…0.95 en HOTA). Para VIGIA hay que **anotar un subconjunto pequeño** (3–5 clips de 20–60 s, en CVAT con formato MOT, muestreado a ~10 fps). Las métricas actuales (IDs, vida media, tracks/objeto) son *proxies* que no distinguen fragmentar de fusionar identidades. |
| 2 | ¿Qué medir para separar ego-movimiento de movimiento real? | HOTA/DetA/AssA/IDF1/IDSW/Frag **estratificados por intensidad de ego-movimiento** (la magnitud de H por frame, que ya calculamos como `mov_camara_px`). Además: **evaluación separada de móviles y estáticos**, con una etiqueta GT `moving` derivada de la trayectoria estabilizada y revisada a mano. Para el decisor `en_movimiento` (que ningún toolkit evalúa): precisión/recall/F1 por frame y por track, **latencia de encendido y apagado**, falsas alarmas sobre objetos estáticos por bin de ego-movimiento, y un **HOTA "de móviles"** de punta a punta. |
| 3 | Métricas | HOTA = √(DetA·AssA), promediado en 19 umbrales α. MOTA está dominado por la detección; IDF1 mide la identidad con un emparejamiento global; HOTA equilibra ambos y añade la localización ([Luiten et al., IJCV][hota]). Se leen juntas: **DetA para el detector, AssA/IDF1/IDSW para el tracker y LocA para el ajuste de las cajas**. |
| 4 | Flujo | Datos → GT (+ `moving` + ignorados) → detector **fijo y cacheado** con umbral bajo → H cacheada → trackers (mismas entradas) → MOT `.txt` + archivo lateral de `en_movimiento` → **un solo evaluador** (TrackEval) → análisis estratificado → bootstrap pareado. Se ajusta en clips y datasets distintos de los de test. |
| 5 | BoxMOT | En `master`/v26 no hay registro externo: los trackers están en diccionarios fijos (`_TRACKER_MANIFEST` y `_TRACKER_CAPABILITIES`), así que integrar los nuestros obliga a **hacer fork**. Los 4 necesitan el frame (para su H), devuelven filas de 8 columnas con índice de detección y no tienen canal para `en_movimiento`. **BoxMOT no trae UCMCTrack.** Su CMC precalculada (`PreparedCMC`) solo acepta afines 2×3, no nuestra homografía 3×3. Su evaluador interno trata las regiones ignoradas de VisDrone distinto del toolkit oficial. Licencia AGPL-3.0, igual que Ultralytics, que ya usamos. |
| 6 | ¿Es BoxMOT lo mejor? | **No como juez ni como contenedor de nuestros trackers.** Recomendación: un **harness propio delgado + TrackEval** (el fork mantenido `trackeval==1.3.0`) como evaluador único, validado una vez contra py-motmetrics y el toolkit de VisDrone. BoxMOT (o Roboflow `trackers`, Apache-2.0) sirve como **biblioteca de baselines extra** (OC-SORT, BoostTrack, etc.) llamados desde ese harness. |

### Hallazgos que contradicen o corrigen la conversación previa y el doc de benchmark

1. **La evaluación de BoxMOT trata las regiones ignoradas de VisDrone distinto del toolkit oficial.** BoxMOT elimina una predicción solo si queda emparejada por Hungarian con IoU ≥ 0.5 a una caja de clase 0/11 ([motmetrics.py L488-506][bm-mm-aabb]). El toolkit oficial elimina toda predicción con **≥ 50 % de su área** dentro de una región ignorada ([dropObjects.m L31][vdt-drop]). Las regiones ignoradas son zonas grandes y un auto pequeño dentro de una casi nunca llega a IoU 0.5 con ella, así que con BoxMOT esos autos cuentan como FP. Confirma lo que proponía la conversación (replicar la regla oficial), pero **no es correcto asumir que `boxmot eval` ya lo hace**.
2. **BoxMOT ya no usa TrackEval para evaluar** ("TrackEval is not required at runtime", [evaluation.md L128][bm-doc-eval]); trae un evaluador propio ([motmetrics.py][bm-mm]). El doc lo decía bien. La propuesta original suponía otra cosa.
3. **El TrackEval original no recibe commits desde 2022-11-29** y usa `np.float`/`np.int` en 81 lugares, alias que NumPy eliminó en la 1.24. En una instalación moderna falla sin un parche. El paquete PyPI `trackeval` (1.3.0) se publica desde el fork [kovalp/TrackEval][te-fork], que corrigió esos alias. Es el que fijan BoxMOT ([trackeval_reference.py L24-38][bm-te-ref]) y py-motmetrics para sus pruebas de paridad ([Readme][pmm-readme]).
4. **"Inyectar el GMC precalculado" ya existe en BoxMOT, pero solo como afín 2×3** ([prepared.py L25][bm-prepared]), y lo controla la política CMC de `eval/tune` ([materialize.md L365-382][bm-doc-mat]). Nuestra H es 3×3 (homografía). Roboflow `trackers` también acepta solo una afín 2×3 (`cmc_transform`, [tracker.py L232-260][rf-botsort]).
5. **BoxMOT no acepta "detecciones cacheadas propias" en el layout MOT.** Las detecciones salen de *sus* backends de detector en la etapa `materialize` (Ultralytics, YOLOX, RT-DETR) ([detectors.md][bm-doc-det]). Las "saved 2D detections" solo existen en una codificación concreta (`trackrcnn`, 138 campos), para el flujo KITTI y sin tune ([experiments.md][bm-doc-exp], [datasets.md][bm-doc-ds]). El doc afirmaba que BoxMOT "reutiliza detecciones guardadas": es cierto para *sus* builds, no para las nuestras.
6. **La afirmación de la conversación de que `update()` siempre exige imagen** quedó desactualizada. Hoy `frame` es opcional (`frame: Frame | np.ndarray | None = None`, [base.py L351-358][bm-base-update]). Sin embargo, **nuestros 4 trackers sí necesitan el frame**, porque todos calculan H (también "UCMC", a diferencia del UCMCTrack original).
7. **El ByteTrack y el BoT-SORT de BoxMOT no son los de Ultralytics**: cambian los valores por defecto, la gestión de pérdida y la CMC (ver §5.9). Además, el ByteTrack de BoxMOT **puede activar CMC** (`use_cmc`, por defecto `False`, [config.py L52][bm-bt-cfg]).
8. **El doc cita a SanMiguel et al.** ("en 32 combinaciones, HOTA correlaciona con el mAP del detector y MOTA con su precisión"). El abstract ([arXiv 2212.08536][tem]) confirma 4 trackers × 8 conjuntos de detecciones y que su TEM "reduce la correlación" con las detecciones, pero **no da esas correlaciones concretas → NO VERIFICADO**.
9. **El doc decidía "los vehículos estacionados cuentan; el análisis solo de móviles queda para después".** Para nuestro objetivo (detectar *qué se mueve de verdad*) ese análisis es **central, no secundario**. Hay precedentes para tratarlo así: MOT16 tiene una clase `static person` tratada como distractora ([MOT16, arXiv 1603.00831][mot16]), y KITTI-MOD genera etiquetas estático/dinámico a partir de GT + odometría ([MODNet, arXiv 1709.04821][kittimod]).
10. **BoxMOT cambia mucho de versión**: v21 (2026-06-03), v22 (07-10), v23 (08-26), v25 (09-09) y v26 (10-27), cinco *majors* en ~5 meses (API de releases de GitHub). La v26.0.0 trae 11 trackers; `master` añade `edgetam`.
11. **Lo que sí quedó verificado de la conversación y del doc:** VisDrone-MOT tiene 56/7/17/16 secuencias (train/val/test-dev/test-challenge) ([arXiv 2001.06303][visdrone-paper]). El perfil `visdrone` de BoxMOT llama `val` a test-dev ([visdrone.yaml][bm-visdrone]). El loader MOTChallenge de TrackEval solo admite `pedestrian` ([L71][te-mot-71]). UAVDT tiene 1080×540 a 30 fps, ~80 k frames y 0.84 M cajas. En DynUAV, la CMC lleva AdapTrack de IDSW 1123→583 y AssA 58.97→66.71 ([arXiv 2603.05970][dynuav]). MMOT tiene un desplazamiento medio de 14.1 px atribuible al dron ([arXiv 2510.12565][mmot]).

---

## 1. ¿Existe realmente "ground truth"?

### 1.1 Qué es

En MOT, el "ground truth" es una **anotación humana**, no una medida física. Para cada frame lista las cajas de los objetos de interés, con un **ID que se mantiene mientras el objeto existe**, su clase y banderas que le dicen al evaluador qué ignorar. No es verdad absoluta: tiene ruido de anotación (UAVDT reconoce que se revisó "with several rounds of double-check", sin poder eliminar todos los errores, [arXiv 1804.00518 §2.1][uavdt]). También tiene convenciones: qué cuenta como objetivo, qué se ignora y cómo se trata la oclusión.

### 1.2 Formatos de GT relevantes

**MOTChallenge (MOT16/17), `gt/gt.txt`** [V] ([MOT16, Tabla 5][mot16]):

| Col | Campo | En GT |
|---|---|---|
| 1 | frame | índice de frame (empieza en 1) |
| 2 | id | ID de la trayectoria (−1 en archivos de detección) |
| 3–6 | left, top, width, height | caja en píxeles (esquina superior izquierda + tamaño) |
| 7 | conf | **bandera "considerar"**: 1 = se evalúa, 0 = se ignora |
| 8 | class | 1 pedestrian, 2 person on vehicle, 3 car, 4 bicycle, 5 motorbike, 6 non-motorized vehicle, **7 static person**, 8 distractor, 9–11 occluders, 12 reflection |
| 9 | visibility | fracción visible en [0, 1] (por oclusión o recorte en el borde) |

MOT16 define clases "parecidas al objetivo" (distractor, reflection, **static person**, person on vehicle): el tracker "neither penalized nor rewarded" por seguirlas. Las predicciones que se superponen más del 50 % con ellas se eliminan antes de puntuar ([MOT16 §4.1.3][mot16]).

**VisDrone-MOT** [V] ([README del toolkit][vdt]): `frame, target_id, bbox_left, bbox_top, bbox_width, bbox_height, score, object_category, truncation, occlusion`.
- `score` en GT: 1 = se evalúa, 0 = se ignora.
- `object_category`: **0 = ignored regions**, 1 pedestrian, 2 people, 3 bicycle, **4 car, 5 van, 6 truck**, 7 tricycle, 8 awning-tricycle, **9 bus**, 10 motor, **11 others**. El challenge solo evalúa car, bus, truck, pedestrian y van.
- `truncation`: 0 = no, 1 = parcial (1–50 % fuera del frame).
- `occlusion`: 0 = no, 1 = parcial (1–50 %), 2 = fuerte (50–100 %).
- Splits: 56 train (24 198 frames), 7 val (2 846), 17 test-dev (6 635), 16 test-challenge (6 322) ([arXiv 2001.06303 §5.1][visdrone-paper]). El GT de test-challenge no es público ([VisDrone-Dataset][visdrone-ds]).
- Protocolo oficial: con detecciones propias, **mAP sobre trayectorias** a IoU 0.25/0.5/0.75. Con las detecciones provistas, **CLEAR-MOT + IDF1** ([arXiv 2001.06303 §7.1][visdrone-paper]; umbrales en [evaluateTrackA.m L5-7][vdt-evalA]).
- Regla oficial de ignorados: se descarta toda predicción con **≥ 50 % de su área** dentro de regiones de clase 0 u 11 ([dropObjects.m][vdt-drop]).

**UAVDT** [V] ([arXiv 1804.00518][uavdt]):
- 100 secuencias, 30 fps, 1080×540. ~80 000 frames anotados, >2 700 vehículos, 0.84 M cajas. Clases car/truck/bus.
- **Regiones ignoradas** donde los vehículos son demasiado pequeños (rosa en la Fig. 1).
- Atributos por secuencia para MOT: clima (día/noche/niebla), **altitud** (baja 10–30 m, media 30–70 m, alta > 70 m) y **vista** (frontal/lateral/cenital). Por objeto: oclusión (0 / 1–30 / 30–70 / 70–100 %) y out-of-view (los objetos con más del 50 % fuera se descartan).
- "Camera Motion" figura como desafío central, pero **no** como atributo anotado por secuencia en MOT.
- Split: 30 secuencias de entrenamiento y 70 de test, de las cuales 20 son las de test de DET/MOT.
- Métricas que reporta: IDF1/IDP/IDR, MOTA, MOTP, MT, ML, FP, FN, IDS y FM.
- La licencia del dataset es NO VERIFICADO (el doc dice "non-commercial"; no lo comprobé en la fuente).

### 1.3 Quién lo produce

- **VisDrone y UAVDT:** equipos académicos con anotadores expertos. UAVDT: ">10 domain experts", herramienta VATIC, dos meses ([§2.1][uavdt]). VisDrone: anotación manual descrita en [arXiv 2001.06303 §5.1][visdrone-paper].
- **MOTChallenge:** el consorcio del benchmark, con un protocolo de anotación publicado ([MOT16][mot16]).
- **VIGIA:** nadie. **No hay GT** (README del repo y "Límite" del paso C del notebook v13).

### 1.4 Cómo se usa el GT en el emparejamiento

| Familia | Unidad de emparejamiento | Umbral | Algoritmo | Fuente |
|---|---|---|---|---|
| **CLEAR (MOTA/MOTP)** | detección por frame | IoU ≥ 0.5 (`THRESHOLD: 0.5`) | Hungarian por frame; primero se mantienen los pares del frame anterior que siguen sobre el umbral (para no crear IDSW) y luego se maximiza la similitud | [clear.py L15, L62-97][te-clear]; [HOTA §4.1][hota] |
| **Identity (IDF1)** | trayectoria completa | IoU ≥ 0.5 por frame | Hungarian **global** sobre trayectorias que minimiza IDFP + IDFN | [identity.py L15, L80][te-id]; [Ristani §3.3][ristani] |
| **HOTA** | detección por frame, ponderada por la alineación global | **α ∈ {0.05, 0.10, …, 0.95}** (19 valores) | Hungarian por frame con `score = alineación_global(gtID, prID) × IoU`; un par es TP en α si IoU ≥ α | [hota.py L17, L85-100][te-hota]; [HOTA Ec. 15-16][hota] |

**Preprocesado antes de emparejar** (TrackEval, MOT17) [V] ([mot_challenge_2d_box.py L320-400][te-mot-pre]):
1. Se emparejan *todas* las cajas GT, incluidas las distractoras, con IoU ≥ 0.5.
2. Se eliminan las predicciones emparejadas con una clase distractora (`person_on_vehicle`, `static_person`, `distractor`, `reflection`).
3. Se eliminan del GT las filas con `zero_marked` (col. 7 = 0) y las de clases no evaluadas.

Esta lógica es la que replican BoxMOT ([motmetrics.py L488-506][bm-mm-aabb]) y Roboflow ([MOTClassConfig][rf-classes]). **No equivale a la regla de área de VisDrone.** Para VisDrone/UAVDT, lo correcto es reproducir la regla oficial: eliminar las predicciones (y las cajas GT) con ≥ 50 % de área dentro de regiones ignoradas.

### 1.5 Opciones cuando no hay GT

| Opción | Qué da | Límites |
|---|---|---|
| **A. Anotar un subconjunto propio** (CVAT o Label Studio) | Métricas reales (HOTA/IDF1…) en *nuestro* dominio: 4K, cenital/oblicuo, altitudes VIGIA | Costo humano. Pocos clips dan intervalos de confianza anchos. Riesgo de sesgo si la pre-anotación sale de uno de los trackers comparados |
| **B. Pseudo-GT** (un tracker/segmentador más fuerte y offline, o propagación con H, corregido a mano) | Abarata la opción A | Si se usa **sin revisión**, mide parecido al generador, no calidad. Si sale de nuestra H o de uno de los 4 trackers, **favorece a ese método** (leakage). Solo vale como pre-anotación con revisión humana completa |
| **C. Métricas sin GT** (las actuales: IDs, tracks, vida media, tracks/objeto, IDs/min, parpadeo) | Baratas y útiles para detectar regresiones entre versiones del mismo tracker | No distinguen un ID nuevo correcto de una fragmentación, **premian fusionar identidades** (menos tracks y vidas más largas aunque haya ID transfer) y no ven FP/FN. Hay que **validarlas contra GT** (§2.6) antes de usarlas para decidir |
| **D. Datasets públicos** (VisDrone-MOT, UAVDT; para ego-movimiento fuerte, MMOT o DynUAV) | GT de calidad y comparabilidad con la literatura | Brecha de dominio (otras ciudades, alturas, fps y resoluciones). No tienen etiqueta `moving`. VisDrone/UAVDT son "high-altitude hovering or slow cruising" según los autores de DynUAV ([arXiv 2603.05970][dynuav]) |

**Formatos de anotación** [V]:
- **CVAT** exporta e importa "MOT 1.1": `img1/` + `gt/gt.txt` + `gt/labels.txt`, con columnas `frame_id, track_id, x, y, w, h, "not ignored", class_id, visibility`. Los atributos `visibility` (número) e `ignored` (casilla) se mapean a esas columnas ([CVAT docs][cvat-mot]). Atributos extra, como `moving`, **no** salen en MOT 1.1: hay que exportarlos en "CVAT for video 1.1" (XML) y convertirlos con un script (NO VERIFICADO el detalle del XML; es el formato nativo de CVAT).
- **Label Studio** exporta tracking de video en JSON con *keyframes* e interpolación **lineal** opcional al exportar (`--interpolate-key-frames`). **No** encontré export MOT nativo, así que habría que convertirlo ([docs de Label Studio][ls-video]; el detalle de la conversión a MOT es NO VERIFICADO).
- **Recomendación [Propuesta]:** CVAT (interpolación entre keyframes en pista, export MOT directo) más un atributo por frame `moving ∈ {sí, no, dudoso}` exportado en el XML nativo.

### 1.6 Estrategia y costo realista de anotar VIGIA

**Lo que sabemos de los clips** (derivado de `comparativa_v13/resumen_metricas.csv`: duración ≈ `ids_emitidos / ids_por_minuto`; densidad = `objetos_simultaneos_prom` de `propio_kf`):

| Video | Toma | Duración aprox. | Objetos simultáneos (prom.) | Mov. cámara (`tipo_toma`) |
|---|---|---|---|---|
| vigia_4 | cenital (OBB) | ~43 s | ~1 | **rápido** (25 % de frames > 3 px) |
| vigia_2 | cenital | ~20 s | ~78 | suave |
| vigia_1 | cenital | ~53 s | ~29 | suave |
| vigia_3 / 5 / 6 | cenital | 39 / 42 / 23 s | 48 / 5 / 5 | estacionario |
| vigia_8 / 9 / 10 / 11 | oblicuo (COCO) | 17 / 26 / 32 / 57 s | 12 / 26 / 13 / 13 | suave (11: 7.5 % rápido) |

**Plan [Propuesta]:**
1. **Elegir 4–5 clips (o tramos de 20–30 s)** que cubran estacionario, suave y rápido, y cenital y oblicuo. Ejemplo: vigia_4 (rápido, pocos objetos), un tramo de vigia_1 (suave, densidad media), un tramo de vigia_9 o vigia_10 (oblicuo) y un tramo de vigia_5 (estacionario, control).
2. **Anotar a ~10 fps** (1 de cada 3 frames si el video es de 30 fps; el fps real de VIGIA es NO VERIFICADO aquí). Los trackers corren a fps completo y **la evaluación se hace solo en los frames anotados**. Es válido porque CLEAR, Identity y HOTA emparejan por frame y TrackEval solo exige que GT y predicciones compartan índices de frame. Lo que se pierde son los cambios de ID de ida y vuelta que ocurren entre frames anotados.
3. **Clases:** solo vehículos (car/truck/bus/van → superclase `vehicle` para la evaluación principal). Lo que no se pueda separar (estacionamientos densos, vehículos de pocos píxeles) se marca como **región ignorada** (en CVAT, caja con `ignored=true`).
4. **Atributos por frame:** `visibility` (0–1), `moving` (sí/no/dudoso) y, opcionalmente, `elevated` (para casos de paralaje).
5. **Pre-anotación:** propagación con interpolación de CVAT entre keyframes. **No** pre-anotar con ninguno de los 4 trackers. Si se usa un propagador automático (p. ej. SAM2 en CVAT), debe ser ajeno a los métodos comparados, y la revisión humana debe ser completa.
6. **QA:** segunda pasada por otra persona en ~20 % de los frames; medir el acuerdo entre anotadores (IoU medio y consistencia de IDs) para conocer el ruido del GT.

**Costo estimado [Propuesta; supuestos explícitos, no medidos]:** la cámara se mueve, así que *los objetos estáticos también se desplazan en la imagen* y la interpolación lineal necesita keyframes frecuentes.

Supuestos: un keyframe cada ~5 frames anotados (≈ 0.5 s) y ~4 s por ajuste de caja. Con 15 objetos simultáneos en 25 s anotados a 10 fps: 15 × 50 keyframes × 4 s ≈ 50 min, más la asignación de IDs y atributos (~30 min) y el QA (~30 min). Unas **2 h por clip de densidad media**. Un clip denso como vigia_2 (~78 objetos) cuesta **8–10 h** o más, así que conviene recortarlo a una región o reemplazarlo. **Total realista para 4–5 clips: 10–20 horas-persona.** Hay que validar estas cifras cronometrando el primer clip.

---

## 2. Nuestro caso: ego-movimiento del dron vs. lo que se mueve en la escena

### 2.1 Qué dice la literatura primaria

| Fuente | Qué aporta a nuestro problema |
|---|---|
| **BoT-SORT** ([arXiv 2206.14651 §3.2][botsort-paper]) | GMC estilo OpenCV videostab: keypoints, flujo óptico disperso y una **afín 2×3** con RANSAC. Con ella transforma la media y la covarianza del Kalman (Ec. 5-8). En la ablación de MOT17-val, CMC lleva HOTA de 68.12 a 69.06 e IDF1 de 79.89 a 81.51 (Tabla 1). Limitación declarada: con muchos objetos dinámicos, la estimación de la cámara "may fail due to lack of background keypoints" (§4.4) |
| **UCMCTrack** ([arXiv 2312.08952][ucmc-paper], AAAI 2024) | Asocia en el **plano del suelo** con Mapped Mahalanobis Distance y aplica "the same compensation parameters across the entire sequence". Los parámetros de cámara de MOT17/MOT20/DanceTrack se **estimaron a mano** (§Experimentos; archivos en [`cam_para/`][ucmc-cam]). Limitación declarada: un solo plano de suelo. En KITTI, añadir CMC *empeoró* el resultado |
| **UAVDT** ([arXiv 1804.00518][uavdt]) | "Camera Motion" es un reto explícito de UAV. Trae atributos de altitud y vista útiles para estratificar, pero **no** un atributo de movimiento de cámara |
| **UAVMOT** (Liu et al., CVPR 2022, [PDF CVF][uavmot]) | Atribuye la falla de los trackers en UAV al "irregular motion caused by moving camera and view change". Propone un filtro de movimiento adaptativo. Evalúa en VisDrone2019 y UAVDT |
| **MMOT** ([arXiv 2510.12565][mmot], NeurIPS 2025) | **Descompone el desplazamiento entre frames** en componente del dron y del objeto con flujo KLT: 14.1 px (dron), 4.3 px (objeto) y 14.4 px (total) de media. Es un precedente directo de "medir el ego-movimiento" como covariable |
| **DynUAV** ([arXiv 2603.05970][dynuav], CVPR 2026) | Benchmark con vuelos agresivos (hover, crucero, rotación, zoom, órbitas), 42 secuencias (30/5/7). Usa la mejora por CMC como "proxy for the intensity of the ego-motion challenge" (AdapTrack: IDSW 1123→583, AssA 58.97→66.71). **No** define una métrica por frame del ego-movimiento |
| **MOT16** ([arXiv 1603.00831][mot16]) | La clase `static person` (7) se trata como distractora: es precedente de evaluar sin castigar ni premiar a los estáticos |
| **KITTI-MOD / MODNet** ([arXiv 1709.04821][kittimod]) | Genera etiquetas estático/dinámico de vehículos **automáticamente** a partir del GT de tracking y de la odometría. Es el precedente de "derivar `moving` del GT" (el abstract no detalla el procedimiento) |
| **SMOT4SB / SO-HOTA** ([arXiv 2507.12832][smot4sb]) | UAV con "motion entanglement" (cámara y objetos se mueven). Propone **SO-HOTA**: HOTA con distancia entre puntos en vez de IoU, para objetos pequeños |

**Conclusión:** la literatura mide el efecto del ego-movimiento casi siempre **indirectamente** (ablación CMC on/off o atributos por secuencia). Nadie publica un protocolo estándar de "IDSW condicionado al ego-movimiento" ni de "clasificación móvil/estático" para MOT desde UAV. Lo que sigue en 2.2–2.6 es **[Propuesta]** construida con piezas verificadas.

### 2.2 Variables de estratificación del ego-movimiento [Propuesta]

Se calculan **una sola vez por frame**, con un estimador **independiente de los trackers** (H offline, más robusta, con las cajas GT enmascaradas), y se guardan junto al GT.

| Variable | Definición | Por qué |
|---|---|---|
| `ego_px` | Desplazamiento mediano que H(t−1→t) produce en una rejilla de puntos, en px de la resolución de trabajo (equivale a nuestro `mov_camara_px`) | Intensidad total del ego-movimiento |
| `ego_rot_deg`, `ego_scale` | Rotación y escala de la parte lineal de H (√\|det A\|) | Yaw/zoom/cambio de altura, que rompen el IoU de otra forma que la traslación |
| `ego_px_1s` | Desplazamiento acumulado en una ventana de 1 s | Lo relevante para el decisor (`window_sec = 1.0`) |
| `H_resid_px` | Residuo de los puntos de fondo tras aplicar H | Calidad de la compensación: si es alto, fallan la estabilización o el plano |
| `H_ok` | Bandera de fiabilidad (inliers ≥ `min_inliers`) | Separa los fallos de la H de los de asociación |

**Bins sugeridos:** estacionario (p90 < 0.5 px/frame), suave (< 3 px/frame) y rápido (≥ 3 px/frame). Son los mismos umbrales que `tipo_toma` en v13, pero **por frame o por ventana**, no por video.

### 2.3 Métricas estándar a reportar (siempre)

HOTA, DetA, AssA, LocA, DetRe, DetPr, AssRe, AssPr, MOTA, MOTP, IDF1, IDP, IDR, IDSW, Frag, MT, PT, ML, FP, FN y número de IDs. Son las que calcula TrackEval ([hota.py L19-20][te-hota], [clear.py L22-26][te-clear], [identity.py L23][te-id]). Detalle en §3.

### 2.4 Métricas específicas de nuestro caso [Propuesta]

**a) Estratificación por ego-movimiento** (con lo que TrackEval ya calcula internamente; hay que exponer los emparejamientos por frame):
- **DetA por bin:** la suma de TP/FN/FP de los frames del bin. Es exacto, porque esos conteos son por frame.
- **AssA por bin:** la media de A(c) de los TP del bin. Es válido porque A(c) se define por TP (Ec. 14 de [HOTA][hota]); la alineación sigue siendo global, solo se agrega por bin.
- **HOTA por bin:** √(DetA_bin · AssA_bin) para cada α, promediado.
- **Tasa de IDSW por bin:** IDSW ocurridos en frames del bin / (GT-objetos·frame del bin) × 1000. Mismo cálculo para **Frag**.
- **IDSW condicionados:** para cada IDSW en t, registrar `ego_px` y `ego_rot` en [t−k, t]. Comparar con la distribución incondicional de esos valores (¿hay más IDSW de lo esperado tras un giro?).

**b) Móviles vs. estáticos (requiere la etiqueta GT `moving`, §2.5).** Dos protocolos complementarios:

| Protocolo | GT evaluado | Predicciones | Qué responde |
|---|---|---|---|
| **P1 — Tracking de móviles** | Solo los objetos con `moving = sí`. Los estáticos y los `dudoso` se marcan como **distractores** (regla MOT16/TrackEval) | **Todas** las salidas del tracker | ¿Sigue bien a lo que se mueve, sin que el número de estáticos lo distorsione? |
| **P1' — Tracking de estáticos** | Simétrico: los estáticos se evalúan y los móviles son distractores | Todas | ¿Mantiene el ID de lo quieto mientras el dron se mueve? (es justo lo que la H debería garantizar) |
| **P2 — De punta a punta "lo que se mueve"** | Solo los `moving = sí`. Los estáticos **no** son distractores; los `dudoso`, sí | **Solo las salidas con `en_movimiento = True`** | Sistema completo: tracker + decisor. Un estático marcado como móvil cuenta como FP; un móvil no marcado, como FN |

**c) Clasificación del decisor `en_movimiento`** (ningún toolkit lo evalúa):
- **Por frame, sobre los TP del emparejamiento CLEAR (α = 0.5):** matriz de confusión `en_movimiento` vs. GT `moving` (excluyendo los `dudoso`). Reportar **precisión, recall, F1 y especificidad**, global y por bin de ego-movimiento.
- **Por track GT:**
  - % de objetos estáticos marcados móviles en algún momento (**falsa alarma por track**).
  - % de móviles marcados móviles alguna vez (**detección por track**).
  - Fracción de su vida bien clasificada.
- **Latencia de encendido:** para cada tramo GT móvil (desde que empieza a moverse, o desde su primera aparición si ya se mueve), tiempo hasta que el track emparejado pasa a `en_movimiento`. Reportar mediana y p90 en segundos. Con la config actual (ventana de 1 s, `on_frames = 3`), se espera una latencia mínima del orden de 0.5–1 s; conviene confirmarlo en datos.
- **Latencia de apagado:** igual para las transiciones móvil→quieto (`off_frames = 10` la alarga a propósito).
- **Parpadeo:** cambios de estado por track-minuto, comparados con los cambios del GT.
- **Falsas alarmas por paralaje:** falsas alarmas sobre estáticos con `elevated = sí` o clases no vehiculares, en función de `ego_px_1s` (el paralaje crece con la traslación del dron). Reportar por separado la tasa de `paralaje = True` en estáticos (aciertos del filtro) y en móviles (falsos rechazos).

**d) Recuperación tras pérdidas** (propuesta de la conversación previa, que mantengo): tasa de mismo-ID al reaparecer, por longitud del hueco (en s) y **por bin de ego-movimiento durante el hueco**, junto con la tasa de "falsa recuperación".

**e) Tamaño de objeto:** todo lo anterior por área de caja (< 16², 16²–32², 32²–64², > 64² px a resolución original). Una caja de 16×16 px desplazada 3 px en x y en y tiene IoU = 169/343 ≈ 0.49 y ya no llega al umbral de 0.5. Como complemento, reportar **HOTA(α = 0.5)** y, si hay objetos diminutos, una variante de distancia de centro tipo SO-HOTA ([arXiv 2507.12832][smot4sb]).

### 2.5 Cómo derivar la etiqueta `moving` del GT [Propuesta]

1. **Estabilizar las trayectorias GT con una H de referencia independiente.** Se calcula offline, con más puntos y la resolución completa, y con las cajas GT enmascaradas para que los vehículos no contaminen el fondo. **No** debe ser la `comun.Estabilizador` (sería circular: el GT heredaría el error de la misma H que usan los trackers). Por ejemplo: ECC u ORB a resolución completa con RANSAC, bidireccional (t→t+1 y t+1→t, comprobando consistencia), o una reconstrucción SfM si la escena lo permite.
2. **Punto de apoyo:** el centro de la base de la caja en tomas oblicuas y el centro de la caja en cenitales. Es el mismo criterio que `ucmc_punto` (pitch < 30° → centro).
3. **Desplazamiento neto en una ventana W** (p. ej. 1 s centrada; puede ser *no causal*, porque el GT es offline): `d = ‖p(t+W/2) − p(t−W/2)‖`, con medianas en los extremos para quitar el jitter de la caja.
4. **Umbral relativo al tamaño:** `moving` si `d > max(τ_px, ρ·√área)`; `estático` si `d < τ_bajo`; `dudoso` en medio. Usar un umbral semántico ("más de ~0.3 longitudes de vehículo por segundo", ≈ 1.35 m/s para un auto de 4.5 m), **no los mismos valores que el decisor**, para no premiar su configuración.
5. **Revisión humana:** todos los `dudoso` y todos los cambios de estado se revisan a mano en CVAT. Lo que siga ambiguo queda como `dudoso`, que se ignora (distractor) en P1/P2 y se excluye de la matriz de confusión.
6. **Sesgos a vigilar:**
   - En zonas no planas (ladera de vigia_11), la H describe otro plano y genera "movimiento" falso. Esas zonas se marcan como ignoradas o `dudoso`.
   - En hover puro, el paralaje no se manifiesta, pero la estabilización es trivial.

### 2.6 Métricas sin GT: cómo validarlas

En los clips anotados se calculan **a la vez** las métricas sin GT de v13 y las de GT, y se comprueba:

| Métrica sin GT | Debería correlacionar con | Punto ciego |
|---|---|---|
| `tracks_por_objeto` | (IDs predichos emparejados) / (IDs GT), Frag, IDSW, AssRe | Las fusiones de identidad (ID transfer, AssPr bajo) **la bajan**, así que parece "mejor" |
| `vida_media_s`, `duracion_mediana_s` | MT/ML, AssRe | Lo mismo: un track que salta entre objetos vive mucho |
| `ids_por_minuto` | IDSW + nº de IDs | Depende de la densidad de objetos (no está normalizada) |
| `objetos_simultaneos_prom` | DetRe y DetPr | No separa los aciertos de los duplicados y los FP |
| `pct_frames_en_movimiento`, `cambios_estado_por_track` | Precisión y recall del decisor; parpadeo | Sin GT no se sabe cuál es la tasa correcta |

**Criterio [Propuesta]:**
- Kendall τ entre el ranking de trackers por cada métrica sin GT y por la de GT correspondiente, por clip.
- Si τ < ~0.6 o hay inversiones sistemáticas, esa métrica sin GT **no se usa para elegir tracker**, solo para detectar regresiones.
- Nota: con 4 trackers por clip, τ es muy ruidoso, así que se agrega sobre todos los clips y variantes (ablaciones).

---

## 3. Las métricas en detalle

Notación: TP/FN/FP son detecciones emparejadas, no emparejadas del GT y no emparejadas de la predicción. `gtDet` es el total de cajas GT. Las fórmulas son las de TrackEval [V].

### 3.1 Tabla resumen

| Métrica | Fórmula | Mide | Sesgo / cuidado | Fuente |
|---|---|---|---|---|
| **MOTA** | 1 − (FN + FP + IDSW) / gtDet | Exactitud global estilo CLEAR | Dominada por FP/FN (detección): el BoT-SORT paper lo dice ("focuses more on detection performance"). Puede ser negativa. No ve la localización. Cuenta los IDSW solo contra el TP anterior y **no** cuenta el *ID transfer* | [clear.py L178][te-clear]; [HOTA Ec. 2][hota]; [CLEAR MOT][clear] |
| **MODA** | (TP − FP) / (TP + FN) | MOTA sin IDSW | Solo detección | [clear.py L177][te-clear] |
| **MOTP** | Σ IoU(TP) / TP | Localización de los TP (a IoU ≥ 0.5) | En TrackEval es IoU medio (más alto, mejor); en py-motmetrics es una **distancia** ([Readme][pmm-readme]) | [clear.py L179][te-clear] |
| **sMOTA** | (Σ IoU(TP) − FP − IDSW) / (TP + FN) | MOTA con un "TP suave" | Poco usado | [clear.py L180][te-clear] |
| **IDSW** | TP cuyo prID difiere del último prID de ese gtID | Cambios de identidad | En TrackEval se compara con la *última vez* que el gtID estuvo emparejado, aunque haya sido hace muchos frames ([L62-63][te-clear]) | [clear.py][te-clear] |
| **Frag** | Σ por gtID de (nº de tramos seguidos − 1) | Interrupciones del seguimiento | Cuenta también las interrupciones que se recuperan con el mismo ID | [clear.py L122][te-clear] |
| **MT / PT / ML** | Fracción de la vida GT emparejada: > 0.8 / ≥ 0.2 / resto | "¿Lo seguí casi siempre?" | Ignora el ID: un objeto seguido por 5 IDs distintos puede ser MT | [clear.py L119-121][te-clear] |
| **IDF1** | 2·IDTP / (2·IDTP + IDFP + IDFN) | Proporción de detecciones con la **identidad global correcta** | Emparejamiento bijectivo global (un prID ↔ un gtID para toda la secuencia). Sobrevalora la asociación y es **no monótono respecto a la detección** ([HOTA §4.2, §9][hota]). Mira hacia el futuro: no es "online" | [identity.py L132-134][te-id]; [Ristani Ec. 9-11][ristani] |
| **IDP / IDR** | IDTP/(IDTP+IDFP), IDTP/(IDTP+IDFN) | Precisión y recall de identidad | IDR bajo indica fragmentación o pérdidas; IDP bajo, IDs sobrantes o falsos | [identity.py][te-id] |
| **DetA** | TP / (TP + FN + FP), por α | Detección (Jaccard) | Depende sobre todo del detector, pero el tracker la cambia: umbrales, `min_hits`, si emite predicciones sin detección | [hota.py L172][te-hota] |
| **DetRe / DetPr** | TP/(TP+FN), TP/(TP+FP) | Recall y precisión de detección | — | [hota.py][te-hota] |
| **AssA** | (1/TP) Σ_c A(c), A(c) = TPA/(TPA+FNA+FPA) | Calidad de asociación, promediada sobre los TP | Sola es **no monótona**: añadir un TP mal asociado la baja. Por eso se combina con DetA ([HOTA §9][hota]) | [hota.py L108][te-hota]; [HOTA Ec. 14][hota] |
| **AssRe / AssPr** | (1/TP) Σ TPA/(TPA+FNA), (1/TP) Σ TPA/(TPA+FPA) | AssRe bajo: un GT partido en varios IDs (fragmentación). AssPr bajo: un ID que cubre varios GT (fusión/ID transfer) | **Separan fragmentar de fusionar**, que es justo lo que nuestras métricas sin GT confunden | [hota.py L110-112][te-hota]; [HOTA Ec. 25-26][hota] |
| **LocA** | Σ IoU(TP)/TP, por α | Ajuste de las cajas | Afecta a los trackers que emiten la caja **filtrada por Kalman** (el propio, Ultralytics) frente a la detección cruda | [hota.py L115][te-hota] |
| **HOTA** | (1/19) Σ_α √(DetA_α · AssA_α), α = 0.05…0.95 | Detección, asociación y localización equilibradas | Media **geométrica**: si DetA o AssA es muy bajo, HOTA cae aunque el otro sea alto. Al promediar en α incluye la localización. Las implementaciones usan 19 valores de α (`np.arange(0.05, 0.99, 0.05)`) | [hota.py L17, L173][te-hota]; [HOTA Ec. 13, 16][hota] |
| **HOTA(0), LocA(0)** | HOTA y LocA en α = 0.05 | Rendimiento con una localización casi irrelevante | Útil para objetos diminutos | [hota.py L176-178][te-hota] |
| **OWTA** | √(DetRe · AssA) | Variante "open-world" (ignora los FP) | No aplica a nuestro caso | [hota.py L174][te-hota] |

### 3.2 Cómo leerlas juntas [Propuesta basada en las definiciones]

1. **Con el detector fijo**, las diferencias de **AssA, IDF1, IDSW/1000 y Frag** son del tracker. Las de **DetA** son en parte del tracker: confirmación (`min_hits`), umbrales `conf_high/low` y política de salida.
2. **AssRe vs. AssPr** explica la dirección del error. ByteTrack sin compensar, con el dron girando, debería mostrar AssRe bajo (fragmenta). Un tracker con umbrales permisivos o MMD muy ancha, AssPr bajo (fusiona).
3. **MOTA** solo para comparar con papers UAV antiguos (VisDrone Task 4B y UAVDT la reportan). Nunca para elegir.
4. **IDF1 vs. HOTA:** si IDF1 sube y HOTA no, el cambio está en la identidad de largo plazo; si sube DetA y no IDF1, en la detección.
5. **Agregación:** TrackEval combina secuencias **sumando conteos** (TP, FN, FP) y pondera AssA y LocA por TP ([hota.py L124-127][te-hota]). El HOTA "COMBINED" **no** es la media de los HOTA por secuencia. MOT16 justifica concatenar las secuencias por la variación en número de objetivos ([§4.1][mot16]).
6. **Online vs. offline:** HOTA, IDF1 y AssA juzgan la asociación con información de toda la secuencia. Para trackers causales cuya salida se usa en tiempo real, el paper propone **Online HOTA (OHOTA)** ([HOTA §8][hota]). **No está implementado en TrackEval** (no aparece entre los campos de `hota.py`, [V]).

---

## 4. El flujo completo del benchmark

### 4.1 Diagrama

```
                         ┌──────────────────────────────────────────────┐
  DATOS                  │ VIGIA (4–5 clips anotados) · VisDrone-MOT    │
                         │ (val / test-dev) · UAVDT (train / test)      │
                         └───────────────┬──────────────────────────────┘
                                         │ splits CONGELADOS (DECISIONS.md)
          ┌──────────────────────────────┼─────────────────────────────────┐
          ▼                              ▼                                 ▼
  ┌───────────────┐            ┌───────────────────┐           ┌─────────────────────────┐
  │ GT MOT .txt   │            │ DETECTOR FIJO     │           │ EGO-MOVIMIENTO (offline)│
  │ + ignorados   │            │ (yolo11n / -obb)  │           │ H_ref por frame, ego_px,│
  │ + visibilidad │            │ conf ≥ 0.05,      │           │ ego_rot, resid, H_ok    │
  │ + moving/dud. │            │ cacheado .parquet │           │ (independiente de los   │
  │ (CVAT → conv.)│            │ (hash de pesos,   │           │  trackers)              │
  └──────┬────────┘            │  imgsz, versión)  │           └───────────┬─────────────┘
         │                     └────────┬──────────┘                       │
         │                              │ mismas dets a todos              │
         │                              ▼                                  │
         │        ┌──────────────────────────────────────────────┐         │
         │        │ HARNESS (por secuencia, tracker nuevo)       │         │
         │        │  propio · UCMC · ByteTrack · BoT-SORT        │         │
         │        │  (+ baselines: OC-SORT… vía BoxMOT/Roboflow) │         │
         │        │  use_proposals=False · fps real · seeds      │         │
         │        │  frames decodificados (los 4 calculan H)     │         │
         │        └───────┬──────────────────────┬───────────────┘         │
         │                ▼                      ▼                         │
         │     ┌────────────────────┐  ┌───────────────────────────┐       │
         │     │ MOT .txt (1-based) │  │ lateral .parquet:         │       │
         │     │ frame,id,x,y,w,h,  │  │ frame,id,en_movimiento,   │       │
         │     │ conf,cls,-1        │  │ disp_px,paralaje,H_ok,ms  │       │
         │     └─────────┬──────────┘  └─────────────┬─────────────┘       │
         ▼               ▼                           │                     │
  ┌───────────────────────────────────────┐          │                     │
  │ PREPROCESADO ÚNICO (versionado)       │          │                     │
  │ ignorados por ÁREA ≥ 50 % (VisDrone)  │          │                     │
  │ clases → 'vehicle' · distractores     │          │                     │
  │ frames anotados (submuestreo)         │          │                     │
  └──────────────────┬────────────────────┘          │                     │
                     ▼                               │                     │
  ┌───────────────────────────────────────┐          │                     │
  │ TrackEval (trackeval==1.3.0, fork)    │          │                     │
  │ HOTA · CLEAR · Identity · Count       │          │                     │
  │ + volcado de matches por frame/α      │          │                     │
  └──────────────────┬────────────────────┘          │                     │
                     ▼                               ▼                     ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ ANÁLISIS: por secuencia · por bin de ego-movimiento · P1/P1'/P2 (móviles)    │
  │ decisor (P/R/F1, latencias, paralaje) · tamaño de objeto · recuperación      │
  └──────────────────────────────────┬───────────────────────────────────────────┘
                                     ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ ESTADÍSTICA: Δ pareado por secuencia/bloque · bootstrap IC95 % · Wilcoxon    │
  │ Kendall τ entre rankings · sensibilidad a hiperparámetros                    │
  └──────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Pasos

1. **Congelar el alcance:** clases (`vehicle` como evaluación principal), umbrales de tamaño, frames anotados, definición de `moving`, métrica objetivo para el ajuste (p. ej. HOTA) y tabla de reporte. Todo en un `DECISIONS.md` **antes** de ver resultados.
2. **GT:**
   - VIGIA: anotar según §1.6.
   - VisDrone y UAVDT: convertir a MOT con su regla de ignorados.
   - Prueba de sanidad: evaluar **el GT como predicción → HOTA ≈ 100**. Si no da, hay un error de convención (frame base 0/1, xywh/xyxy, clases).
3. **Detector fijo y cacheado:**
   - Una pasada por secuencia con umbral **bajo** (≤ 0.05–0.10, por debajo de `conf_low = 0.10`, porque los trackers hacen una segunda ronda con las detecciones débiles).
   - Se guardan las cajas, la confianza, la clase y la OBB original si existe.
   - Hash de pesos, `imgsz`, versión de Ultralytics y dispositivo.
   - **Auditoría de leakage:** registrar con qué datos se entrenó el detector. Hoy usamos COCO y DOTA-OBB preentrenados, sin VIGIA, así que no hay leakage en VIGIA.
4. **Ego-movimiento offline:** `H_ref` y covariables por frame (§2.2). Sirven para estratificar y para generar `moving`, **nunca** como entrada de los trackers.
5. **Trackers:**
   - Una instancia nueva por secuencia (nuestros trackers "no guardan estado entre videos").
   - Mismas detecciones, mismo fps real y `use_proposals=False` en los 4.
   - **Política de salida explícita.** Los 4 actuales emiten solo los tracks confirmados *con detección en el frame*. Si se agregan baselines de BoxMOT o Roboflow, deben configurarse igual, o hay que reportar la diferencia, porque emitir predicciones sin detección cambia DetA.
6. **Formato de salida:**
   - MOT `.txt` con 10 columnas, frames 1-based: `frame,id,x,y,w,h,conf,cls,-1,-1` (BoxMOT escribe 9 columnas, [motmetrics.py L31][bm-mm-cols]).
   - Archivo lateral con `en_movimiento`, `desplazamiento_px`, `paralaje`, `H_ok` y ms por etapa (`tiempos_ms`).
7. **Evaluación:**
   - TrackEval como **único** juez, con un dataset class propio (copia del de MOTChallenge con `valid_classes = ['vehicle']`, distractores configurables y la regla de área para ignorados).
   - **Exportar los emparejamientos por frame y por α.** TrackEval no los expone directamente; hay que subclasear sus métricas o recomputarlos con su misma lógica.
   - **Validación única:**
     - mismo resultado que py-motmetrics 1.4.0 (que trae prueba de paridad con TrackEval, [Readme][pmm-readme]);
     - en VisDrone, mismo MOTA/IDF1/IDSW que el toolkit oficial, o diferencia explicada.
8. **Análisis estratificado:** §2.4. Siempre con N (secuencias, GT-objetos y frames) junto a cada celda.
9. **Estadística y repetibilidad:**
   - **Determinismo:** con las detecciones y la H cacheadas, los trackers deberían ser deterministas. Se comprueba corriendo dos veces y comparando los archivos byte a byte. El RANSAC de `Estabilizador` (OpenCV) es aleatorio salvo con semilla fija (`cv2.setRNGSeed`); hay que fijarla y registrarla.
   - **Incertidumbre:** viene de *qué* secuencias se evalúan. Usar Δ pareado por secuencia y bootstrap (≥ 10 000 remuestreos) **recombinando conteos** como TrackEval, no promediando HOTA.
   - **Con pocos clips VIGIA:** bootstrap por **bloques temporales** (p. ej. segmentos de 5–10 s) dentro de cada clip, asumiendo la autocorrelación. Reportar "gana en k/n secuencias".
   - **Latencia:** sí requiere repeticiones (mediana y p95, con calentamiento).
10. **Splits para ajustar y para test (sin leakage):**
    - **VisDrone:** ajustar en `val` (7 secuencias) y reportar en `test-dev` (17). Ojo: el perfil `visdrone` de BoxMOT llama `val` a test-dev ([visdrone.yaml][bm-visdrone]).
    - **UAVDT:** ajustar en un subconjunto de train y reportar en test (20).
    - **VIGIA:**
      - O se ajusta en VisDrone/UAVDT y se usa VIGIA *solo* como test de transferencia (recomendado, porque hay pocos clips);
      - o se separan 1–2 clips anotados solo para ajustar y nunca se reportan.
    - **Mismo presupuesto de ajuste para los 4** (p. ej. 100 pruebas de Optuna sobre un espacio declarado de antemano).
    - Reportar la columna "por defecto" y la "ajustada". El README del proyecto ya fija que los ruidos del Kalman del propio "no se ajustan con los videos de evaluación"; mantener esa regla para todos.

---

## 5. Integrar nuestros 4 trackers en BoxMOT

### 5.1 Estado verificado de BoxMOT

- Versión `26.0.0`, licencia AGPL-3.0 ([pyproject L125-129][bm-pyproject]; API de GitHub: `spdx_id: AGPL-3.0`).
- Python ≥ 3.10, < 3.14; dependencias gestionadas con `uv` ([AGENTS.md][bm-agents]).
- Comandos CLI: `track`, `materialize`, `eval`, `tune`, `research`, `export`… ([docs/modes][bm-doc-modes]).

### 5.2 Contrato de un tracker

- **Clase base:** `BaseTracker` ([base.py L60-85][bm-base]), o `BoxTracker` para estado AABB/OBB ([box/base.py][bm-boxbase]). Atributos que declara la subclase: `_requires_frame`, `use_embeddings`, `_requires_masks`, `_requires_camera`, `_requires_ego_motion`, `supports_obb`, etc.
- **Entrada pública:** `update(detections, frame=None, *, timestamp_s=None, detections_3d=None, camera=None, camera_to_world=None)` ([base.py L351-471][bm-base-update]).
  - `detections` es un `np.ndarray` **N×6 `[x1,y1,x2,y2,conf,cls]`** (AABB) o N×7 (OBB), o un objeto `Detections`.
  - Valida `x2 > x1`, `y2 > y1` ([input.py L78-93][bm-input]).
- **Lo que implementa la subclase:** `_track_detections(dets, img, embs=None, masks=None) -> np.ndarray` ([base.py L877-886][bm-base-track]).
  - `dets` llega como N×6 con la clase recodificada a un id interno.
  - Debe devolver **M×8 `[x1,y1,x2,y2,track_id,conf,cls,det_idx]`** (`track_cols = 8`, [layout.py L55][bm-layout]).
  - `det_idx` debe ser un entero menor que N ([base.py L686-687][bm-base-arrays]) y la conf debe estar en [0, 1] ([input.py L231-232][bm-input-pack]).
- **`reset()`:** reinicia el estado de la secuencia ([base.py L920-948][bm-base-reset]).
- **Cuándo llega la imagen a `_track_detections`:** solo si el tracker la declara necesaria. Si no, solo se registran sus dimensiones ([base.py L606-616][bm-base-arrays]).

### 5.3 Registrar un tracker nuevo

Según la guía oficial ([add-tracker.md][bm-doc-add]) y el código:

1. `boxmot/trackers/<nombre>/tracker.py` (clase) y `config.py` (dataclass `<Clase>Config` congelada).
2. Una entrada en **`_TRACKER_MANIFEST`** (`class_path`, `config_class_path`) en [manifest.py L22-76][bm-manifest].
3. Una entrada en **`_TRACKER_CAPABILITIES`** en [registry.py L15][bm-registry]. Si las claves no coinciden con el manifest, el módulo lanza `RuntimeError` al importarse ([registry.py L94-97][bm-registry]).
4. `boxmot/configs/trackers/<nombre>.yaml` con metadatos de búsqueda (`type`, `default`, `range` u `options`) para `tune` ([bytetrack.yaml][bm-bt-yaml]).
5. Docs y tests.

`get_tracker_config(name)` resuelve la ruta YAML desde el registro ([registry.py L170][bm-registry]). `create_tracker(spec, config=..., **overrides)` construye desde el nombre o un `TrackerSpec` ([factory.py L235][bm-factory]).

**No existe registro externo:** no hay `entry_points` ni `register_tracker` (búsqueda sin resultados en `boxmot/`). Pasar una instancia no registrada falla: "The provided tracker instance is not registered" ([specs.py L222][bm-specs]). **Usar `boxmot eval/tune` con nuestros trackers obliga a hacer fork** o a parchear el diccionario en tiempo de ejecución (frágil).

### 5.4 Trackers que trae

| Tracker | En v26.0.0 | En `master` | Necesita frame | Embeddings |
|---|---|---|---|---|
| ByteTrack | ✓ | ✓ | opcional (para CMC) | — |
| BoT-SORT | ✓ | ✓ | sí si `use_cmc` (por defecto True) | sí (`use_embeddings: True` por defecto) |
| OC-SORT | ✓ | ✓ | opcional | — |
| Deep OC-SORT | ✓ | ✓ | opcional | sí |
| StrongSORT | ✓ | ✓ | opcional | **requiere** |
| HybridSORT | ✓ | ✓ | opcional | sí |
| BoostTrack | ✓ | ✓ | opcional | sí |
| OccluBoost | ✓ | ✓ | opcional | sí |
| SFSORT | ✓ | ✓ | opcional | — |
| MAF-HDA (máscaras) | ✓ | ✓ | — | requiere máscaras |
| EagerMOT (2D+3D) | ✓ | ✓ | — | requiere 3D y cámara |
| EdgeTAM | ✗ | ✓ | requiere | — |
| **UCMCTrack** | ✗ | ✗ | — | — |
| **ImprAssoc** | ✗ | ✗ | — | — |

Fuentes: [manifest.py][bm-manifest] en master; [manifest.py en v26.0.0][bm-v26-manifest]; capacidades en [registry.py][bm-registry].

### 5.5 `materialize`, `eval` y `tune`

- **Builds de percepción:** `materialize` ejecuta el detector (y el ReID) **una vez** y publica un build inmutable con manifiesto e identificador. `eval` y `tune` lo reproducen sin volver a inferir ([eval.md][bm-doc-evalmode], [materialize.md][bm-doc-mat]). Si se omite `--build`, se prepara o reutiliza uno compatible. Las detecciones se cachean en `.cache/detect` ([evaluation.md][bm-doc-eval]).
- **Detector:** sale de sus backends (`ultralytics`, `yolox`, `rtdetr`). Se puede usar un checkpoint local creando un perfil YAML con `image_size`, `confidence_threshold` y `checkpoints` ([detectors.md][bm-doc-det]). **No hay forma de cargar *nuestro* caché de detecciones** en el layout MOT: las "saved 2D detections" solo existen en la codificación `trackrcnn` del flujo KITTI, que además no admite `tune` ([experiments.md][bm-doc-exp]).
- **Datasets incluidos:** `mot17`, `mot17-mini`, `mot20`, `dancetrack`, `sportsmot`, `kitti-mots`, `mmot`, `mmot-mini`, `sensor-fusion` y `visdrone` ([configs/datasets][bm-cfg-ds]). **No incluye UAVDT.** El perfil `visdrone` usa `storage.root: VisDrone2019-MOT-test-dev` con split `val`, y declara `ignored-regions: 0` y `others: 11` como `ignore` ([visdrone.yaml][bm-visdrone]).
- **Dataset propio:** sí, con `format.layout: mot` y `seqinfo.ini` (`frameRate`), o `timestamps.csv` si los tiempos son irregulares ([datasets.md L96-130][bm-doc-ds]). VIGIA tendría que convertirse a frames (`img1/000001.jpg`…) y a `gt/gt.txt`.
- **Evaluador:**
  - In-repo, con HOTA, CLEAR, Identity y Count ([motmetrics.py][bm-mm]) y `HOTA_ALPHA_VALUES = arange(0.05, 0.99, 0.05)` ([L30][bm-mm]).
  - TrackEval queda solo como **referencia opcional** para KITTI, fijado en `trackeval==1.3.0` ([trackeval_reference.py][bm-te-ref]).
  - **Regiones ignoradas:** por emparejamiento IoU ≥ 0.5 contra cajas de clase ignorada ([motmetrics.py L488-506][bm-mm-aabb]), **no** por área como el toolkit oficial ([dropObjects.m][vdt-drop]).
- **`tune`:** Ray Tune + Optuna/HyperOpt/random. Por defecto hace 10 pruebas maximizando HOTA. Objetivos posibles: HOTA, MOTA, IDF1, AssA, AssRe, IDSW, IDs e IDSW_rate. Puede calibrar el Kalman con `--calibrate-kf` y ajustar ruidos con `--tune-kf` ([tune.md][bm-doc-tune]).
- **CMC en eval/tune:**
  - Política por secuencia (`--cmc-policy auto|off|PATH`, y `tune --select-kf --select-cmc`).
  - Las transformadas se guardan en `<build>/cmc/<clave>/warps.npz` y se reutilizan ([materialize.md L365-382][bm-doc-mat]; [eval/cmc.py][bm-eval-cmc]).
  - En el tracker se inyectan con `set_prepared_cmc(warps)` → `PreparedCMC`, que **exige forma (N, 2, 3)** y la identidad en el frame 0 ([box/base.py L71-85][bm-boxbase]; [prepared.py L22-30][bm-prepared]).

### 5.6 El GMC de BoxMOT

- Registro de métodos `ecc`, `orb`, `sof` (sparse optical flow) y `sift` ([cmc/registry.py L37-42][bm-cmc-reg]).
- Contrato `BaseCMC.apply(img, dets) -> afín 2×3 u homografía 3×3`, con reducción de la imagen por defecto (`scale = 0.15`) ([cmc/base.py][bm-cmc-base]).
- BoT-SORT: `use_cmc=True`, `cmc_method="sof"` ([config.py L55-56][bm-bs-cfg]). ByteTrack: `use_cmc=False` ([config.py L52-53][bm-bt-cfg]).
- La ruta *precalculada* solo acepta afines 2×3 (§5.5).

### 5.7 Licencia AGPL-3.0: implicaciones

- La AGPL-3.0 obliga a ofrecer el código fuente correspondiente a quien reciba el programa (distribución) y, además, a los usuarios que interactúen **por red** con una versión **modificada** ([§13 AGPL-3.0][agpl]).
- **Uso interno de investigación** (correr benchmarks en nuestras máquinas sin distribuir ni ofrecer un servicio): no dispara esas obligaciones.
- **Si un fork de BoxMOT con nuestros trackers se distribuye, o se ofrece como servicio**, hay que publicar el código combinado bajo AGPL. Esto afecta al tracker propio si se integra *dentro* del fork.
- **Ultralytics ya es AGPL-3.0** (metadata del paquete instalado: `License: AGPL-3.0`), y `tracker_bytetrack.py` y `tracker_botsort.py` ya dependen de él, así que BoxMOT **no introduce** una licencia nueva en el proyecto. Sí lo haría si el producto final quisiera evitar AGPL: en ese caso conviene **no** acoplar el tracker propio ni UCMC (MIT) a BoxMOT ni a Ultralytics. Roboflow `trackers` es Apache-2.0 ([LICENSE][rf-license]).
- *No es asesoría legal.*

### 5.8 Plan concreto [Propuesta]

**Recomendación principal: no meter los 4 trackers dentro de BoxMOT.** Usar BoxMOT, como mucho, como **biblioteca** desde nuestro harness. Si aun así se quiere `boxmot eval/tune`, este sería el plan.

**a) Adaptador genérico `AdaptadorBoxmot(BaseTracker)`, uno por tracker:**

```python
class PropioBoxmot(BaseTracker):           # fork: boxmot/trackers/propio/tracker.py
    _requires_frame = True                  # los 4 calculan H con la imagen
    def __init__(self, fps=30.0, config=None, **kw):
        super().__init__(**kw)
        self._fps, self._cfg = fps, config or ConfigPropio(use_proposals=False)
        self._trk = TrackerPropio(self._fps, self._cfg, self.class_names)
    def _track_detections(self, dets, img, embs=None, masks=None):
        salida = self._trk.actualizar(img, dets[:, :6])           # BGR original + Nx6
        arr = como_array(salida)                                  # Nx7 [x1,y1,x2,y2,id,conf,cls]
        det_idx = asignar_por_iou(arr[:, :4], dets[:, :4])        # BoxMOT exige el índice de detección
        cls = np.where(arr[:, 6] < 0, dets[det_idx, 5], arr[:, 6])  # clase -1 → la de la detección
        self._log_lateral(salida)                                 # en_movimiento, paralaje, disp, H_ok
        return np.column_stack([arr[:, :4], arr[:, 4], arr[:, 5].clip(0, 1), cls, det_idx])
    def reset(self):
        super().reset(); self._trk = TrackerPropio(self._fps, self._cfg, self.class_names)
```

Puntos que obliga el contrato de BoxMOT (todos verificados en el código citado en §5.2):
- **`det_idx`:** nuestros trackers no lo devuelven. Como solo emiten tracks *con detección en el frame*, se recupera emparejando la caja de salida con la detección de entrada por IoU. Es mejor añadir el índice en `comun.Track`, porque la caja del propio está filtrada por Kalman y el IoU puede elegir mal en zonas densas.
- **Clase −1:** `TrackSalida.clase` puede ser −1 ("desconocido") hasta reunir `cls_min_hits` votos. BoxMOT valida las clases contra su catálogo ([base.py L599-605][bm-base-arrays]), así que hay que mapear −1 a la clase de la detección.
- **Resolución:** `actualizar()` espera el **frame original** y reduce internamente a 960 px. BoxMOT pasa el frame del build; hay que asegurarse de que es el original (4K) o ajustar `work_width`.
- **fps:** BoxMOT usa `max_lost_frames` en *updates*, sin escalar por fps ([bytetrack.yaml][bm-bt-yaml]). Nuestros trackers usan segundos (`max_miss_sec`), así que se pasa el `frameRate` de `seqinfo.ini` al constructor.
- **OBB:** si el build es OBB (detector `yolo11n-obb`), BoxMOT entrega geometría OBB (N×7). Nuestros trackers son AABB, así que hay que convertir antes (en un detector profile AABB) o declarar `supports_obb = False`.

**b) Qué hacer con la estabilización H propia:**
- Dejarla **dentro** del tracker propio y de UCMC, porque es parte del método.
- Para la **ablación limpia de la compensación** (pregunta 6 de la propuesta), cachear `H_ref` por frame y añadir un modo de inyección (`TrackerPropio(..., H_externa=...)`).
- Esto **no** se puede hacer con `PreparedCMC` de BoxMOT (exige 2×3). Opciones:
  - aproximar H por su afín (pierde la perspectiva, que importa en tomas oblicuas);
  - o hacer la ablación **fuera de BoxMOT** (harness propio).
- Para comparar contra el GMC de BoT-SORT con la misma compensación: Roboflow `BoTSORTTracker.update(..., cmc_transform=A)` acepta una afín 2×3 externa ([tracker.py L232-260][rf-botsort]).

**c) Calibración UCMC:** es causal y en línea. Necesita el frame (para H), `hfov` (dato de cámara), `pitch_respaldo_deg` y opcionalmente `altura_m`, y estos **cambian por video**. En BoxMOT no hay canal para parámetros por secuencia de un tracker 2D: el `camera` de `update()` dispara la ruta multimodal ([base.py L384-401][bm-base-update]). Habría que leerlos de un archivo lateral por secuencia en el constructor del adaptador. BoxMOT **no tiene UCMCTrack**, así que entra solo como tracker nuevo vía fork.

**d) El decisor `en_movimiento`:** el `Tracks` de BoxMOT solo lleva geometría, IDs, conf, clase, índice de detección y máscaras ([base.py L723-735][bm-base-arrays]). **No hay canal para atributos por track**, y `eval` no evaluaría el decisor aunque lo hubiera. Hay que escribirlo en un **archivo lateral** (Parquet: `seq, frame, id, en_movimiento, disp_x, disp_y, paralaje, H_ok`) y evaluarlo con nuestro propio código (§2.4c), uniéndolo por (frame, id) a los emparejamientos de TrackEval.

**e) `use_proposals`:** forzar `Config(use_proposals=False)` en los 4 adaptadores. Hay que incluirlo en el hash de configuración del run.

**f) Configs YAML para `tune`:** un `propio.yaml` con espacio de búsqueda solo para lo que se declara ajustable (`match_iou`, `box_pad`, `conf_high`, `conf_low`, `max_miss_sec`, `min_hits`) y **no** para los ruidos del Kalman, que se quedan congelados por regla del proyecto. El mismo número de pruebas para los 4.

### 5.9 ByteTrack/BoT-SORT de BoxMOT vs. Ultralytics

| Aspecto | Ultralytics 8.4.192 (lo que usamos) | BoxMOT `master` |
|---|---|---|
| Umbrales ByteTrack por defecto | `track_high 0.25`, `track_low 0.1`, `new_track 0.25`, `match 0.8`, `track_buffer 30`, `fuse_score True` ([bytetrack.yaml][ul-bt]) | `track_thresh 0.6`, `min_conf 0.1`, `match_thresh 0.9`, `second_match 0.5`, `unconfirmed 0.7`, `max_lost_frames 30` ([config.py L39-46][bm-bt-cfg]) |
| Pérdida de tracks | `track_buffer` escalado por `frame_rate/30` (en nuestro adaptador se fija explícitamente en frames) | `max_lost_frames` "without FPS scaling" ([bytetrack.yaml][bm-bt-yaml]) |
| CMC en ByteTrack | No tiene | Opcional (`use_cmc`, por defecto False) |
| BoT-SORT GMC | `gmc_method: sparseOptFlow`, ReID desactivado por defecto ([botsort.yaml][ul-bs]); `downscale=2` ([gmc.py][ul-gmc]) | `use_cmc: True`, `cmc_method: sof`, `use_embeddings: True` por defecto, umbrales ajustados con decimales, `max_lost_frames 40` ([config.py L48-70][bm-bs-cfg]) |
| Filtro de Kalman | XYAH (BoT-SORT: XYWH) | Elegible (`xyah`, `xywh`, `xysr`, `xyhr`, `xyscr`) y calibrable |
| Licencia | AGPL-3.0 | AGPL-3.0 |

**Implicación:** los números de "BoT-SORT" con BoxMOT y con Ultralytics **no son intercambiables**. En nuestra comparativa hay que fijar *una* implementación de cada uno, con su versión. Lo más coherente con la historia v9–v14 es mantener Ultralytics (nuestro `adaptador_ultralytics.py` ya absorbe sus cambios de API). Las versiones de BoxMOT se añaden como **baselines extra**, etiquetadas como tales.

---

## 6. ¿Es BoxMOT la mejor opción? 

| Opción | Pros (verificados) | Contras (verificados) | Encaje con nosotros |
|---|---|---|---|
| **TrackEval directo + harness propio** | Implementación de referencia de HOTA, escrita por sus autores ([hota.py][te-hota]). MIT. Dataset classes extensibles. Las detecciones y la H cacheadas son *nuestras*; los 4 trackers se usan tal cual, sin adaptadores de API ajena. El lateral de `en_movimiento` es trivial | El original no se mantiene desde 2022 y usa `np.float` (falla con NumPy ≥ 1.24). Hay que usar el fork `trackeval==1.3.0` ([kovalp/TrackEval][te-fork]) o parchear. El loader MOTChallenge solo admite `pedestrian` ([L71][te-mot-71]), así que hace falta un dataset class propio con la regla de área para ignorados. Tuning y bootstrap corren por nuestra cuenta | **Mejor encaje** |
| **BoxMOT** | Muchos baselines (11–12 trackers), builds de percepción inmutables, CMC cacheada, `tune` con Ray/Optuna, calibración de KF, perfil VisDrone y soporte de MMOT (OBB) | Sin registro externo (fork obligatorio). Detector propio, no el nuestro. CMC precalculada solo 2×3. Ignorados VisDrone ≠ regla oficial. Sin UAVDT. Sin canal para atributos (`en_movimiento`). AGPL. API cambiante (5 *majors* en 5 meses) | **Bueno como fuente de baselines**, malo como juez y como contenedor |
| **Roboflow `trackers` 2.7.0** | Apache-2.0. SORT, ByteTrack, OC-SORT, BoT-SORT, C-BIoU y McByte ([src/trackers/core][rf-core]). **BoT-SORT acepta `cmc_transform` 2×3 externo** ([tracker.py][rf-botsort]). Evaluador alineado con TrackEval, con clases configurables (`MOTClassConfig`). Su metodología separa los splits de ajuste y test ([methodology.md][rf-method]) | Datasets incluidos: solo MOT17 y SportsMOT ([manifest.py][rf-manifest]). Ignorados solo por clase distractora, no por región. API basada en `supervision.Detections` | **Buena alternativa a BoxMOT** para baselines y para la ablación de GMC con compensación inyectada |
| **Ultralytics** | Ya integrado (ByteTrack, BoT-SORT) | No es un harness de benchmark: no calcula HOTA ni tiene `eval` MOT. AGPL | Ya cubierto |
| **MMTracking / MMDetection** | Apache-2.0. Tiene trackers clásicos | MMTracking: última release v1.0.0rc1 (2022) y último push 2023-09. MMDetection: último push 2024-08 (API de GitHub) | **No recomendado** (estancado) |
| **py-motmetrics 1.4.0** | MIT. CLEAR, Identity y HOTA. **Prueba de paridad con TrackEval** ([Readme][pmm-readme]). Útil para tests unitarios | MOTP como distancia (ojo al convertir). No es la referencia de HOTA | **Validador cruzado** |
| **Toolkit VisDrone (MATLAB)** | Regla oficial de ignorados ([dropObjects.m][vdt-drop]) y comparabilidad con papers VisDrone | MATLAB. Sin mantenimiento desde 2019. Sin HOTA | **Validación única** de nuestro preprocesado |

**Recomendación:**
1. **El evaluador único es TrackEval (fork 1.3.0)**, con un dataset class propio que haga:
   - superclase `vehicle`;
   - ignorados por área ≥ 50 %;
   - distractores `moving` y `dudoso` para P1/P1'/P2;
   - submuestreo de frames anotados;
   - volcado de los emparejamientos por frame.
2. **Harness propio delgado:**
   - lee detecciones y `H_ref` cacheadas;
   - instancia los 4 trackers tal cual (`use_proposals=False`);
   - escribe MOT `.txt` y el lateral;
   - mide la latencia por etapa con `tiempos_ms`.
3. **Baselines extra** (OC-SORT, BoostTrack, Deep OC-SORT…): llamar a `boxmot.create_tracker(...)` o a Roboflow **desde el harness**, con nuestras detecciones (su `update(dets_Nx6, frame)` acepta arrays). Así no se usa su `eval` ni hace falta fork. Para la ablación del GMC con transformada inyectada, Roboflow BoT-SORT (`cmc_transform`).
4. **Validación una sola vez:**
   - GT→GT = 100;
   - TrackEval vs. py-motmetrics en 2–3 secuencias;
   - nuestro preprocesado VisDrone vs. el toolkit oficial en 2 secuencias;
   - opcional: `boxmot eval` en VisDrone para **documentar** la diferencia por ignorados.
5. **Trade-offs:**
   - Se pierde el `tune` distribuido de BoxMOT; se reemplaza por Optuna directo (un bucle de ~50 líneas).
   - Se gana control del protocolo, estabilidad frente a cambios de API, la posibilidad de evaluar el decisor y no tener que hacer fork AGPL de nuestro código.

---

## 7. Decisiones abiertas

1. **Clips VIGIA a anotar** (§1.6): cuáles, qué tramo, a qué fps de anotación, y si vigia_2 (~78 objetos) se recorta a una región.
2. **Definición operativa de `moving`:** umbral semántico (m/s o longitudes de vehículo por segundo), ventana, tratamiento de `dudoso` y de laderas o zonas no planas.
3. **Estimador de `H_ref` para el GT:** ECC/ORB a resolución completa, bidireccional o SfM. Debe ser independiente de `comun.Estabilizador`.
4. **Política de salida común:** ¿todos emiten solo tracks con detección en el frame (como hoy), o también predicciones? Afecta a DetA y LocA.
5. **Evaluación principal por superclase `vehicle` o por clase:** nuestros trackers no asocian car con truck ("un car no se une con un truck"), así que el parpadeo de clase genera IDSW que no son de asociación.
6. **Ajuste:** ¿en VisDrone/UAVDT con transferencia a VIGIA, o en 1–2 clips VIGIA reservados? Presupuesto (N pruebas) y objetivo (HOTA o HOTA + IDF1).
7. **¿BoT-SORT/ByteTrack de Ultralytics, de BoxMOT o ambos?** Recomiendo Ultralytics como principal (continuidad con v9–v14) y BoxMOT/Roboflow como baselines etiquetados.
8. **¿Se distribuirá el sistema final?** Si se quiere evitar AGPL, mantener el propio y UCMC desacoplados de Ultralytics y BoxMOT.
9. **Datasets con ego-movimiento fuerte:** ¿incorporar MMOT (ya tiene perfil en BoxMOT; es multiespectral y OBB) o DynUAV (licencia y disponibilidad NO VERIFICADAS)?
10. **OHOTA:** ¿vale la pena implementar Online HOTA ([HOTA §8][hota]) para reflejar el uso en tiempo real? No está en TrackEval.
11. **Semilla RANSAC:** fijar `cv2.setRNGSeed` en `Estabilizador` para que las corridas sean reproducibles (hay que verificar que `findHomography` la respeta).

---

## 8. Fuentes

### Papers (fuente primaria)
- Luiten et al., *HOTA: A Higher Order Metric for Evaluating Multi-Object Tracking*, IJCV 2020/2021 — [arXiv 2009.07736][hota]
- Bernardin & Stiefelhagen, *Evaluating Multiple Object Tracking Performance: The CLEAR MOT Metrics*, EURASIP JIVP 2008 — [doi:10.1155/2008/246309][clear]
- Ristani et al., *Performance Measures and a Data Set for Multi-Target, Multi-Camera Tracking*, ECCVW 2016 — [arXiv 1609.01775][ristani]
- Milan et al., *MOT16: A Benchmark for Multi-Object Tracking* — [arXiv 1603.00831][mot16]
- Dendorfer et al., *MOTChallenge: A Benchmark for Single-Camera Multiple Target Tracking*, IJCV 2021 — [arXiv 2010.07548][motchallenge]
- Zhu et al., *Detection and Tracking Meet Drones Challenge*, TPAMI 2021 — [arXiv 2001.06303][visdrone-paper]
- Du et al., *The Unmanned Aerial Vehicle Benchmark: Object Detection and Tracking* (UAVDT), ECCV 2018 — [arXiv 1804.00518][uavdt]
- Aharon et al., *BoT-SORT: Robust Associations Multi-Pedestrian Tracking* — [arXiv 2206.14651][botsort-paper]
- Yi et al., *UCMCTrack: Multi-Object Tracking with Uniform Camera Motion Compensation*, AAAI 2024 — [arXiv 2312.08952][ucmc-paper]
- Liu et al., *Multi-Object Tracking Meets Moving UAV*, CVPR 2022 — [CVF][uavmot]
- Li et al., *MMOT: The First Challenging Benchmark for Drone-based Multispectral MOT*, NeurIPS 2025 — [arXiv 2510.12565][mmot]
- Ye et al., *Breaking Smooth-Motion Assumptions: A UAV Benchmark for MOT…* (DynUAV), CVPR 2026 — [arXiv 2603.05970][dynuav]
- Siam et al., *MODNet: Moving Object Detection Network…* (KITTI-MOD) — [arXiv 1709.04821][kittimod]
- SanMiguel, Muñoz, Poiesi, *Detection-aware multi-object tracking evaluation* (TEM) — [arXiv 2212.08536][tem]
- *MVA 2025 Small Multi-Object Tracking for Spotting Birds Challenge* (SO-HOTA) — [arXiv 2507.12832][smot4sb]

### Código y documentación (con commit)
- BoxMOT `master` [`fab3fe0`][bm]: [manifest.py][bm-manifest], [registry.py][bm-registry], [specs.py][bm-specs], [factory.py][bm-factory], [base.py][bm-base], [box/base.py][bm-boxbase], [layout.py][bm-layout], [input.py][bm-input], [cmc/registry.py][bm-cmc-reg], [cmc/base.py][bm-cmc-base], [cmc/prepared.py][bm-prepared], [eval/motmetrics.py][bm-mm], [eval/trackeval_reference.py][bm-te-ref], [eval/cmc.py][bm-eval-cmc], [bytetrack/config.py][bm-bt-cfg], [botsort/config.py][bm-bs-cfg], [configs/trackers/bytetrack.yaml][bm-bt-yaml], [configs/datasets/visdrone.yaml][bm-visdrone], [pyproject.toml][bm-pyproject]; docs: [add-tracker.md][bm-doc-add], [evaluation.md][bm-doc-eval], [eval.md][bm-doc-evalmode], [materialize.md][bm-doc-mat], [tune.md][bm-doc-tune], [datasets.md][bm-doc-ds], [detectors.md][bm-doc-det], [experiments.md][bm-doc-exp]; release [v26.0.0][bm-v26] ([manifest v26][bm-v26-manifest])
- TrackEval [`12c8791`][te]: [hota.py][te-hota], [clear.py][te-clear], [identity.py][te-id], [mot_challenge_2d_box.py][te-mot-pre]; fork PyPI [kovalp/TrackEval][te-fork]
- Roboflow `trackers` [`84569dc`][rf]: [botsort/tracker.py][rf-botsort], [eval/mot_classes.py][rf-classes], [datasets/manifest.py][rf-manifest], [methodology.md][rf-method], [LICENSE][rf-license]
- py-motmetrics [`e5ae766`][pmm]: [Readme][pmm-readme]
- VisDrone2018-MOT-toolkit [`2dc45b4`][vdt]: [dropObjects.m][vdt-drop], [evaluateTrackA.m][vdt-evalA]; [VisDrone-Dataset][visdrone-ds]
- UCMCTrack [`7851a49`][ucmc-repo] ([cam_para][ucmc-cam]); BoT-SORT oficial [`2519854`][botsort-repo]
- Ultralytics 8.4.192: [bytetrack.yaml][ul-bt], [botsort.yaml][ul-bs], [gmc.py][ul-gmc] (verificado en el paquete instalado)
- Herramientas de anotación: [CVAT – formato MOT][cvat-mot], [Label Studio – video tracking][ls-video]
- Licencia: [GNU AGPL-3.0][agpl]
- Contexto interno (no fuente primaria): `README.md`, `VD - Trackers To Real Implementation/README.md` y `*.py`, `Historial de desarrollo de trackers/comparativa_v13/resumen_metricas.csv`, `comparativa_trackers_v13.ipynb`, la propuesta `README_UAV_BoxMOT_Propuesta.md`, la conversación previa y el doc "Benchmarking MOT trackers × detectors…".

<!-- Definiciones de enlaces -->
[hota]: https://arxiv.org/abs/2009.07736
[clear]: https://doi.org/10.1155/2008/246309
[ristani]: https://arxiv.org/abs/1609.01775
[mot16]: https://arxiv.org/abs/1603.00831
[motchallenge]: https://arxiv.org/abs/2010.07548
[visdrone-paper]: https://arxiv.org/abs/2001.06303
[uavdt]: https://arxiv.org/abs/1804.00518
[botsort-paper]: https://arxiv.org/abs/2206.14651
[ucmc-paper]: https://arxiv.org/abs/2312.08952
[uavmot]: https://openaccess.thecvf.com/content/CVPR2022/papers/Liu_Multi-Object_Tracking_Meets_Moving_UAV_CVPR_2022_paper.pdf
[mmot]: https://arxiv.org/abs/2510.12565
[dynuav]: https://arxiv.org/abs/2603.05970
[kittimod]: https://arxiv.org/abs/1709.04821
[tem]: https://arxiv.org/abs/2212.08536
[smot4sb]: https://arxiv.org/abs/2507.12832
[bm]: https://github.com/mikel-brostrom/boxmot/tree/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7
[bm-v26]: https://github.com/mikel-brostrom/boxmot/releases/tag/v26.0.0
[bm-v26-manifest]: https://github.com/mikel-brostrom/boxmot/blob/v26.0.0/boxmot/trackers/common/manifest.py
[bm-manifest]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/manifest.py#L22-L76
[bm-registry]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/registry.py#L15
[bm-specs]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/specs.py#L177-L222
[bm-factory]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/factory.py#L235
[bm-base]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/base.py#L60-L85
[bm-base-update]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/base.py#L351-L471
[bm-base-arrays]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/base.py#L586-L735
[bm-base-track]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/base.py#L877-L886
[bm-base-reset]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/base.py#L920-L948
[bm-boxbase]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/box/base.py#L71-L85
[bm-layout]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/detections/layout.py#L55
[bm-input]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/input.py#L78-L93
[bm-input-pack]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/input.py#L186-L240
[bm-cmc-reg]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/motion/cmc/registry.py#L37-L42
[bm-cmc-base]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/motion/cmc/base.py
[bm-prepared]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/common/motion/cmc/prepared.py#L12-L30
[bm-mm]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/engine/eval/motmetrics.py#L1-L30
[bm-mm-cols]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/engine/eval/motmetrics.py#L31
[bm-mm-aabb]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/engine/eval/motmetrics.py#L431-L506
[bm-te-ref]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/engine/eval/trackeval_reference.py#L13-L38
[bm-eval-cmc]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/engine/eval/cmc.py
[bm-bt-cfg]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/bytetrack/config.py#L39-L54
[bm-bs-cfg]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/trackers/botsort/config.py#L48-L70
[bm-bt-yaml]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/configs/trackers/bytetrack.yaml
[bm-visdrone]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/configs/datasets/visdrone.yaml
[bm-cfg-ds]: https://github.com/mikel-brostrom/boxmot/tree/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/boxmot/configs/datasets
[bm-pyproject]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/pyproject.toml#L124-L129
[bm-agents]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/AGENTS.md
[bm-doc-add]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/docs/contributing/add-tracker.md
[bm-doc-eval]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/docs/guides/evaluation.md#L128
[bm-doc-evalmode]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/docs/modes/eval.md
[bm-doc-modes]: https://github.com/mikel-brostrom/boxmot/tree/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/docs/modes
[bm-doc-mat]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/docs/modes/materialize.md#L365-L382
[bm-doc-tune]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/docs/modes/tune.md
[bm-doc-ds]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/docs/config/datasets.md#L96-L130
[bm-doc-det]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/docs/config/detectors.md
[bm-doc-exp]: https://github.com/mikel-brostrom/boxmot/blob/fab3fe0383e62993b4e15148c09d4ea0ccf54bf7/docs/config/experiments.md
[te]: https://github.com/JonathonLuiten/TrackEval/tree/12c8791b303e0a0b50f753af204249e622d0281a
[te-hota]: https://github.com/JonathonLuiten/TrackEval/blob/12c8791b303e0a0b50f753af204249e622d0281a/trackeval/metrics/hota.py
[te-clear]: https://github.com/JonathonLuiten/TrackEval/blob/12c8791b303e0a0b50f753af204249e622d0281a/trackeval/metrics/clear.py
[te-id]: https://github.com/JonathonLuiten/TrackEval/blob/12c8791b303e0a0b50f753af204249e622d0281a/trackeval/metrics/identity.py
[te-mot-pre]: https://github.com/JonathonLuiten/TrackEval/blob/12c8791b303e0a0b50f753af204249e622d0281a/trackeval/datasets/mot_challenge_2d_box.py#L320-L400
[te-mot-71]: https://github.com/JonathonLuiten/TrackEval/blob/12c8791b303e0a0b50f753af204249e622d0281a/trackeval/datasets/mot_challenge_2d_box.py#L71
[te-fork]: https://github.com/kovalp/TrackEval
[rf]: https://github.com/roboflow/trackers/tree/84569dcf06bdf9ecc2266ea14add22f4bd337280
[rf-core]: https://github.com/roboflow/trackers/tree/84569dcf06bdf9ecc2266ea14add22f4bd337280/src/trackers/core
[rf-botsort]: https://github.com/roboflow/trackers/blob/84569dcf06bdf9ecc2266ea14add22f4bd337280/src/trackers/core/botsort/tracker.py#L232-L260
[rf-classes]: https://github.com/roboflow/trackers/blob/84569dcf06bdf9ecc2266ea14add22f4bd337280/src/trackers/eval/mot_classes.py
[rf-manifest]: https://github.com/roboflow/trackers/blob/84569dcf06bdf9ecc2266ea14add22f4bd337280/src/trackers/datasets/manifest.py#L25-L26
[rf-method]: https://github.com/roboflow/trackers/blob/84569dcf06bdf9ecc2266ea14add22f4bd337280/docs/evaluations/methodology.md
[rf-license]: https://github.com/roboflow/trackers/blob/84569dcf06bdf9ecc2266ea14add22f4bd337280/LICENSE
[pmm]: https://github.com/cheind/py-motmetrics/tree/e5ae766fab03b804f485d7c95ee1809b94d32dcc
[pmm-readme]: https://github.com/cheind/py-motmetrics/blob/e5ae766fab03b804f485d7c95ee1809b94d32dcc/Readme.md
[vdt]: https://github.com/VisDrone/VisDrone2018-MOT-toolkit/tree/2dc45b4634f9374bf886382258dcfc43f8cb964b
[vdt-drop]: https://github.com/VisDrone/VisDrone2018-MOT-toolkit/blob/2dc45b4634f9374bf886382258dcfc43f8cb964b/eval/dropObjects.m#L31
[vdt-evalA]: https://github.com/VisDrone/VisDrone2018-MOT-toolkit/blob/2dc45b4634f9374bf886382258dcfc43f8cb964b/evaluateTrackA.m#L5-L7
[visdrone-ds]: https://github.com/VisDrone/VisDrone-Dataset
[ucmc-repo]: https://github.com/corfyi/UCMCTrack/tree/7851a49078a6666725d1bda670f0046fcc0d715f
[ucmc-cam]: https://github.com/corfyi/UCMCTrack/tree/7851a49078a6666725d1bda670f0046fcc0d715f/cam_para
[botsort-repo]: https://github.com/NirAharon/BoT-SORT/tree/251985436d6712aaf682aaaf5f71edb4987224bd
[ul-bt]: https://github.com/ultralytics/ultralytics/blob/v8.4.192/ultralytics/cfg/trackers/bytetrack.yaml
[ul-bs]: https://github.com/ultralytics/ultralytics/blob/v8.4.192/ultralytics/cfg/trackers/botsort.yaml
[ul-gmc]: https://github.com/ultralytics/ultralytics/blob/v8.4.192/ultralytics/trackers/utils/gmc.py
[cvat-mot]: https://docs.cvat.ai/docs/dataset_management/formats/format-mot/
[ls-video]: https://labelstud.io/templates/video_object_detector
[agpl]: https://www.gnu.org/licenses/agpl-3.0.html
