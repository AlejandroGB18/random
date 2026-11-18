# Investigación: ¿existe un ground truth público que sirva de base para nuestro benchmark?

> **Fecha:** 2026-11-18 · **Pregunta:** ¿hay un GT público, accesible y utilizable para evaluar los 4 trackers (propio, UCMC, ByteTrack, BoT-SORT) y el decisor `en_movimiento` en video de dron con cámara en movimiento, para no anotar VIGIA desde cero?
>
> **Contexto que no se repite aquí:** [`investigacion-benchmark-boxmot.md`](investigacion-benchmark-boxmot.md) (formatos VisDrone/UAVDT/MOT16, reglas de ignorados, métricas, plan y costo de anotar VIGIA) y [`../benchmark/README.md`](../benchmark/README.md) + [`../benchmark/DECISIONS.md`](../benchmark/DECISIONS.md) (formato `gt.txt` + `moving.csv`, criterio D12 para `moving`).
>
> Convención: **[V]** = verificado en la fuente primaria citada (página oficial, arXiv, repo, cabecera HTTP); **NO VERIFICADO** = no se pudo comprobar; **[Propuesta]** = recomendación propia; **[Derivado]** = cálculo propio a partir de cifras verificadas. No se descargó ningún dataset. Solo se hizo `HEAD` a algunos enlaces.

---

## 0. Respuesta corta

1. **Ningún dataset público cubre todo a la vez:** tracking de vehículos con ID + cámara de dron en movimiento + etiqueta "se mueve de verdad / quieto". El único que anota explícitamente "solo objetos en movimiento" en video UAV es **MOR-UAV**, pero **no está publicado** (el repo oficial no tiene enlace de descarga) y sus anotaciones son cajas por frame **sin ID** [V] ([repo][moruav-repo], [arXiv 2008.01699][moruav]).
2. **Para tracking con GT real y cámara móvil, la base es UAVDT** (solo vehículos: car/truck/bus, IDs, ignorados, 1080×540 a 30 fps, descarga directa en Google Drive, "research purpose only") y, como segunda fuente, **VisDrone2019-MOT** (val y test-dev tienen GT pública). Los dos tienen ego-movimiento **moderado** (hover o crucero lento según los autores de DynUAV). Para ego-movimiento **fuerte**, el candidato es **DynUAV** (1080p, DJI Mini 4 Pro, órbitas, rotación, zoom), pero solo se descarga por Baidu y **no declara licencia**.
3. **Para el decisor `moving`, la mejor fuente con verdad física es sintética: SynDrone** (CARLA, 1920×1080, 25 fps, dron que vuela, alturas 20/50/80 m con pitch 30/60/90°, IDs únicos, cajas 3D y extrínsecos de cámara). Da `moving` exacto y además la pose para probar la calibración de UCMC. Como fuente **real** de `moving`, **pNEUMA Vision** (Atenas, dron en hover, CC BY 4.0, Zenodo directo) enlaza los frames con trayectorias que tienen **velocidad en km/h**. Su cámara está estabilizada, así que sirve para el decisor solo si se le **añade ego-movimiento sintético** [Propuesta].
4. **Telemetría real con video** solo la ofrece **AU-AIR** (altitud, GPS, roll/pitch/yaw, velocidad cada 20 ms), pero sin IDs de tracking y con sus **enlaces oficiales rotos (404)** hoy.
5. **Ninguno reemplaza a VIGIA** (4K a 59.94 fps oblicuo, tráfico urbano, detector COCO/DOTA sin ajuste). Lo público **reduce** el GT propio, no lo elimina: bastan **2–3 clips VIGIA cortos** como prueba de dominio (≈ 6–10 h-persona en vez de 10–20 h) si la comparación de trackers se apoya en UAVDT/VisDrone/DynUAV y el decisor en SynDrone + pNEUMA.

**Combinación mínima recomendada (por prioridad):** UAVDT → SynDrone → (DynUAV si Baidu funciona) → 2–3 clips VIGIA con GT propio → pNEUMA Vision como control de cámara fija. Detalle y costos en §5.

---

## 1. Tabla comparativa

Leyenda de "Sirve como": **P** = GT principal de tracking · **M** = GT para el decisor `moving` · **A** = ajuste de hiperparámetros · **F** = control de cámara fija · **U** = validar la calibración de UCMC (hfov/pitch/altura) · **✗** = no sirve.

