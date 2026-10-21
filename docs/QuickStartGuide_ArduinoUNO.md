# Guía rápida — Arduino UNO y LGT8F328P

protolink en un **Arduino UNO (ATmega328P)**. Es el hermano pequeño de la Pico: mismo protocolo y misma API,
pero con **menos frecuencia máxima** porque el UNO tiene 16 MHz de CPU de 8 bits, 2 KB de RAM y un puerto
serie (UART) como único enlace con el PC.

> **Estado:** el device SDK del UNO se construye y compila en CI (4–6 KB de flash, ~500 B de RAM) y la lógica común
> está probada en el PC contra una placa simulada, pero **aún no se ha probado en un UNO real**. Los 500 Hz de máximo
> son una estimación por ancho de banda; la prueba con la placa decide el límite real.

> **LGT8F328P** (clon de ATmega328P a 32 MHz): **no tiene device SDK por ahora**, porque su núcleo de Arduino
> (`lgt8fx`) no admite librerías precompiladas. En el PC funciona igual que el UNO, pero a 1000000 baudios y hasta
> 1000 Hz (`baud=protolink.LGT8F_BAUD`).

## Qué cambia respecto a la Pico

| | Pico | Arduino UNO |
|---|---|---|
| USB | nativo (USB-CDC) | chip puente (ATmega16U2 o CH340) → UART de la placa |
| Frecuencia | 1–1000 Hz | **1–500 Hz** (100 Hz al encender); una petición mayor se rechaza y se mantiene la anterior |
| Velocidad del puerto | da igual | **500000 baudios** (hay que abrirlo así en el PC) |
| Abrir el puerto | no resetea | **resetea la placa** (DTR): esperar ~2 s antes del primer `start` |
| Abrir en Python | `Link("COM5")` | `Link("COM5", baud=protolink.UNO_BAUD)` y `time.sleep(2.5)` |
| `hw_check.py` / `hw_suite.py` | `hw_check.py COM5` | `hw_check.py COM5 --uno` · `hw_suite.py COM5 --board uno` |

Por qué 500 Hz: a 500000 baudios la línea da para ~1000 tramas/s en cada sentido; el límite deja el flujo en la mitad, lo que da margen a la CPU de 8 bits y a los búferes de 64 bytes del UART. Si bajas la velocidad, baja el límite
(115200 → ~120 Hz). Es la constante `PL_HAL_MAX_RATE_HZ` del firmware.

## 1. Descargar

De la página [Releases](https://github.com/racarla96/ProtoLAB_USB_Communication_Release/releases/latest):

- `protolink-device-sdk-uno-X.Y.Z.zip` — el SDK del UNO. Trae el **ejemplo ya compilado** (`firmware/protolink_example_uno.hex`),
  la librería `ProtolinkDevice` ya compilada y el mismo ejemplo como sketch para el Arduino IDE.
- `protolink-X.Y.Z-<tu sistema>` — librería, bindings y ejemplos del PC.

## 2. Grabar

**Con `avrdude`** (viene con el IDE; en Linux `sudo apt install avrdude`), con el `.hex` del SDK:

```bash
avrdude -p atmega328p -c arduino -P /dev/ttyACM0 -b 115200 -U flash:w:firmware/protolink_example_uno.hex:i   # Linux
avrdude -p atmega328p -c arduino -P COM5 -b 115200 -U flash:w:firmware/protolink_example_uno.hex:i            # Windows
avrdude -p atmega328p -c arduino -P /dev/cu.usbmodem1101 -b 115200 -U flash:w:firmware/protolink_example_uno.hex:i  # macOS
```

(clones con CH340: el puerto es `/dev/ttyUSB0` o `COMx`; si falla, prueba `-b 57600` por si lleva el bootloader antiguo.)

**Con el Arduino IDE:** copia la carpeta `library/ProtolinkDevice` del SDK a la carpeta `libraries` de tu sketchbook, abre
`example/ProtolinkExample/ProtolinkExample.ino`, elige *Herramientas → Placa → Arduino UNO* y el puerto, y pulsa *Subir*.

## 3. Probar

Con el UNO conectado y **sin ningún otro programa (monitor serie del IDE) usando el puerto**:

```bash
python examples/hw_check.py /dev/ttyACM0 --uno --rate 500          # eco, frecuencia y pérdidas
python examples/hw_suite.py /dev/ttyACM0 --board uno --unplug      # batería completa
```

Desde tu código:

```python
import time, protolink
with protolink.Link("/dev/ttyACM0", baud=protolink.UNO_BAUD) as link:
    time.sleep(2.5)            # el UNO se resetea al abrir el puerto
    link.start(200)            # 1..500
    link.send([0.0] * 9)
    f = link.recv(timeout=0.1)
```

C/C++: `pl_open(port, PL_UNO_BAUD)`; MATLAB: `protolink.Link("COM5", 500000)` y `pause(2.5)`.

## Notas

- **Reset al abrir**: es el comportamiento normal del UNO. Tiene una ventaja: la frecuencia vuelve a 100 Hz
  cada vez que se abre el puerto. Si no quieres el reset, un condensador de 10 µF entre RESET y GND lo evita
  (pero entonces no podrás volver a subir sketches sin quitarlo).
- **Simulink**: el bloque es el mismo; usa un puerto abierto a 500000 baudios y una frecuencia ≤ 500 Hz
  (con la máscara, la espera de 2,5 s tras el reset aún no está automatizada).
- **Tu código**: sustituye `pl_app_process` en `app.c` del sketch de ejemplo (ahora es un filtro paso bajo). Con 2 KB de RAM y sin
  FPU, cada trama deja ~2 ms de CPU a 500 Hz; cálculos pesados en `float` bajan la frecuencia alcanzable.
