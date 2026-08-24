# Bitácora de Integración y Calibración: Cámara Industrial Daheng Galaxy

**Fecha:** 24 de agosto de 2026  
**Entorno de Trabajo:** Ubuntu Linux | Python 3 | SDK `gxipy` | OpenCV 

---

## 📌 Resumen de Avances del Día

Hoy se logró la configuración completa de la interfaz de captura a alta velocidad, la optimización de parámetros ópticos y la detección robusta del patrón de calibración en visión artificial.

---

## ⚙️ hitos Alcanzados

### 1. Optimización de Rendimiento y ROI (`roi_fps_test.py`)
* **Resolución recortada (ROI):** Reducción de la resolución completa (~20MP) a una región de interés de **$2400 \times 2400$** con offset vertical ($Y=200$) para centrar el objetivo.
* **Tasa de refresco alcanzada:** **37.62 FPS reales** en transmisión continua sostenida.
* **Ajuste de Exposición y Ganancia:** Se compensó el fuerte contraluz de la escena aumentando la exposición a **`80000.0 µs` (80 ms)** y aplicando **`10.0 dB`** de ganancia digital.

### 2. Detección Subpíxel del Tablero de Ajedrez (`detect_chessboard.py`)
* **Especificación del Patrón:** Modelo **`GP200-15-12x9`**. El número de esquinas internas (intersecciones) a detectar se fijó en **`11 x 8`**.
* **Tratamiento de Imagen:** Uso del algoritmo **CLAHE** (ecualización de histograma adaptativa) para eliminar deslumbramientos y sombras severas producidas por la iluminación trasera.
* **Resultado:** Se logró la alineación y dibujo exacto de las 88 intersecciones con precisión **Sub-Píxel** mediante `cv2.cornerSubPix`.

---

## 🛠️ Archivos del Proyecto Creados/Modificados

| Archivo | Descripción |
| :--- | :--- |
| `roi_fps_test.py` | Configura el streaming, establece el ROI, controla la exposición/ganancia y mide la tasa de cuadros por segundo (FPS). |
| `detect_chessboard.py` | Carga la imagen recortada, aplica filtros de contraste y detecta las esquinas internas del tablero ($11 \times 8$). |
| `calibrate_capture.py` | Script interactivo para tomar y guardar múltiples capturas (`sample_XX.png`) desde distintas orientaciones. |

---

## 🚀 Próximos Pasos (Fase 3: Calibración Intrínseca)

Mañana continuaremos con los siguientes puntos:

1. **Captura de Muestras:**  
   Ejecutar `python calibrate_capture.py` y tomar entre 10 y 15 imágenes del tablero cambiando ligeramente su inclinación, ángulo y distancia respecto al lente.
2. **Cálculo de la Matriz Intrínseca ($K$):**  
   Ejecutar el script de calibración matemática mediante `cv2.calibrateCamera` para obtener:
   * **Focales y Centro Óptico:** $f_x, f_y, c_x, c_y$
   * **Coeficientes de Distorsión ($D$):** $k_1, k_2, p_1, p_2, k_3$
3. **Guardado de Parámetros:**  
   Exportar la matriz a un archivo `.npz` o `.json` para la corrección de distorsión de la lente en tiempo real.