| Dataset | (a) IDs vehículos | (b) Cámara | (c) `moving` | (d) Telemetría | (e) Acceso / licencia / tamaño | (f) Vista, res., fps | Sirve como |
|---|---|---|---|---|---|---|---|
| **UAVDT** | Sí (car/truck/bus) | Móvil moderada | Derivar con H | Solo atributo altitud/vista por secuencia | Google Drive directo; "research purpose only"; tamaño NO VERIFICADO | Frontal/lateral/cenital; 1080×540; 30 fps | **P**, A, M (derivado) |
| **VisDrone2019-MOT** | Sí (car/van/truck/bus + otras) | Móvil moderada | Derivar con H | No | GDrive/Baidu directo; licencia no declarada; 7.5 + 1.5 + 2.1 GB | Mixta; ~2K y menor (NO VERIFICADO por secuencia) | **P** (2ª), A |
| **DynUAV** | Sí (car, truck, maquinaria…) | **Fuerte** (órbitas, rotación, zoom) | Derivar con H (difícil) | No | Solo Baidu; **sin licencia** | 1080p; fps NO VERIFICADO; 80–120 m | P (estrés de ego), A |
| **MMOT** | Sí, OBB (car/van/truck/bus) | Jitter y rotación; 14.1 px/frame | Derivar con H | No | HF directo, 120 GB; CC BY-NC-ND 4.0 | Cenital; 1200×900, 8 bandas; ~5 fps [Derivado] | Secundario (dominio lejano) |
| **MDMT** | Sí (car, persona, bici) | Multi-dron; movimiento NO VERIFICADO | Derivar con H | No | Baidu, 25.8 GB; licencia no declarada | NO VERIFICADO | Secundario |
| **VETRA** | Sí (8 clases de vehículo) | **Fuerte** (avión/helicóptero) | Derivar con H | NO VERIFICADO | Descarga directa DLR 38.9 GB; imágenes CC BY-NC-ND 3.0 | Cenital; baja tasa de frames | Estrés de ego (no dron) |
| **MOR-UAV** | **No** (solo cajas de móviles) | Móvil | **Sí, implícito** (solo se anota lo que se mueve) | No | **No publicado** | 30 videos; res. NO VERIFICADO | ✗ hoy (contactar autores) |
| **AU-AIR** | **No** (cajas por frame) | Móvil, 45–90° | No | **Sí** (GPS, alt., RPY, vel.) | **Enlaces oficiales 404** | 1920×1080; 30 fps (anotado a 5 fps) | U (si se recupera), ✗ tracking |
| **SynDrone** (sintético) | **Sí, IDs únicos** | **Vuelo** (~20 m/s) | **Exacto** (cajas 3D en el tiempo) | **Extrínsecos**; intrínsecos NO VERIFICADO | Descarga directa por ciudad (4.98 GB c/u); repo Apache-2.0, licencia de datos NO VERIFICADO | 20/50/80 m con pitch 30/60/90°; 1920×1080; 25 fps | **M**, **U**, P sintético |
| **pNEUMA Vision** | Sí (ID de pNEUMA) | Hover, frames estabilizados | **Sí** (velocidad km/h de la trayectoria) | Posición georreferenciada | Zenodo directo, 104.6 GB en 19 zips; CC BY 4.0 | Cenital; 1 de cada 10 frames; res. NO VERIFICADO | **M** (con ego sintético), **F** |
| **highD / inD / rounD / exiD / uniD** | Trayectorias, **sin video publicado** | Hover + estabilización | Sí (velocidades) | Georreferencia | Formulario manual; solo no comercial; sin redistribución | 4K, 25 fps (highD) | ✗ (sin video); solo calibrar umbrales |
| **CitySim** | Trayectorias + cajas rotadas | NO VERIFICADO | Sí (trayectorias) | Mapas 3D | Por solicitud; no comercial | 30 fps | Posible M/F; NO VERIFICADO video |
| **DRIFT** | Pseudo-GT (YOLOv11 + **ByteTrack**) | Estabilizada, 250 m | Derivable | No | HF, CC BY 4.0, **solo CSV** | 4K, ~30 fps | ✗ (sesgo a ByteTrack, sin video) |
| **Stanford Drone** | Sí (sobre todo peatones/bicis) | Casi fija (NO VERIFICADO) | Derivable | No | 69 GB; licencia NO VERIFICADO | Cenital campus | F (débil) |
| **Okutama-Action** | Solo personas | Móvil | No | No | okutama-action.org | 4K, 30 fps | ✗ |
| **DroneVehicle** | No (detección RGB-IR) | — | No | No | GitHub VisDrone | Imágenes | ✗ (solo detector) |
| **CARPK** | No (conteo) | — | No | No | — | Imágenes de estacionamiento | ✗ (solo detector) |
| **VIVID (DARPA)** | Sí (vehículos) | Móvil | Derivable | NO VERIFICADO | Acceso actual NO VERIFICADO | 640×480 | ✗ (obsoleto) |
| **WPAFB 2009 (WAMI)** | Sí (vehículos, manual) | Avión, 6 cámaras | Derivable | NO VERIFICADO | AFRL SDMS, acceso NO VERIFICADO | 1.25 Hz | ✗ (tasa muy baja) |
| **UAV3D** (sintético) | IDs | Formación de drones | Exacto | Intr. + extr. | Web del proyecto | 800×450; **0.5 Hz** | ✗ (0.5 Hz) |
| **TartanAir** (sintético) | No (sin anotación de objetos) | Móvil | — | Pose | — | — | ✗ |
| **SMOT4SB** | Aves | Móvil | — | — | Challenge MVA 2025 | — | ✗ |

---

## 2. Fichas por candidato

