# MeowCV — Guía de migración e instalación

Detector de expresiones faciales con OpenCV + MediaPipe que muestra memes de
gatos de TikTok según tu expresión, en tiempo real, usando la cámara de tu
celular conectada por USB.

Este documento migra el proyecto original (`inteligencia_tiempo_real.py`,
basado en YOLOv8 + DeepFace) hacia el enfoque **MeowCV**, renombrando el
script principal a `meowcv.py`.

**Repositorio de referencia:** https://github.com/reinesana/MeowCV
(la instalación de este documento sigue como guía principal el `README.md`
de ese repositorio, citado en el historial de esta conversación).

---

## Índice

1. [Por qué migrar de YOLO+DeepFace a MediaPipe](#1-por-qué-migrar-de-yolodeepface-a-mediapipe)
2. [Instalación base (siguiendo el README de MeowCV)](#2-instalación-base-siguiendo-el-readme-de-meowcv)
3. [Configurar Visual Studio Code para Python + OpenCV](#3-configurar-visual-studio-code-para-python--opencv)
4. [Migración del código a MeowCV](#4-migración-del-código-a-meowcv)
5. [Activar depuración USB y conectar el celular con DroidCam](#5-activar-depuración-usb-y-conectar-el-celular-con-droidcam)
6. [Ejecución y calibración](#6-ejecución-y-calibración)
7. [Checklist final](#checklist-final)

---

## 1. Por qué migrar de YOLO+DeepFace a MediaPipe

| Enfoque original | MeowCV |
|---|---|
| YOLOv8 detecta "persona" en el frame completo | MediaPipe Face Mesh detecta 478 landmarks faciales directamente |
| DeepFace clasifica la emoción con un modelo de deep learning sobre el recorte de cabeza | Reglas geométricas simples (distancias entre landmarks) determinan la expresión |
| Pesado: se corre DeepFace cada N cuadros para no saturar la CPU | Liviano: corre en cada frame sin problema |
| Requiere `ultralytics` + `deepface` + `tensorflow` de fondo | Solo requiere `opencv-python` + `mediapipe` |
| Emociones: happy, sad, angry, surprise, neutral, fear, disgust | Expresiones: shock (ojos muy abiertos), tongue/lengua (boca abierta), glare (ojos entrecerrados), neutral |

El motivo práctico de migrar: DeepFace es lento y poco confiable en tiempo
real sobre un recorte pequeño hecho por YOLO, mientras que MediaPipe Face
Mesh está diseñado específicamente para trackear landmarks faciales rápido y
en vivo — que es exactamente lo que necesita este proyecto.

---

## 2. Instalación base (siguiendo el README de MeowCV)

### 2.1 Descarga la documentación oficial como referencia

```bash
git clone https://github.com/reinesana/MeowCV.git
cd MeowCV
cat README.md
```

O si solo quieres el archivo README sin clonar todo el repo:

```bash
curl -o MeowCV_README.md https://raw.githubusercontent.com/reinesana/MeowCV/main/README.md
```

### 2.2 Requisitos de versión (tal como los especifica el README)

- **Python 3.9 – 3.12** (el README indica que fue probado en 3.11.7).
  **Python 3.13+ no es compatible** con `mediapipe==0.10.14`.
- Dependencias: `opencv-python`, `mediapipe`.

### 2.3 Instala Python correctamente (Windows)

Ve a https://www.python.org/downloads/windows/ y descarga el **instalador
clásico `.exe`** (no el `.msix` de Microsoft Store — ver Troubleshooting
2.6). En el instalador, marca la casilla **"Add python.exe to PATH"** antes
de darle a Install Now.

Verifica:
```powershell
python --version
```

### 2.4 Crea el entorno virtual

```powershell
python -m venv meowcv_env
.\meowcv_env\Scripts\activate
```

En Linux/macOS:
```bash
python3 -m venv meowcv_env
source meowcv_env/bin/activate
```

### 2.5 Instala las dependencias fijando versión de MediaPipe

```bash
pip install opencv-python mediapipe==0.10.14
```

> **Importante:** se fija `mediapipe==0.10.14` explícitamente en vez de
> instalar "lo último". Ver Troubleshooting 2.6 para entender por qué.

### 2.6 Troubleshooting — Instalación

- **`AttributeError: module 'mediapipe' has no attribute 'solutions'`**
  A partir de `mediapipe 1.0.0`, la librería reemplazó la API vieja
  (`mp.solutions.face_mesh`, la que usa MeowCV) por una nueva "Tasks API".
  Si instalaste `mediapipe` sin fijar versión, `pip` trae la última (1.0.0+)
  y el código deja de funcionar. Solución:
  ```bash
  pip uninstall mediapipe -y
  pip install mediapipe==0.10.14
  ```
  Este es el error de incompatibilidad de versiones más común de todo el
  proceso — pasó exactamente así durante el desarrollo de este proyecto.

- **Tienes Python 3.13+ instalado**: `mediapipe==0.10.14` no tiene wheel
  compatible. Instala Python 3.12 en paralelo (no lo desinstales) y crea el
  entorno virtual apuntando específicamente a esa versión:
  ```powershell
  py -3.12 -m venv meowcv_env
  ```

- **`error: externally-managed-environment` (Linux/Debian/Ubuntu)**:
  Python 3.12+ en distros basadas en Debian bloquea `pip install` fuera de
  un entorno virtual. Siempre activa el `venv` (paso 2.4) antes de instalar
  — no uses `--break-system-packages` como solución permanente.

- **Descargaste un `.msix` en vez de un `.exe` (Windows)**: python.org
  ofrece tanto el instalador clásico como la versión de Microsoft Store
  (`.msix`). Usa el **instalador clásico** — la versión de Store corre en un
  entorno restringido que puede dar problemas de permisos con `pip`.

- **PowerShell no permite activar el entorno virtual** (error de política de
  ejecución de scripts): ejecuta una sola vez, como administrador:
  ```powershell
  Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
  ```

- **Estás dentro de WSL2 y algo no cuadra**: WSL2 usa un kernel
  personalizado de Microsoft, sin los `linux-headers-*` estándar de apt.
  Para este proyecto (que necesita acceso directo a la cámara por USB), se
  recomienda **no usar WSL2** y correr todo en Windows nativo — ver sección
  5.4.

---

## 3. Configurar Visual Studio Code para Python + OpenCV

### 3.1 Instala la extensión de Python

Abre VS Code → panel de Extensiones (`Ctrl+Shift+X`) → busca **"Python"**
(publicada por **Microsoft**, ícono azul/amarillo). Es la única extensión
necesaria — trae Pylance (autocompletado) como dependencia automática.

No es necesario instalar "Python Extension Pack" ni extensiones sueltas de
linters para este proyecto.

### 3.2 Abre la carpeta del proyecto (no el archivo suelto)

`File > Open Folder` → selecciona la carpeta `meowcv` completa (con
`meowcv.py` y la subcarpeta `assets/` dentro). Abrir la carpeta completa es
importante para que las rutas relativas a `assets/` funcionen.

### 3.3 Selecciona el intérprete correcto (tu entorno virtual)

`Ctrl+Shift+P` → escribe **"Python: Select Interpreter"** → elige el que
está dentro de `meowcv_env` (algo como
`.\meowcv_env\Scripts\python.exe`). Si no aparece en la lista, usa
"Enter interpreter path" y navega manualmente hasta ese archivo.

### 3.4 Ejecuta y depura

- Botón **▶ (Run Python File)** arriba a la derecha, o `Ctrl+F5`, para correr
  el script normal.
- `F5` (Run > Start Debugging) para correr con el debugger — útil para poner
  breakpoints, por ejemplo en el cálculo de `eye_opening` y `mouth_open`, y
  revisar esos valores frame a frame.

### 3.5 Troubleshooting — VS Code

- **Errores de incompatibilidad de mediapipe dentro de VS Code**: el
  intérprete seleccionado (paso 3.3) debe ser el del entorno virtual donde
  corriste `pip install mediapipe==0.10.14`. Si seleccionaste el Python
  global del sistema por error, vas a tener el mismo problema de versión
  del punto 2.6 aunque el `pip install` en la terminal haya sido correcto.
  Verifica con la terminal integrada de VS Code (que ya usa el intérprete
  seleccionado):
  ```bash
  pip show mediapipe
  ```

---

## 4. Migración del código a MeowCV

### 4.1 Elimina las dependencias del proyecto original

Quita del script:
- `from ultralytics import YOLO`
- `from deepface import DeepFace`
- La carga del modelo `model_yolo = YOLO("yolov8n.pt")`
- El bloque de detección de persona y recorte de cabeza con YOLO
- El bloque de análisis con `DeepFace.analyze(...)`

### 4.2 Agrega MediaPipe Face Mesh

```python
import mediapipe as mp

face_mesh = mp.solutions.face_mesh.FaceMesh(
    min_detection_confidence=0.5,
    min_tracking_confidence=0.5,
    max_num_faces=1
)
```

### 4.3 Implementa las reglas geométricas de expresión

Usando los índices de landmarks de MediaPipe para ojos (159/145 y 386/374) y
boca (13/14):

```python
def medir_apertura_ojos(face_landmark_points):
    l_top = face_landmark_points.landmark[159]
    l_bot = face_landmark_points.landmark[145]
    r_top = face_landmark_points.landmark[386]
    r_bot = face_landmark_points.landmark[374]
    return (abs(l_top.y - l_bot.y) + abs(r_top.y - r_bot.y)) / 2.0

def medir_apertura_boca(face_landmark_points):
    top_lip = face_landmark_points.landmark[13]
    bottom_lip = face_landmark_points.landmark[14]
    return abs(top_lip.y - bottom_lip.y)

def cat_shock(eye_opening):
    return eye_opening > eye_opening_threshold

def cat_tongue(mouth_open):
    return mouth_open > mouth_open_threshold

def cat_glare(eye_opening):
    return eye_opening < squinting_threshold
```

### 4.4 Renombra el archivo principal a `meowcv.py`

Estructura final de carpeta:
```
meowcv/
├── meowcv.py
└── assets/
    ├── larry.jpeg
    ├── cat-shock.jpeg
    ├── cat-tongue.jpeg
    ├── cat-glare.jpeg
    └── cat-disgust.jpeg
```

### 4.5 Troubleshooting — Migración del código

- **El programa detecta rostro pero nunca cambia de expresión**: revisa que
  no quede ningún condicional comparando tipos incorrectamente, por ejemplo
  `variable is np.ndarray` en vez de `isinstance(variable, np.ndarray)` —
  ese bug bloquea silenciosamente el bloque de análisis completo sin lanzar
  ningún error visible.

- **Todas las expresiones disparan la misma categoría (ej. "SHOCK") sin
  importar tu cara real**: los umbrales de referencia del README fueron
  calibrados con la cámara y el ángulo del autor original — no son
  universales. Activa un modo debug que imprima en pantalla los valores
  reales de `eye_opening` y `mouth_open` en vivo, compáralos con cara
  neutral vs. cara exagerada, y ajusta los tres umbrales con tus propios
  números.

- **Los umbrales de ojos (shock/glare) compiten entre sí**: si
  `eye_opening_threshold` queda muy bajo, tu cara neutral puede caer cerca
  del límite de "entrecerrado" (`squinting_threshold`) también. Ajusta
  ambos umbrales juntos, no de a uno.

---

## 5. Activar depuración USB y conectar el celular con DroidCam

### 5.1 Por qué USB en vez de WiFi/HTTP

Usar el stream HTTP de DroidCam
(`cv2.VideoCapture("http://IP:PUERTO/video")`) agrega latencia de red,
compresión/decompresión por cada frame, y depende de la calidad del WiFi.
La conexión por **USB** evita todo eso: el celular aparece como una webcam
local del sistema, igual que en el proyecto original de MeowCV (que usa
`cv2.VideoCapture(0)` directo sobre una webcam conectada).

### 5.2 Activa la depuración USB en el celular (Android)

1. Ve a **Ajustes → Acerca del teléfono**.
2. Toca 7 veces seguidas sobre **"Número de compilación"** hasta que
   aparezca el mensaje "Ya eres desarrollador".
3. Vuelve a **Ajustes**, entra a **Opciones de desarrollador** (nueva
   sección que apareció).
4. Activa el interruptor **"Depuración USB"**.
5. Conecta el celular a la PC por cable USB. Va a aparecer un diálogo en el
   celular pidiendo autorizar la PC — acepta ("Permitir siempre desde esta
   computadora").

### 5.3 Instala el cliente DroidCam

- **Windows**: descarga el cliente oficial desde `droidcam.app/windows`.
  Instálalo — agrega un driver de cámara virtual normal a Windows, sin
  módulos de kernel ni configuración adicional.
- **Linux nativo** (no WSL2): requiere compilar `v4l2loopback-dkms` junto al
  cliente `droidcam-cli` desde https://www.dev47apps.com/droidcam/linux/.

### 5.4 Troubleshooting — USB en WSL2

- **Estás en WSL2 y quieres conectar por USB**: no lo intentes ahí
  directamente. WSL2 corre sobre un kernel personalizado de Microsoft sin
  los `linux-headers-*` estándar de apt, lo que hace prácticamente
  inviable compilar el módulo `v4l2loopback` necesario para DroidCam por
  USB (requeriría pasar el dispositivo con `usbipd-win` y luego compilar el
  módulo a mano contra el kernel-source de Microsoft — mucho esfuerzo para
  poco beneficio). **La alternativa recomendada es correr el proyecto en
  Windows nativo** (no dentro de la subcapa Linux), usando el cliente
  Windows de DroidCam del paso 5.3.

### 5.5 Conecta el celular

1. Abre la app **DroidCam** en el celular.
2. Abre el **cliente DroidCam en la PC**.
3. En ambos, selecciona modo **USB** (no WiFi).
4. Presiona conectar/Start en ambos lados.

### 5.6 Captura el video en Python

```python
INDICE_CAMARA = 0  # prueba 0, 1, 2... según cuántas cámaras tengas

# En Windows, CAP_DSHOW evita errores comunes de detección de índice
cam = cv2.VideoCapture(INDICE_CAMARA, cv2.CAP_DSHOW)

# Buffer chico: evita acumular frames viejos (lag)
cam.set(cv2.CAP_PROP_BUFFERSIZE, 1)

if not cam.isOpened():
    print(f"Error: No se puede acceder a la cámara en el índice {INDICE_CAMARA}.")
    exit()
```

Para saber qué índice corresponde al celular, abre la app de Cámara de
Windows o cualquier programa de videollamada y revisa en qué posición de la
lista aparece "DroidCam Source" (si además tienes la webcam integrada de tu
notebook, prueba con 0, 1, 2 hasta encontrar la correcta).

### 5.7 Troubleshooting — Conexión de cámara

- **La cámara va lenta / con delay**: si estás usando el modo WiFi/HTTP en
  vez de USB, es esperable por la latencia de red. Migrar a USB (esta
  sección) resuelve la mayoría de los casos. Si además corrías desde WSL2,
  la capa de red virtualizada agregaba latencia extra.

- **`cam.isOpened()` da `False`, o abre la webcam integrada en vez del
  celular**: prueba otros valores de `INDICE_CAMARA` (0, 1, 2...) — cuando
  hay más de una cámara en el sistema, el orden de los índices no siempre es
  predecible.

- **La imagen del celular se ve espejada**: el script usa
  `cv2.flip(image, 1)` para corregir esto por defecto (así lo hace MeowCV
  original, pensado para webcams frontales). Si tu cámara ya llega en la
  orientación correcta, quita esa línea.

---

## 6. Ejecución y calibración

1. Con el celular conectado y DroidCam en modo USB activo, corre
   `meowcv.py` desde VS Code (`▶` o `Ctrl+F5`).
2. Deberían abrirse dos ventanas: **"Face Detection"** (tu cámara con los
   landmarks dibujados) y **"Cat Image"** (el gato meme correspondiente).
3. Si activaste el modo debug, vas a ver en pantalla los valores numéricos
   de `ojos=` y `boca=` en vivo — úsalos para calibrar los tres umbrales
   (`eye_opening_threshold`, `mouth_open_threshold`, `squinting_threshold`)
   con tu propia cara y cámara, en vez de los valores de referencia del
   README (calibrados con la cámara del autor original).

---

## Checklist final

- [ ] Python 3.9–3.12 instalado (instalador `.exe`, no `.msix`), con
      "Add to PATH" marcado
- [ ] Entorno virtual `meowcv_env` creado y activado
- [ ] `opencv-python` y `mediapipe==0.10.14` instalados dentro del venv
- [ ] Extensión "Python" (Microsoft) instalada en VS Code, con el
      intérprete del venv seleccionado
- [ ] Código migrado: sin YOLO/DeepFace, con MediaPipe Face Mesh
- [ ] Script renombrado a `meowcv.py`, junto a la carpeta `assets/`
- [ ] Depuración USB activada en el celular y autorizada para la PC
- [ ] Cliente DroidCam instalado y conectado en modo **USB**
- [ ] `INDICE_CAMARA` ajustado al índice correcto
- [ ] Umbrales calibrados con valores reales propios (modo debug)
- [ ] Script corriendo y mostrando ambas ventanas (rostro + gato meme)
