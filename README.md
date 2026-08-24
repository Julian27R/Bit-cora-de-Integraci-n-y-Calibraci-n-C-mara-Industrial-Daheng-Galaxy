```markdown
# Bitácora de Integración y Calibración: Cámara Industrial Daheng Galaxy

**Fecha:** 24 de agosto de 2026  
**Entorno:** Ubuntu Linux | Python 3 | SDK `gxipy` | OpenCV

---

## 📌 Resumen de la Sesión

Hoy se configuró la captura a alta velocidad mediante ROI, se compensó la iluminación por contraluz mediante parámetros de la cámara y se validó la detección matemática del patrón de ajedrez ($11 \times 8$ intersecciones) a nivel Sub-Píxel.

---

## 🛠️ Historial de Archivos y Comandos Ejecutados

### 1. Prueba de Rendimiento y ROI (`roi_fps_test.py`)

Configura el ROI ($2400 \times 2400$), ajusta la exposición a **80 ms** y la ganancia a **+10 dB** para vencer la subexposición por contraluz, logrando **37.62 FPS**.

* **Código del archivo:**
```python
import os
import sys
import time

sys.path.insert(0, "./api")
import gxipy as gx

try:
    import cv2
    HAS_CV2 = True
except ImportError:
    from PIL import Image as PILImage
    HAS_CV2 = False