### 2.1 UAVDT — **la mejor base real para tracking de vehículos**
- **(a)** 100 secuencias, ~80 k frames, >2 700 vehículos, 0.84 M cajas, clases car/truck/bus. Anotado por >10 expertos con VATIC [V] ([arXiv 1804.00518][uavdt]).
- **(b)** El paper cita "camera motion" como desafío, pero **no es un atributo anotado**; la vista puede cambiar dentro de una secuencia [V] ([página oficial][uavdt-site]). Los autores de DynUAV lo describen como "hovering or slow cruising" [V] ([arXiv 2603.05970][dynuav]).
- **(c)** No hay etiqueta `moving`. Se deriva con homografía (§3).
- **(d)** Solo atributos discretos por secuencia: altitud (baja 10–30 m, media 30–70 m, alta >70 m) y vista (frontal/lateral/cenital) [V] ([arXiv 1804.00518][uavdt]). Sin hfov ni pitch numérico.
- **(e)** Dataset, toolkit DET/MOT y atributos en **Google Drive, sin registro**. Licencia: "This dataset is for research purpose only" [V] ([página oficial][uavdt-site]). Tamaño NO VERIFICADO.
- **(f)** 1080×540, 30 fps. Vistas frontal, lateral y cenital, cercanas a VIGIA oblicuo y cenital. Resolución mucho menor que el 4K de VIGIA.
- **(g)** Regiones ignoradas donde los vehículos son demasiado pequeños; se descartan objetos con >50 % fuera de cuadro [V] ([página][uavdt-site]). Split: 30 train / 70 test (20 de ellas de DET/MOT).
- **Sirve como:** **P** (GT principal) y **A** (ajuste con train). **M** solo derivado.

### 2.2 VisDrone2019-MOT
- **(a)** 56/7/17/16 secuencias train/val/test-dev/test-challenge; clases car, van, truck, bus y otras (formato y regla de ignorados en el doc previo).
- **(e)** Descarga directa (Google Drive y BaiduYun) sin registro: train 7.53 GB, val 1.48 GB, test-dev 2.14 GB, test-challenge 2.70 GB. **test-challenge sin GT**; test-dev con GT y "Researchers can use test-dev to publish papers". La página **no declara licencia** [V] ([repo VisDrone-Dataset][visdrone-ds]).
- **(b)/(d)** Igual que UAVDT: ego-movimiento moderado según DynUAV [V] ([arXiv 2603.05970][dynuav]). Sin telemetría (NO VERIFICADO que no exista; la página no la ofrece).
- **(g)** Mezcla peatones y vehículos; clases 0/11 ignoradas por área.
- **Sirve como:** P secundario y A. Licencia ambigua → uso interno de investigación, sin redistribuir.

### 2.3 DynUAV (CVPR 2026) — **el que tiene ego-movimiento fuerte**
- **(a)** 42 secuencias, 37 893 frames, >1.7 M cajas, 8 categorías (car, truck, excavator, crane, bulldozer, cycle, cycler, person). Anotado en **CVAT** con ID único y en tres etapas (inicial, revisión, refinamiento). Se anota desde que el objeto es "fully visible" y durante la oclusión si su posición se puede inferir [V] ([arXiv 2603.05970 HTML][dynuav-html]).
- **(b)** 80–120 m. Incluye hover, crucero, velocidad variable, rotación, zoom, órbitas y acercamiento/alejamiento [V] ([HTML][dynuav-html]).
- **(f)** 1080p con DJI Mini 4 Pro. fps NO VERIFICADO. Secuencias de 1 076 a 2 655 frames. Split 30/5/7 [V].
- **(e)** Solo **Baidu Netdisk** (con código); el layout de test incluye `gt/`. **No hay licencia del dataset**: el repo no tiene licencia en la API de GitHub y el README dice MIT solo para el código [V] ([repo][dynuav-repo]).
- **Sirve como:** P para el estrato "rápido", A. **Riesgo:** sin licencia explícita (por defecto, todos los derechos reservados) → pedir permiso por escrito antes de publicar resultados. Baidu puede pedir cuenta (NO VERIFICADO).

### 2.4 MMOT (NeurIPS 2025)
- 125 secuencias, ~13.8 k frames, 488.8 k OBB, 8 clases (Pedestrian, Car, Van, Truck, Bus, Tricycle, Bike, Awning-bike), IDs persistentes, cámara cenital a 80–200 m con jitter y rotación; desplazamiento medio de la plataforma ≈ 14.1 px/frame frente a 4.3 px de los objetos [V] ([arXiv 2510.12565][mmot-html]).
- 1200×900 y **8 bandas espectrales en `.npy`**. ~2 767 s para 13.8 k frames → **≈ 5 fps** [Derivado].
- Hugging Face, **120 GB**, **CC BY-NC-ND 4.0**, "academic research only" [V] ([HF][mmot-hf], [repo][mmot-repo]).
- **Sirve como:** secundario. Hay que sintetizar RGB desde las bandas (dominio distinto al del detector COCO), la tasa es baja y "ND" limita distribuir derivados.

### 2.5 MDMT
- 88 secuencias, 39 678 frames, 11 454 IDs (personas, bicicletas, autos), 2.2 M cajas; multi-dron con asociación entre drones. Solo Baidu (25.8 GB); sin licencia declarada [V] ([repo][mdmt]). Movimiento de cámara, resolución y fps NO VERIFICADO.
- **Sirve como:** secundario; aporta poco frente a UAVDT para nuestra pregunta.

