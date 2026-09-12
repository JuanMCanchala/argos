# Argos

Deteccion de incidentes en video en tiempo real, **zero-shot y multi-dominio**.
Un mismo sistema cubre robo en tienda, agresiones, seguridad industrial y caidas
sin entrenar un modelo por vertical: cambiar de dominio es cambiar un YAML.

> El proyecto se presento en la hackaton como **Sentinel**. El repositorio se renombro a
> **Argos** — por Argos Panoptes, el vigia de cien ojos — para no confundirlo con
> [centinela](https://github.com/JuanMCanchala/centinela), que es otro proyecto.
> Las variables de entorno conservan el prefijo `SENTINEL_`.

---

## Layout del monorepo

| Carpeta | Rol |
|---|---|
| `backend/` | Pipeline del **modelo** (Python): captura, VLM, dominios YAML |
| `convex-backend/` | Backend de **producto Sentra** (Convex/TypeScript): workspaces, cámaras, incidentes, API contract |
| `frontend/` | UI del producto |

---

## Por que no entrenamos un modelo

Es la pregunta que hara el jurado. La respuesta corta: la literatura de 2025-2026
dice que no hace falta, y los datasets publicos de robo son una trampa.

| Enfoque | Problema |
|---|---|
| Detector de objetos con clase "shoplifting" (Roboflow) | Un detector no ve *acciones*. Aprende el fondo de la tienda del dataset y no generaliza a otra camara |
| CNN 3D sobre UCF-Crime / DCSASS | El subset de robo es pequeno y de CCTV degradada. Entrenado ahi, en una demo en vivo no dispara nunca |
| VLM zero-shot orquestado | Es el estado del arte actual y no necesita anotacion |

Referencias que sostienen la decision:

- **LAVIDA** — 82.18% AUC en UCF-Crime *sin datos de anomalias*. [arXiv](https://arxiv.org/html/2602.19248v1)
- **ASK-HINT** — prompting estructurado, SOTA training-free. [WACV 2026](https://openaccess.thecvf.com/content/WACV2026/papers/Zou_Unlocking_Vision-Language_Models_for_Video_Anomaly_Detection_via_Fine-Grained_Prompting_WACV_2026_paper.pdf)
- **Paza-AI** — YOLO + tracking + VLM, model-agnostic y sin fine-tuning. [arXiv](https://arxiv.org/pdf/2604.14846)
- **Veesion** (referencia comercial) — 85-90% de precision en alertas, o sea 1 de cada 7 es falsa. Ese es el techo real del sector.

---

## Arquitectura: cascada de 3 etapas

Mandar todos los frames a un VLM es inviable por coste, latencia y rate limit.
La cascada resuelve las tres cosas a la vez.

```
Camara (webcam / archivo / RTSP)
  |
  +-- ETAPA 0   siempre, ~30 FPS, local, coste 0
  |   YOLO11-pose + ByteTrack -> tracks con 17 keypoints
  |   Buffer circular de 12 s en RAM (JPEG, ~30 KB/frame)
  |
  +-- ETAPA 1   filtro geometrico -- descarta la inmensa mayoria
  |   senales normalizadas por el torso: concealment, motion, proximity,
  |   fall, immobility, dwell, zone, presence
  |   score ponderado por dominio -> sostenido -> dispara
  |
  +-- ETAPA 2   VLM, solo en disparos (~1 cada 15-45 s por persona)
  |   10 frames del buffer con el sujeto marcado -> Gemini -> JSON validado
  |
  +-- ETAPA 3   evento, clip de evidencia, cronologia, WebSocket, aviso externo
```

**Multi-camara.** N camaras comparten UN presupuesto de inferencia repartido
round-robin, no N pipelines. Cada feed lleva su propia instancia de tracker
porque ByteTrack guarda estado y mezclarlo confundiria las identidades de una
sala con las de otra. Medido con dos feeds en CPU: 28,6 + 12,0 = 40,6 FPS de
inferencia total, sin que ninguna camara ahogue a la otra.

Que gana cada etapa: la 0 y la 1 dan el "tiempo real" honesto; la 2 da la
explicacion en lenguaje natural que un operador puede leer y auditar; la 3
cierra el bucle con la persona que decide.

**Sesgo.** El sistema no usa biometria ni reconocimiento facial. La Etapa 1 solo
mide geometria corporal, y al VLM se le prohibe escribir una sola palabra sobre
raza, etnia, genero, edad, complexion o estilo de vestir, incluso si se lo
preguntan directamente en el chat. La unica excepcion es el equipo de proteccion
en el dominio industrial, porque comprobar si alguien lleva casco *es* el
trabajo. Cada alerta guarda su evidencia textual, asi que es auditable despues.

Sobre los keypoints de la cara: se usan nariz y orejas para estimar hacia donde
apunta la cabeza, igual que se usan hombros y caderas para saber hacia donde
apunta el torso. Es **geometria de pose, no biometria**: no se genera ninguna
huella facial, no se identifica a nadie y no se compara a una persona con otra
ni consigo misma en otro momento. Las tracks son numeros que se olvidan a los
2 segundos de que la persona sale de cuadro.

---

## Dominios

Cada vertical es un YAML en `backend/domains/`. Anadir una no toca Python.

| Dominio | Senales que pesan | Dispara con |
|---|---|---|
| `retail_theft` | concealment 0.62, dwell 0.26, motion 0.12 | mano que alcanza un estante y vuelve al torso |
| `violence` | motion 0.55, proximity 0.45 | energia alta + dos personas cerca |
| `fall_detection` | fall 0.62, immobility 0.38 | caida no controlada + no levantarse |
| `industrial_safety` | presence, dwell, zone, fall | auditoria periodica de EPP y zonas restringidas |

---

## Puesta en marcha

```powershell
# 1. Dependencias
py -3.13 -m venv .venv
.venv\Scripts\python.exe -m pip install -r requirements.txt
cd frontend; npm install; cd ..
```

> **Sobre la GPU.** `requirements.txt` trae torch desde PyPI, que en Windows es
> CPU-only. YOLO11n-pose en CPU da 10-15 FPS, de sobra para la Etapa 0. Para usar
> la GPU hace falta el indice propio de PyTorch:
>
> ```powershell
> .venv\Scripts\python.exe -m pip install --force-reinstall torch torchvision --index-url https://download.pytorch.org/whl/cu124
> ```
>
> Ese indice se quedo colgado a 0 bytes en la red de la hackathon. Si no baja, no
> insistas: deja `SENTINEL_DEVICE=cpu` y sigue. La GPU no es el cuello de botella
> de este sistema, la latencia del VLM lo es.

```powershell
# 2. Configuracion
copy .env.example .env      # y pon tu GEMINI_API_KEY

# 3. Arrancar los dos servicios
.\dev.ps1
```

- Dashboard: <http://localhost:5173>
- API: <http://localhost:8000/api/state>

Sin `GEMINI_API_KEY` el pipeline arranca igual en **modo offline**: todo funciona
salvo el veredicto real, que se sustituye por uno sintetico. Sirve para
desarrollar sin gastar cuota.

### Variables

| Variable | Por defecto | Para que |
|---|---|---|
| `GEMINI_API_KEY` | — | Key de [AI Studio](https://aistudio.google.com/apikey) |
| `GEMINI_MODEL` | `gemini-3.5-flash-lite` | Modelo de la Etapa 2 |
| `SENTINEL_SOURCE` | `0` | Camaras separadas por coma, con nombre opcional: `Entrada=0,Almacen=rtsp://...` |
| `SENTINEL_DOMAIN` | `retail_theft` | Dominio inicial |
| `SENTINEL_DEVICE` | `cuda` | `cuda` o `cpu` |
| `SENTINEL_POSE_IMGSZ` | `480` | Resolucion de inferencia. Subir a 640 si sobra GPU |
| `TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID` | — | Avisa al movil con la foto del incidente |
| `ALERT_WEBHOOK_URL` | — | Webhook generico (WhatsApp via FunnelChat, Slack, Discord) |
| `PUBLIC_BASE_URL` | — | Base publica para que el webhook incluya un enlace a la evidencia |

**Eleccion de modelo.** Medido sobre esta misma tarea con 10 frames:

| Modelo | Latencia Etapa 2 |
|---|---|
| `gemini-3.5-flash-lite` | **4,4 s** |
| `gemini-flash-lite-latest` | 5,5 s |
| `gemini-3.7-flash` | 9,9 s |

`flash-lite` va por defecto: es 2,2x mas rapido, la calidad del razonamiento
aguanta, y en free tier tiene mas cuota diaria. Si algun veredicto dificil sale
flojo, sube a `gemini-3.7-flash` y asume ~5 s mas por alerta.

Poner `SENTINEL_SOURCE` a un archivo lo reproduce **en bucle**: es el plan B para
la demo si la camara falla o la sala esta demasiado llena.

---

## Medir antes de presentar

Un numero medido vale mas que cualquier adjetivo en el pitch. El benchmark
reporta las **dos etapas por separado**, que es lo unico que demuestra si la
cascada aporta algo: la Etapa 1 debe tener recall alto aunque falle en
precision, y la Etapa 2 debe tumbar los falsos positivos sin llevarse el recall.

```powershell
# 1. Bajar un dataset publico y dejarlo con nombres pos_/neg_
.venv\Scripts\kaggle.exe auth login
.venv\Scripts\python.exe -m tools.get_dataset --list
.venv\Scripts\python.exe -m tools.get_dataset rwf2000

# 2. Etapa 1 sobre todo el conjunto: gratis y ~3x mas rapido que tiempo real
.venv\Scripts\python.exe -m tools.bench data\rwf2000 --domain violence --no-vlm

# 3. Etapa 1+2 sobre una muestra, que si gasta cuota de API
.venv\Scripts\python.exe -m tools.bench data\rwf2000 --domain violence ^
    --limit 40 --out data\bench_rwf2000.json
```

Si teneis clips propios, `tools/prepare_dataset.py` los adapta igual. Grabar 10
`pos_` y 10 `neg_` en la sala de la demo sigue siendo lo mas valioso: es la
unica distribucion que coincide con lo que vera el jurado.

### Datasets y sus trampas

| Dataset | Para que sirve | Trampa |
|---|---|---|
| RWF-2000 | Violencia. El mas limpio: 5 s, balanceado, split predefinido | Kaggle lo mirrorea; el original pide permiso al SMIIP Lab |
| RLVS | Violencia, fuentes mas variadas | Mezcla peliculas con CCTV real |
| UCF-Crime | Credibilidad de benchmark | 240p de CCTV: la pose degrada y la Etapa 1 pierde recall. Decirlo en el pitch suma, no resta |

---

## Que mas hace

- **Sube un MP4 y lo analiza** con la misma cascada. Un jurado puede traer su
  propio video. Medido: 12,5 s de video en 8,4 s, mas rapido que tiempo real.
- **Cronologia del incidente** derivada de las senales medidas frame a frame,
  no inventada por el modelo. Cada linea es auditable contra su numero.
- **Pregunta sobre un incidente**: se le devuelven los mismos frames al VLM para
  que mire otra vez, en vez de fiarse de su propio resumen.
- **Avisos externos**: Telegram con la foto, o webhook generico para WhatsApp via
  FunnelChat, Slack o Discord. Solo con incidentes que la Etapa 2 confirma.
- **Boton de pausa** que suelta las camaras sin tumbar la API.

## API

El contrato completo, con los gotchas, esta en [API.md](API.md).

| Metodo | Ruta | Que hace |
|---|---|---|
| GET | `/api/state` | FPS, camaras, senales vivas, estadisticas |
| GET | `/api/cameras` | Estado por camara |
| GET | `/api/domains` · POST `/api/domain/{id}` | Verticales; cambio en caliente |
| GET | `/api/events` | Historial de eventos |
| POST | `/api/events/{id}/feedback` | `confirmed` o `false_positive` |
| POST | `/api/events/{id}/chat` | Pregunta libre sobre un incidente |
| POST | `/api/analyze` · GET `/api/jobs` | Sube y analiza un video |
| POST | `/api/pause` · `/api/resume` | Suelta o reabre las camaras |
| GET | `/video.mjpg?camera={id}` | Video anotado en vivo |
| WS | `/ws` | Estado cada 500 ms, eventos y progreso al vuelo |

## Instalación rápida

Deja el motor de visión funcionando en una máquina nueva: entorno de Python,
dependencias, el modelo de pose y la configuración mínima.

```powershell
# Windows
.\instalar.ps1
```

```bash
# Linux / macOS
./instalar.sh
```

Al terminar arranca el motor en `http://localhost:8000` y te da el enlace del
panel. El panel se conecta solo a tu máquina: abre **En vivo** para ver la
cámara y prueba con un clip de `prueba/`.

Opciones: `-SoloModelo` / `--solo-modelo` baja solo el modelo YOLO;
`-NoArrancar` / `--no-arrancar` instala sin levantar nada.

Sin `GEMINI_API_KEY` en el `.env` el motor arranca igual: el filtro geométrico
funciona y las detecciones quedan sin verificar, que basta para ver el sistema
moverse. La clave se saca en [aistudio.google.com/apikey](https://aistudio.google.com/apikey).