def set_roi_safe(cam, target_w, target_h, offset_y_shift):
    cam.OffsetX.set(0)
    cam.OffsetY.set(0)

    w_range = cam.Width.get_range()
    h_range = cam.Height.get_range()

    width = min(max(target_w, w_range['min']), w_range['max'])
    height = min(max(target_h, h_range['min']), h_range['max'])

    width = (width // w_range['inc']) * w_range['inc']
    height = (height // h_range['inc']) * h_range['inc']

    cam.Width.set(width)
    cam.Height.set(height)

    max_offset_x = 5496 - width
    x_range = cam.OffsetX.get_range()
    y_range = cam.OffsetY.get_range()

    offset_x = min(max_offset_x // 2, x_range['max'])
    offset_y = min(max(offset_y_shift, y_range['min']), y_range['max'])

    offset_x = (offset_x // x_range['inc']) * x_range['inc']
    offset_y = (offset_y // y_range['inc']) * y_range['inc']

    cam.OffsetX.set(offset_x)
    cam.OffsetY.set(offset_y)

    print(f"-> ROI Ajustado: {width}x{height} en Offset ({offset_x}, {offset_y})")

def main():
    device_manager = gx.DeviceManager()
    dev_num, _ = device_manager.update_device_list()

    if dev_num == 0:
        print("No se encontró ninguna cámara.")
        return

    cam = None
    try:
        cam = device_manager.open_device_by_index(1)
        print("Cámara abierta.")

        cam.TriggerMode.set(gx.GxSwitchEntry.OFF)

        # Ajustes de Exposición (80 ms) y Ganancia (+10 dB)
        if hasattr(cam, 'ExposureTime'):
            cam.ExposureTime.set(80000.0) 
        if hasattr(cam, 'Gain'):
            cam.Gain.set(10.0)

        # ROI centrado horizontalmente con OffsetY = 200
        set_roi_safe(cam, target_w=2400, target_h=2400, offset_y_shift=200)

        cam.stream_on()
        print("\nCapturando frame corregido con ROI...")

        time.sleep(0.1)
        raw_image = cam.data_stream[0].get_image(timeout=2000)

        if raw_image is not None:
            rgb_img = raw_image.convert("RGB")
            numpy_array = rgb_img.get_numpy_array()

            os.makedirs("captures", exist_ok=True)
            output_file = os.path.join("captures", "frame_roi_crop.png")
            if HAS_CV2:
                bgr = cv2.cvtColor(numpy_array, cv2.COLOR_RGB2BGR)
                cv2.imwrite(output_file, bgr)
            else:
                pil_img = PILImage.fromarray(numpy_array)
                pil_img.save(output_file)
            print(f"[ÉXITO] Nueva imagen guardada en: {output_file}")

    except Exception as e:
        print(f"Error: {e}")

    finally:
        if cam is not None:
            try:
                cam.stream_off()
                cam.close_device()
                print("Cámara cerrada.")
            except Exception as e:
                print(f"Error al cerrar: {e}")

if __name__ == "__main__":
    main()

```

* **Comando para ejecutar:**

```bash
python roi_fps_test.py

```

---

### 2. Detección del Tablero de Ajedrez (`detect_chessboard.py`)

Aplica ecualización adaptativa (CLAHE) y detecta las **$11 \times 8$ esquinas internas** del patrón `GP200-15-12x9` a precisión Sub-Píxel.

* **Código del archivo:**

```python
import os
import cv2

def main():
    image_path = os.path.join("captures", "frame_roi_crop.png")
    if not os.path.exists(image_path):
        print(f"ERROR: No se encuentra {image_path}")
        return

    img = cv2.imread(image_path)
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

    # Patrón del tablero impreso 12x9 cuadros -> (11, 8) esquinas internas
    target_patterns = [(11, 8), (8, 11), (11, 7), (10, 7)]

    # Ecualización adaptativa para corregir sombras y contraluz
    clahe = cv2.createCLAHE(clipLimit=3.0, tileGridSize=(8, 8))
    gray_enhanced = clahe.apply(gray)

    flags = (cv2.CALIB_CB_ADAPTIVE_THRESH + 
             cv2.CALIB_CB_NORMALIZE_IMAGE + 
             cv2.CALIB_CB_FAST_CHECK)

    print("Iniciando detección de esquinas...")

    found = False
    selected_pattern = None
    corners = None

    for pattern in target_patterns:
        print(f"Probando patrón de intersecciones {pattern[0]}x{pattern[1]}...")
        found, corners = cv2.findChessboardCorners(gray_enhanced, pattern, flags)
        
        if not found:
            found, corners = cv2.findChessboardCorners(gray, pattern, flags)

        if found:
            selected_pattern = pattern
            break

    if found:
        print(f"\n -> ¡ÉXITO TOTAL! Patrón detectado: {selected_pattern[0]}x{selected_pattern[1]}")

        # Refinamiento a nivel Sub-Píxel (Sintaxis OpenCV Python)
        criteria = (cv2.TERM_CRITERIA_EPS + cv2.TERM_CRITERIA_MAX_ITER, 30, 0.001)
        corners_subpix = cv2.cornerSubPix(gray, corners, (11, 11), (-1, -1), criteria)

        output_img = img.copy()
        cv2.drawChessboardCorners(output_img, selected_pattern, corners_subpix, found)

        output_path = os.path.join("captures", "frame_corners_detected.png")
        cv2.imwrite(output_path, output_img)
        print(f"[ÉXITO] Imagen anotada guardada en: {output_path}")
    else:
        print("\n -> No se logró detectar la rejilla automáticamente.")

if __name__ == "__main__":
    main()

```

* **Comando para ejecutar:**

```bash
python detect_chessboard.py

```

---

### 3. Captura Interactiva para Calibración (`calibrate_capture.py`)

Abre el streaming y guarda imágenes numeradas en `captures/calib_samples/` cada vez que el usuario presiona `ENTER`.

* **Código del archivo:**

```python
import os
import sys
import time

sys.path.insert(0, "./api")
import gxipy as gx

try:
    import cv2
    HAS_CV2 = True
except ImportError:
    from PIL import Image as PILImage
    HAS_CV2 = False

def set_roi_safe(cam, target_w, target_h, offset_y_shift):
    cam.OffsetX.set(0)
    cam.OffsetY.set(0)
    w_range = cam.Width.get_range()
    h_range = cam.Height.get_range()
    width = (min(max(target_w, w_range['min']), w_range['max']) // w_range['inc']) * w_range['inc']
    height = (min(max(target_h, h_range['min']), h_range['max']) // h_range['inc']) * h_range['inc']
    cam.Width.set(width)
    cam.Height.set(height)
    
    x_range = cam.OffsetX.get_range()
    y_range = cam.OffsetY.get_range()
    offset_x = (min((5496 - width) // 2, x_range['max']) // x_range['inc']) * x_range['inc']
    offset_y = (min(max(offset_y_shift, y_range['min']), y_range['max']) // y_range['inc']) * y_range['inc']
    cam.OffsetX.set(offset_x)
    cam.OffsetY.set(offset_y)

def main():
    save_dir = os.path.join("captures", "calib_samples")
    os.makedirs(save_dir, exist_ok=True)

    device_manager = gx.DeviceManager()
    dev_num, _ = device_manager.update_device_list()
    if dev_num == 0:
        print("No se encontró ninguna cámara.")
        return

    cam = None
    try:
        cam = device_manager.open_device_by_index(1)
        cam.TriggerMode.set(gx.GxSwitchEntry.OFF)
        if hasattr(cam, 'ExposureTime'):
            cam.ExposureTime.set(80000.0)
        if hasattr(cam, 'Gain'):
            cam.Gain.set(10.0)

        set_roi_safe(cam, target_w=2400, target_h=2400, offset_y_shift=200)
        cam.stream_on()

        print("\n" + "="*50)
        print(" SENSOR LISTO PARA CAPTURA DE CALIBRACIÓN")
        print("="*50)
        print("Instrucciones:")
        print("  - Presiona ENTER en la terminal para guardar una muestra.")
        print("  - Mueve/inclina el tablero entre capturas.")
        print("  - Escribe 'q' y presiona ENTER para finalizar.")
        print("="*50 + "\n")

        count = 0
        while True:
            cmd = input(f"Presiona [ENTER] para capturar muestra #{count + 1} (o 'q' para salir): ")
            if cmd.strip().lower() == 'q':
                break

            raw_image = cam.data_stream[0].get_image(timeout=2000)
            if raw_image is not None:
                rgb_img = raw_image.convert("RGB")
                numpy_array = rgb_img.get_numpy_array()

                count += 1
                filename = os.path.join(save_dir, f"sample_{count:02d}.png")
                
                if HAS_CV2:
                    bgr = cv2.cvtColor(numpy_array, cv2.COLOR_RGB2BGR)
                    cv2.imwrite(filename, bgr)
                else:
                    pil_img = PILImage.fromarray(numpy_array)
                    pil_img.save(filename)

                print(f" -> Muestra #{count} guardada en: {filename}")
            else:
                print(" -> ERROR: Timeout al tomar la captura.")

        print(f"\nProceso finalizado. Total de capturas tomadas: {count}")

    except Exception as e:
        print(f"Error: {e}")
    finally:
        if cam is not None:
            cam.stream_off()
            cam.close_device()
            print("Cámara cerrada.")

if __name__ == "__main__":
    main()

```

* **Comando para ejecutar:**

```bash
python calibrate_capture.py

```

---

## 🚀 Próximos Pasos (Mañana)

1. Ejecutar `python calibrate_capture.py` para guardar 10-15 muestras del tablero en distintas posiciones.
2. Crear el script de calibración para calcular la **Matriz Intrínseca ($K$)** y los **Coeficientes de Distorsión ($D$)**.

```

```