### 2.6 VETRA (ECCV 2024, DLR)
- 52 secuencias desde **avión y helicóptero** (cámaras 3K/4K del DLR), 8 clases de vehículo, polígonos/OBB/HBB, **baja tasa de frames**, movimiento de cámara generalmente alto. Descarga directa de 38.9 GB, sin registro; imágenes "DLR (CC BY-NC-ND 3.0)" [V] ([DLR][vetra]). fps exacto y formato de IDs NO VERIFICADO.
- **Sirve como:** prueba de estrés de ego-movimiento, pero no es dron ni urbano a baja altura.

### 2.7 MOR-UAV — el único con "solo lo que se mueve", pero no está disponible
- 30 videos, 10 948 frames, 89 783 instancias **en movimiento** (car, heavy vehicle), cajas alineadas, con movimiento de cámara y cambios de altura [V] ([arXiv 2008.01699][moruav]).
- Repo oficial: formato CSV `path,x1,y1,x2,y2,class` (**sin ID**) y **sin enlace de descarga**; sección del paper "TODO"; licencia Apache-2.0 del código [V] ([repo][moruav-repo]).
- **Sirve como:** ✗ hoy. Si los autores lo compartieran, sería un buen GT del decisor a nivel de frame (no de track). Escribir a los autores [Propuesta].

### 2.8 AU-AIR — telemetría real, pero sin IDs y con enlaces caídos
- 32 823 frames anotados (8 clases: person, car, van, truck, motorbike, bike, bus, trailer), anotados con MTurk (3 trabajadores por frame). Video 1920×1080 a 30 fps, **anotado a 5 fps**, altitud 10–30 m, cámara 45–90°. Telemetría cada 20 ms: GPS, altitud (mm), roll/pitch/yaw (rad) y Vx/Vy/Vz (m/s) [V] ([arXiv 2001.11737][auair]).
- **No hay IDs de tracking** en el paper [V] (cajas por frame). Intrínsecos NO VERIFICADO.
- **Acceso:** `bozcani.github.io/auairdataset` redirige a `odtum.de/auairdataset` → **404**; `github.com/bozcani/auairdataset` → **404** (comprobado 2026-11-18). Los mirrors de terceros (Roboflow, etc.) no son fuente oficial.
- **Sirve como:** ✗ para tracking. Si se recupera, U (probar el pitch/altura que estima UCMC contra la telemetría), anotando IDs a mano.

### 2.9 SynDrone (sintético, CARLA) — **la mejor fuente para `moving` y para UCMC**
- CARLA 0.9.12 modificado; alturas 20/50/80 m con pitch 30/60/90°; 1920×1080 a 25 fps; 72 k frames (8 ciudades, 3 000 frames por sensor y trayectoria); vehículos y peatones dinámicos; el dron recorre 2–3 km a ~20 m/s; cajas 3D de todos los actores con **identificador único** por actor; segmentación de 28 clases [V] ([arXiv 2308.10491][syndrone]).
- Los zips de color incluyen "RGB images, Semantic Segmentation labels, Camera Extrinsics, and Bounding Box ground truth" [V] ([repo][syndrone-repo]). `HEAD` a `Town01_Opt_120_color.zip` → 200, **4.98 GB**, sin registro [V]. `syndrone.zip` completo no respondió al `HEAD` (NO VERIFICADO).
- Intrínsecos (FOV de CARLA) NO VERIFICADO en la documentación; probablemente se deducen de la configuración de generación. Licencia del **código** Apache-2.0; la de los **datos** NO VERIFICADO. El README recomienda su sucesor FlyAwareV2 [V].
- **Ventaja única:** la velocidad de cada actor en el mundo es exacta → `moving` perfecto, incluido el caso de autos estacionados con paralaje. La pose de cámara permite medir el error de la calibración automática de UCMC.
- **Límites:** brecha sim→real (texturas, densidad); el dron a 20 m/s es más rápido que VIGIA; las cajas 2D hay que **proyectarlas** desde las 3D con la cámara (el paper menciona cajas 2D solo de objetos móviles; detalle NO VERIFICADO).
- **Sirve como:** **M**, **U**, P sintético.

### 2.10 pNEUMA Vision — fuente **real** de `moving`, con cámara fija
- Extensión de pNEUMA (10 drones en hover sobre Atenas): frames de dron con posiciones de vehículos en coordenadas de imagen, azimut, y **ID, tipo y timestamp heredados de pNEUMA**; anotado **1 de cada 10 frames**; los frames están **estabilizados** (hay frames ocluidos por la estabilización con viento) [V] ([toolbox][pneuma-tb]). El toolbox genera cajas rotadas o alineadas.
- Zenodo: **104.6 GB** en 19 zips de 4.7–6.7 GB (se puede bajar uno solo), **CC BY 4.0**, acceso abierto [V] ([Zenodo 7426506][pneuma-vision]). Paper: Kim et al., TR-C 147 (2023) 103966 [V] ([Infoscience][pneuma-paper]).
- Trayectorias pNEUMA: velocidad en km/h, muestreo 0.04 s (25 fps; un dron a 23.97 fps) [V] ([descargas][pneuma-dl]). Licencia del pNEUMA base: la web dice CC BY 4.0 en la FAQ y CC BY-NC 4.0 en el pie → **ambigua** [V] ([descargas][pneuma-dl]); Zenodo 10491408 aparece como CC BY-NC 4.0 en búsqueda (NO VERIFICADO en la ficha).
- El paper separa errores de trayectoria "estacionarios" y "no estacionarios" [V] ([Infoscience][pneuma-paper]) → el ruido de velocidad no es despreciable.
- **Sirve como:** **F** (control de cámara fija: los 4 trackers deberían empatar) y **M** real si se le aplica **ego-movimiento sintético** (warps de homografía aleatorios y suaves sobre frames estabilizados) [Propuesta]. Ese ego sintético es plano: **no genera paralaje real**, así que no prueba las falsas alarmas por paralaje.

