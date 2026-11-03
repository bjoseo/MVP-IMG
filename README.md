# Laboratorio Práctico - Detector de Atención en Tiempo Real

**Materia:** Técnicas de Procesamiento Digital de Imágenes  
**Alumno:** José Barrionuevo  
**Repositorio:** MVP-IMG  
**Estado:** MVP Funcional - Atención 0-100%

> Aplicación web que detecta en tiempo real si una persona está prestando atención usando la webcam, combinando señales de ojos y postura de cabeza. Desarrollado con modelos pre-entrenados de MediaPipe y lógica propia de fusión.

---

## Objetivo Definido con Agente

Siguiendo la consigna, antes de generar código se utilizó un agente como mentor de producto:

**Prompt utilizado en VS Code (Continue + Ollama):**

> "Quiero construir una pequeña aplicación con inteligencia artificial. Actuá como mentor de desarrollo de productos digitales. Haceme preguntas, una por vez, para definir qué quiero lograr, qué necesito aprender, qué tecnologías utilizar y cómo construir una primera versión mínima. No escribas código todavía."

**Definición final del producto:**

- **Qué hace:** Detecta si estoy prestando atención o no frente a la PC con webcam.
- **Señales visuales:** Ojos abiertos/cerrados, dirección de cabeza (giro/cabeceo), postura.
- **Contexto:** 1 sola persona, luz de casa, PC de escritorio.
- **MVP:** Porcentaje de atención 0-100% con semáforo (verde >75%, amarillo 35-75%, rojo <35%).
- **Aprendizajes priorizados:**
  - C) Uso de dataset de Hugging Face como referencia teórica
  - D) Visualización y lógica propia con p5.js / Canvas

---

## Pipeline de Procesamiento Digital de Imágenes

```
1. Captura (640x480 @ 30fps) -> getUserMedia
2. Pre-procesamiento -> MediaPipe FaceLandmarker (WASM, GPU)
3. Extracción de características:
   - 468 landmarks faciales
   - EAR (Eye Aspect Ratio) = (|p2-p6| + |p3-p5|) / (2*|p1-p4|)
   - Blink blendshapes: eyeBlinkLeft, eyeBlinkRight
   - Head pose: dx = |nariz.x - centroOjos.x|, dy = nariz.y - 0.5
4. Fusión y Decisión:
   - Ojos cerrados (EAR<0.18 o blink>0.6) => eyeOpen = 0%
   - Ojos: 0-100% basado en EAR/blink con curva no lineal estricta
   - Cabeza: yawScore + pitchScore (penalización si dx>0.05 o dy>0.06)
   - Atención = 75% ojos + 25% cabeza, si ojos<8% => atención=eyeOpen*0.2 (fuerza 0%)
5. Suavizado: Media móvil de 5 frames
6. Segmentación : MediaPipe Selfie Segmentation separa persona de fondo
7. Visualización: Canvas overlay (puntos ojos verde, nariz amarillo) + barra % + fondo difuminado
```

---

## Tecnologías Utilizadas

| Herramienta                       | Cómo se usó                                                                               |
| --------------------------------- | ----------------------------------------------------------------------------------------- |
| **MediaPipe FaceLandmarker**      | 468 landmarks + blendshapes, modelo pre-entrenado para ojos y cabeza                      |
| **MediaPipe Selfie Segmentation** | Segmentación real persona/fondo para difuminado (V2.2)                                    |
| **Canvas / p5.js**                | Dibujado de landmarks y lógica de visualización propia                                    |
| **Hugging Face Dataset**          | `1aurent/drowsiness-detection` como referencia teórica para umbrales (EAR<0.21 = cerrado) |

**Stack extra:**

- `uv` para gestión de entorno virtual Python 3.11
- `Continue + Ollama (Llama 3.1:8b)` como agente dentro de VS Code (local y gratuito)
- VS Code + Live Server

---

## Instalación con UV (PowerShell)

```powershell
# 1. Clonar repo
git clone https://github.com/TU_USUARIO/MVP-IMG.git
cd MVP-IMG

# 2. Crear entorno con uv
uv venv --python 3.11
.\.venv\Scripts\Activate.ps1

# 3. Instalar deps base (para futura versión Python/YOLO)
uv add mediapipe opencv-python numpy
uv add --dev ipykernel jupyter

# 4. Abrir en VS Code
code .
```

### Configurar agente en VS Code

1. Extensión: `Continue - open-source AI code agent v2.0.0`
2. Loguearse en Continue Hub o usar modelo local
3. Modelo recomendado: `Llama 3.1 70B Free` (sin Ollama) o `llama3.1:8b` local con Ollama
4. Atajo para abrir: `Ctrl + L` o click en ícono Continue en sidebar

---

## Cómo ejecutar

**Opción A - Live Server (recomendada):**

1. Instalar extensión Live Server en VS Code
2. Abrir `index.html` en el editor
3. Click derecho dentro del código > Open with Live Server
4. Abre `http://127.0.0.1:5500` y permitir cámara

**Opción B - Python:**

```powershell
.\.venv\Scripts\Activate.ps1
python -m http.server 8000
# Abrir http://localhost:8000
```

**Pruebas:**

- Ojos abiertos mirando frente => 75-100% verde ATENTO
- Ojos cerrados 1s => 0-35% rojo NO ATENTO
- Cabeza agachada/costado => 35-75% amarillo MEDIA
- Botón Fondo Difuminado ON => persona nítida, fondo con blur 18px

---

## Ajuste de Umbrales

**Problema inicial:** Con ojos cerrados daba 58% (muy permisivo).  
**Solución:**

- Blink >0.6 => eyeOpen=0% directo (antes era lineal)
- EAR <0.18 => 0%, <0.21 => 10%
- Head: dx>0.08 => 5%, dy>0.10 => 5%
- Fusión final 75/25 y forzado a 0 si ojos cerrados

Esto demuestra el uso del dataset de Hugging Face como referencia para calibrar.

## Estructura del Proyecto

```
MVP-IMG/
├── .venv/                 # Entorno virtual (no se sube)
├── .gitignore
├── pyproject.toml         # Gestionado por uv
├── index.html             # V2.2 - Atención + Fondo difuminado
├── style.css              # Estilos separados
├── README.md              # Este archivo
└── docs/                  # Capturas de mentoría con agente (para defensa)
```

---

## Información

1. **Interacción con agente:** Capturas de Continue en VS Code haciendo las 5 preguntas de mentoría
2. **Alucinación detectada:** Continue intentó leer `/path/to/dataset/Hugging_Face_dataset.json` y `github raw data.json` inexistentes -> se corrigió explicando que HF se usa vía `datasets` library, no JSON local. buen punto para explicar limitaciones de LLMs locales.
3. **Repo con uv + GitHub:** Commits con modificaciones
4. **Demo en vivo:** funcionando en localhost con webcam

---

## Licencia

Uso educativo - Tecnicatura en CD e IA 2026 - Imágenes