### 2.11 highD / inD / rounD / exiD / uniD (levelXData)
- highD: DJI Phantom 4 Pro Plus en **hover**, 4096×2160 a 25 fps, ~10×10 cm por píxel, tramo de ~420 m; videos **estabilizados** al primer frame con OpenCV; vehículos extraídos **automáticamente** (U-Net + tracker + suavizado RTS) [V] ([arXiv 1810.05642][highd]). Es decir, **pseudo-GT**, no anotación humana.
- Las páginas de highD/inD/rounD listan trayectorias georreferenciadas con velocidad y aceleración, mapas y **una foto aérea por lugar sin vehículos**; **no mencionan video** [V] ([highD][highd-site], [inD][ind-site], [rounD][round-site]). Acceso por formulario revisado a mano, solo no comercial, sin redistribución [V].
- **Sirve como:** ✗ para evaluar trackers (sin video). Como mucho, calibrar umbrales de velocidad de "quieto vs. móvil" (p. ej. distribución de velocidades en atasco) [Propuesta].

### 2.12 CitySim
- Trayectorias de 1 140 min de video de dron a 30 fps en 12 lugares, con cajas rotadas, mapas 3D y tiempos de semáforo [V] ([repo][citysim], [arXiv 2208.11036][citysim-paper]). Acceso completo por solicitud, no comercial (según el resultado de búsqueda; el README remite a "contact the relevant team" para algunos sitios) — si se distribuye video y si el dron está en hover: **NO VERIFICADO**.
- **Sirve como:** posible F/M si se obtiene video; no prioritario.

### 2.13 DRIFT
- 81 699 trayectorias de 4K a 30 fps, 250 m, estabilizadas; **generadas con YOLOv11m + ByteTrack (OBB)** [V] ([repo][drift-repo]). En Hugging Face solo hay CSV (CC BY 4.0), sin frames [V] ([HF][drift-hf]).
- **✗:** es salida de uno de los trackers que comparamos → favorecería a ByteTrack. Además no tiene video.

### 2.14 Descartados (por qué)
- **Stanford Drone Dataset:** campus, sobre todo peatones y ciclistas (autos ≤ ~30 % solo en "nexus"); 69 GB [V] ([página][sdd], leída con `curl`; el certificado TLS fue rechazado por WebFetch). Formato de anotación, licencia y estabilización NO VERIFICADO. Control de cámara fija, débil para vehículos.
- **Okutama-Action:** solo humanos; 4K a 30 fps, anotado a 10 fps e interpolado [V] ([arXiv 1706.03038][okutama]).
- **DroneVehicle:** 28 439 pares de imágenes RGB-IR para **detección** [V] ([arXiv 2003.02437][dronevehicle]). Útil solo para ajustar el detector.
- **CARPK:** conteo de autos en estacionamientos (~90 k autos), sin tracks [V] ([arXiv 1707.05972][carpk]).
- **VIVID (DARPA, 2005):** 9 secuencias, solo vehículos, 640×480; citado como "outdated" [V vía cita en [arXiv 1708.05869][sim4cv]]. Acceso oficial actual NO VERIFICADO.
- **WPAFB 2009:** WAMI desde avión, 1 025 frames a **1.25 Hz**, vehículos anotados a mano [V vía [arXiv 1911.01727][wami]]; acceso en AFRL SDMS NO VERIFICADO.
- **UAV3D:** CARLA + AirSim con intrínsecos/extrínsecos, pero 800×450 a **0.5 Hz** [V] ([arXiv 2410.11125][uav3d]) → inservible para tracking.
- **TartanAir:** pose, profundidad, flujo y segmentación, sin anotación de objetos para tracking [V] ([arXiv 2003.14338][tartanair]).
- **SMOT4SB:** 211 secuencias de **aves** pequeñas con cámara móvil [V] ([arXiv 2507.12832][smot4sb]).
- **KITTI-MOD / MOT16 `static person`:** precedentes de etiqueta estático/dinámico, no dron (ver doc previo).

---

## 3. Derivar `moving` en un dataset público: ¿qué tan fiable es?

| Fuente | Cómo se obtiene | Fiabilidad | Errores típicos |
|---|---|---|---|
| **SynDrone** | Desplazamiento 3D del actor en el mundo entre frames (cajas 3D por ID) | **Exacta** (verdad del simulador) | Ninguno de etiqueta; solo la brecha sim→real |
| **pNEUMA Vision** | Velocidad km/h de la trayectoria pNEUMA (25 Hz) enlazada por ID | **Alta** con umbral + histéresis (p. ej. > 5 km/h sostenido 1 s, < 2 km/h quieto, resto dudoso) [Propuesta] | Ruido de trayectoria (el paper lo separa en estacionario y no estacionario); la cámara es fija, así que el problema de ego-movimiento no existe |
| **highD/inD/…** | Velocidad m/s del archivo | Alta, pero es pseudo-GT automático y **no hay video** | — |
| **UAVDT / VisDrone / DynUAV / MMOT** | Solo cajas en píxeles → estabilizar con homografía por frame y aplicar el criterio D12 de [`gt.py`](../benchmark/gt.py) (ventana de 1 s, d ≥ 0.30·L móvil, d ≤ 0.10·L quieto, resto dudoso, revisión humana) | **Media**: buena en cenital y escenas planas; **baja** donde falla la H (ego rápido, rotación, zoom en DynUAV) | (1) **Paralaje**: en oblicuas, objetos elevados o cercanos parecen moverse tras estabilizar → justo la falsa alarma que queremos medir, así que el GT derivado **hereda** el error; (2) jitter de las cajas del GT (UAVDT/VisDrone anotan con interpolación) → usar mediana temporal como ya hace `gt.py`; (3) secuencias con poca textura de fondo → H no fiable → marcar `dudoso` |

**Reglas [Propuesta]:**
- La H para derivar `moving` del GT debe ser **independiente** de la de los trackers (ya fijado en D10: SIFT+MAGSAC), y los frames con H no fiable deben quedar `dudoso`.
- En datasets públicos, revisar a mano **solo** la lista `moving_revisar.csv` (dudosos y cambios de estado), no todo.
- Para medir el decisor en el caso de **paralaje real**, la única fuente con verdad física es SynDrone (o VIGIA anotado). pNEUMA con ego sintético no lo contiene.

---

## 4. Riesgos

| Riesgo | Detalle | Mitigación [Propuesta] |
|---|---|---|
| **Clases** | VisDrone separa car/van/truck/bus (+ otras); UAVDT car/truck/bus; MMOT incluye tricycle/awning-bike; DynUAV incluye maquinaria; nuestro detector es COCO (car/truck/bus/motorcycle) o DOTA (small/large vehicle) | Fusionar todo en `vehicle` (D7); tratar como **ignorado** lo que nuestro detector no puede ver por diseño (tricycle, maquinaria) en vez de FN |
| **Regiones ignoradas** | VisDrone clases 0/11 y UAVDT zonas rosas → regla por área ≥ 50 % (D8) | El conversor debe mapearlas a clase 0 del harness; verificar con el toolkit oficial en una secuencia |
| **Anotación cada k frames / interpolación** | AU-AIR a 5 fps; pNEUMA Vision 1/10 frames; Okutama interpolado; UAVDT con VATIC (interpolado) | Evaluar solo en frames anotados (D9); no interpretar latencias del decisor por debajo de la resolución de anotación |
| **Pseudo-GT** | highD (U-Net+tracker), DRIFT (YOLO+ByteTrack) | No usar como GT de tracking; DRIFT favorece a ByteTrack |
| **Dominio** | UAVDT 1080×540 vs VIGIA 4K 59.94 fps; MMOT multiespectral; VETRA avión; SynDrone sintético | Reportar por dataset, nunca promediar con VIGIA; los rankings que coincidan entre dominios son los fiables |
| **fps** | VIGIA oblicuo a 59.94 fps vs 25–30 en públicos (≈ 5 fps en MMOT) | Estratificar por desplazamiento por frame (px), no por segundos; opcionalmente submuestrear VIGIA a 30 fps para comparar |
| **Detector COCO vs objetos diminutos** | En VisDrone/UAVDT/DynUAV a 80–120 m los autos tienen pocos px; YOLO11n COCO tendrá DetA muy bajo y el ranking de trackers queda dominado por FN | Para datasets públicos, usar **un detector entrenado en su train** (p. ej. YOLO11 en VisDrone/UAVDT train), igual para los 4 trackers; o evaluar con las **detecciones provistas** (VisDrone las trae) |
| **UCMC sin hfov** | Ningún dataset real accesible da hfov + pitch + altitud por frame (AU-AIR caído) | Validar la calibración de UCMC en SynDrone (pose conocida); en reales, reportar sensibilidad a hfov (A7) |
| **Licencias** | UAVDT "research only"; MMOT CC BY-NC-ND; highD/levelX no comercial; VisDrone, MDMT y DynUAV **sin licencia declarada**; pNEUMA ambigua (BY vs BY-NC) | Uso interno de investigación; no redistribuir datos ni GT convertido; si VIGIA es un producto comercial, **consultar** antes de usar cualquiera para ajustar un modelo que se despliegue; pedir permiso escrito a DynUAV/VisDrone |
| **Acceso** | Baidu (DynUAV, MDMT) puede requerir cuenta/cliente; AU-AIR caído; MOR-UAV no publicado | Priorizar los de descarga directa (UAVDT, VisDrone, SynDrone, pNEUMA Vision) |

---

## 5. Si ninguno basta para VIGIA: GT propio mínimo

**Conclusión:** ningún público tiene a la vez el dominio VIGIA (4K oblicuo a 59.94 fps, urbano, sin telemetría) y la etiqueta `moving`. Hace falta un GT propio, **más chico** que el plan previo.

**Herramientas [V]:**
- **CVAT**: el tracker SAM2 existe como *AI Agent* (CVAT Online/Enterprise ≥ 2.42.0) o como función Nuclio (solo Enterprise), pero **solo sobre polígonos y máscaras, no rectángulos** [V] ([docs SAM2][cvat-sam2]). Para cajas, la pestaña *Tracking* de AI Tools propaga rectángulos frame a frame con modelos como TransT [V] ([docs AI tools][cvat-ai]); qué ediciones lo incluyen: NO VERIFICADO. Export MOT 1.1 ya verificado en el doc previo.
- **Label Studio**: keyframes con interpolación lineal y sin export MOT nativo (doc previo).

**Pseudo-GT + revisión [Propuesta]:**
1. **Pre-anotar con algo ajeno a los 4 trackers**: un detector más fuerte y lento (p. ej. YOLO11x a 1920–2560 px o un modelo afinado en VisDrone) + un enlazador offline simple (IoU + interpolación) o el tracker de CVAT. **Nunca** con ByteTrack/BoT-SORT/UCMC/propio, ni con la H del Estabilizador: el GT terminaría pareciéndose al método evaluado.
2. **Revisión humana completa** de cada track (no solo muestreo), con foco en cambios de ID y en el inicio/fin de los tracks. Medir el sesgo: anotar **un** clip desde cero y comparar HOTA(pre-anotación revisada, desde cero); si difiere > 2–3 puntos, la pre-anotación está sesgando.
3. `moving` con `bench.py moving` (D12) + revisión solo de `moving_revisar.csv`.
4. **Recortes** espaciales (ROI) en clips densos en vez de anotarlos completos.

**Cuánto [Propuesta, supuestos del doc previo: ~2 h por clip de densidad media]:** con los públicos cubriendo la comparación general, VIGIA solo tiene que responder "¿el ranking se mantiene en nuestro dominio?" y "¿el decisor funciona en oblicuo 4K con paralaje real?":
- **vigia_4** (ego rápido, ~1 objeto) completo: ~1 h.
- **Tramo de 20–25 s de vigia_9 o vigia_10** (oblicuo, ~13–26 objetos, paralaje): ~2–3 h.
- **Tramo de vigia_1** (cenital suave, ~29 objetos), ROI: ~2 h.
- QA 20 %: ~1–2 h. **Total ≈ 6–8 h-persona** (frente a 10–20 h del plan con 4–5 clips). Cronometrar el primer clip y recalcular.

---

## 6. Recomendación final priorizada

| Prioridad | Qué | Para qué | Costo estimado [Propuesta] |
|---|---|---|---|
| **1** | **UAVDT** (Google Drive, sin registro) | GT principal de tracking de vehículos con cámara móvil; estratos por bin de ego (D11); train para el ajuste (A2) | Descarga + conversor UAVDT→`gt.txt` (ignorados → clase 0; car/truck/bus → `vehicle`) ≈ **1 día**; detector entrenado en UAVDT/VisDrone train ≈ 0.5–1 día de GPU |
| **2** | **SynDrone** (1–2 ciudades, ~5 GB c/u) | GT **exacto** de `moving` (incluido paralaje) y validación de la calibración de UCMC con pose conocida | Conversor (proyectar cajas 3D → 2D, IDs, pose → hfov/pitch/altura) ≈ **2–3 días** |
| **3** | **2–3 clips VIGIA** con GT propio (§5) | Validez en nuestro dominio: ranking de trackers y decisor en oblicuo 4K | **6–8 h-persona** |
| **4** | **VisDrone2019-MOT** val + test-dev | Segunda fuente real; comparabilidad con la literatura | Conversor ≈ 0.5 día (el formato es casi el de UAVDT) |
| **5** | **DynUAV** (si Baidu y permiso) | Estrato de ego-movimiento fuerte con GT humano | Acceso incierto; conversor ≈ 0.5 día; **pedir permiso de uso** |
| **6** | **pNEUMA Vision** (1 zip de ~5 GB) | Control de cámara fija (los 4 deberían empatar) + decisor real con ego sintético plano | ≈ 1–2 días (enlazar IDs con velocidades, warps sintéticos) |
| — | MMOT, MDMT, VETRA | Solo si sobra tiempo, como estrés | — |
| — | MOR-UAV, AU-AIR | Escribir a los autores (datos no publicados / enlaces caídos) | 0 h, solo un correo |

**Lectura de resultados:** el tracking se decide con UAVDT (+ VisDrone, + DynUAV) y se **confirma** en VIGIA; el decisor se mide con SynDrone (verdad física, paralaje incluido) y se **confirma** en VIGIA oblicuo. Si un ranking cambia entre datasets, vale el de VIGIA, pero con el IC ancho que dan pocos clips.

**Costo total aproximado:** ~5–7 días-persona de ingeniería (conversores y detector) + 6–8 h de anotación, frente a 10–20 h de anotación sin datasets públicos y **sin** poder medir el decisor con verdad física ni la calibración de UCMC.

---

## 7. Fuentes

[uavdt]: https://arxiv.org/abs/1804.00518
[uavdt-site]: https://sites.google.com/view/grli-uavdt/
[visdrone-ds]: https://github.com/VisDrone/VisDrone-Dataset
[dynuav]: https://arxiv.org/abs/2603.05970
[dynuav-html]: https://arxiv.org/html/2603.05970
[dynuav-repo]: https://github.com/kxzhang-lab/DynUAV
[mmot-html]: https://arxiv.org/html/2510.12565
[mmot-hf]: https://huggingface.co/datasets/Annzstbl/MMOT
[mmot-repo]: https://github.com/Annzstbl/MMOT
[mdmt]: https://github.com/VisDrone/Multi-Drone-Multi-Object-Detection-and-Tracking
[vetra]: https://www.dlr.de/en/eoc/vetra
[moruav]: https://arxiv.org/abs/2008.01699
[moruav-repo]: https://github.com/murari023/mor-uav
[auair]: https://arxiv.org/abs/2001.11737
[syndrone]: https://arxiv.org/abs/2308.10491
[syndrone-repo]: https://github.com/LTTM/Syndrone
[pneuma-vision]: https://zenodo.org/records/7426506
[pneuma-tb]: https://github.com/shgold/pNUEMA-Vision-toolbox
[pneuma-paper]: https://infoscience.epfl.ch/items/645868a4-a853-41c7-8078-fc9acfb04959
[pneuma-dl]: https://open-traffic.epfl.ch/index.php/downloads/
[highd]: https://arxiv.org/abs/1810.05642
[highd-site]: https://levelxdata.com/highd-dataset/
[ind-site]: https://levelxdata.com/ind-dataset/
[round-site]: https://levelxdata.com/rounD-dataset/
[citysim]: https://github.com/ozheng1993/UCF-SST-CitySim-Dataset
[citysim-paper]: https://arxiv.org/abs/2208.11036
[drift-repo]: https://github.com/AIxMobility/The-DRIFT
[drift-hf]: https://huggingface.co/datasets/Hj-Lee/The-DRIFT
[sdd]: https://cvgl.stanford.edu/projects/uav_data/
[okutama]: https://arxiv.org/abs/1706.03038
[dronevehicle]: https://arxiv.org/abs/2003.02437
[carpk]: https://arxiv.org/abs/1707.05972
[sim4cv]: https://arxiv.org/abs/1708.05869
[wami]: https://arxiv.org/abs/1911.01727
[uav3d]: https://arxiv.org/abs/2410.11125
[tartanair]: https://arxiv.org/abs/2003.14338
[smot4sb]: https://arxiv.org/abs/2507.12832
[cvat-sam2]: https://docs.cvat.ai/docs/annotation/auto-annotation/segment-anything-2-tracker/
[cvat-ai]: https://docs.cvat.ai/docs/annotation/auto-annotation/ai-tools/

- UAVDT — Du et al., ECCV 2018, [arXiv 1804.00518][uavdt]; [página oficial][uavdt-site]
- VisDrone — [repo VisDrone-Dataset][visdrone-ds] (paper en el doc previo, arXiv 2001.06303)
- DynUAV — Ye et al., CVPR 2026, [arXiv 2603.05970][dynuav]; [repo][dynuav-repo]
- MMOT — Li et al., NeurIPS 2025, [arXiv 2510.12565][mmot-html]; [HF][mmot-hf]; [repo][mmot-repo]
- MDMT — Liu et al., IEEE TMM 25 (2023); [repo][mdmt]
- VETRA — Hellekes et al., ECCV 2024; [DLR][vetra]
- MOR-UAV — Mandal et al., ACM MM 2020, [arXiv 2008.01699][moruav]; [repo][moruav-repo]
- AU-AIR — Bozcan & Kayacan, ICRA 2020, [arXiv 2001.11737][auair]
- SynDrone — [arXiv 2308.10491][syndrone]; [repo][syndrone-repo]
- pNEUMA Vision — Kim et al., TR-C 147 (2023) 103966, [Infoscience][pneuma-paper]; [Zenodo][pneuma-vision]; [toolbox][pneuma-tb]; [descargas pNEUMA][pneuma-dl]
- highD — Krajewski et al., ITSC 2018, [arXiv 1810.05642][highd]; levelXData: [highD][highd-site], [inD][ind-site], [rounD][round-site]
- CitySim — TRR 2023, [arXiv 2208.11036][citysim-paper]; [repo][citysim]
- DRIFT — [arXiv 2504.11019](https://arxiv.org/abs/2504.11019); [repo][drift-repo]; [HF][drift-hf]
- Stanford Drone — Robicquet et al., ECCV 2016; [página][sdd]
- Okutama-Action [arXiv 1706.03038][okutama] · DroneVehicle [arXiv 2003.02437][dronevehicle] · CARPK [arXiv 1707.05972][carpk] · UAV3D [arXiv 2410.11125][uav3d] · TartanAir [arXiv 2003.14338][tartanair] · SMOT4SB [arXiv 2507.12832][smot4sb]
- VIVID (cita en Sim4CV, [arXiv 1708.05869][sim4cv]) · WPAFB 2009 (cita en [arXiv 1911.01727][wami])
- CVAT — [SAM2 tracker][cvat-sam2]; [AI tools / Tracking][cvat-ai]
