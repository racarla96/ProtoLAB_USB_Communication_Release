# protolink

Enlace de **9 valores `float32` por USB** entre un PC (Windows, Linux, macOS) y una Raspberry Pi Pico, ESP32,
Arduino UNO o LGT8F328P: 100 Hz por defecto y configurable de 1 a 1000 Hz (entero) con `start`. Librería en C,
con bindings para C++, Python, MATLAB y Simulink.

Descargas en **[Releases](https://github.com/racarla96/ProtoLAB_USB_Communication_Release/releases/latest)**.

## Qué hay en cada release

| Fichero | Contenido |
|---------|-----------|
| `protolink-X.Y.Z-linux-x86_64.tar.gz`, `-macos-universal.tar.gz`, `-windows-x64.zip` | Librería (C/C++), bindings de C++, Python, MATLAB y Simulink, ejemplos y guías |
| `protolink-X.Y.Z-py3-none-<plataforma>.whl` | Paquete de Python (`pip install` y listo) |
| `protolink-matlab-X.Y.Z-<sistema>-<release de MATLAB>.zip` | **MATLAB + Simulink listos**: bloque ya compilado, librería con máscara de desplegables, `Link.m`. Descomprimir y ejecutar `protolink_setup` |
| `protolink_fw.uf2` | Firmware para Raspberry Pi Pico (arrastrar a la unidad RPI-RP2) |
| `SHA256SUMS.txt` | Comprobación de integridad (`sha256sum -c SHA256SUMS.txt`) |

Los firmwares de Pico 2, ESP32, Arduino UNO y LGT8F328P se publicarán cuando se prueben en hardware.

## Guías

En [`docs/`](docs/): guías rápidas
[Linux](docs/QuickStartGuide_Linux.md) · [Windows](docs/QuickStartGuide_Windows.md) ·
[macOS](docs/QuickStartGuide_macOS.md) · [ESP32](docs/QuickStartGuide_ESP32.md) ·
[Arduino UNO / LGT8F328P](docs/QuickStartGuide_ArduinoUNO.md), y [Simulink](docs/SIMULINK.md)
(bloque con máscara de desplegables, instalación y parámetros).

## Uso

**Python**
```python
import protolink
with protolink.Link("COM5") as link:        # /dev/ttyACM0 en Linux
    link.start(500)                         # tramas/s, 1..1000 (100 por defecto)
    link.send([0.0] * 9)
    f = link.recv(timeout=0.1)              # Frame(values, seq, timestamp_us) o None
```

**C++**
```cpp
protolink::Link link("/dev/ttyACM0");
link.start(500);
link.send({0,1,2,3,4,5,6,7,8});
if (auto f = link.recv_latest()) use(f->values, f->seq, f->timestamp_us);
```

**MATLAB**
```matlab
link = protolink.Link("COM5");
link.start(200);
link.send(single(1:9));
[v, seq, tUs, ok] = link.recvLatest();
```

**Simulink**: arrastra el bloque **protolink** de la librería `protolink_lib` (tras `protolink_setup`); placa,
puerto (con la lista de los conectados), frecuencia y modo se eligen en desplegables.

## Compatibilidad: placas × sistema, por lenguaje

Leyenda: ✅ probado con la placa real · 🧪 probado en CI contra una placa simulada (sin placa real) ·
🔨 compila / se instala, sin ejecutarlo con una placa · ❔ sin probar (no hay hardware o aún no se ha intentado).

Placas: **Pico** = Raspberry Pi Pico (RP2040), **Pico 2** = RP2350, **ESP32** = ESP32-WROOM (puente CP210x/CH340),
**UNO** = Arduino UNO (≤ 500 Hz), **LGT8F328P** = clon a 32 MHz (≤ 1000 Hz).

### C / C++

| Placa | Linux | Windows | macOS |
|-------|:-----:|:-------:|:-----:|
| Pico      | ✅ | 🔨 | 🧪 |
| Pico 2    | ❔ | ❔ | ❔ |
| ESP32     | ❔ | ❔ | ❔ |
| UNO       | ❔ | ❔ | ❔ |
| LGT8F328P | ❔ | ❔ | ❔ |

### Python

| Placa | Linux | Windows | macOS |
|-------|:-----:|:-------:|:-----:|
| Pico      | ✅ | 🔨 | 🧪 |
| Pico 2    | ❔ | ❔ | ❔ |
| ESP32     | ❔ | ❔ | ❔ |
| UNO       | ❔ | ❔ | ❔ |
| LGT8F328P | ❔ | ❔ | ❔ |

### MATLAB (`protolink.Link`)

| Placa | Linux | Windows | macOS |
|-------|:-----:|:-------:|:-----:|
| Pico      | ✅ | ❔ | ❔ |
| Pico 2    | ❔ | ❔ | ❔ |
| ESP32     | ❔ | ❔ | ❔ |
| UNO       | ❔ | ❔ | ❔ |
| LGT8F328P | ❔ | ❔ | ❔ |

### Simulink (bloque `protolink`)

| Placa | Linux | Windows | macOS |
|-------|:-----:|:-------:|:-----:|
| Pico      | ✅ | 🔨 | 🔨 |
| Pico 2    | ❔ | ❔ | ❔ |
| ESP32     | ❔ | ❔ | ❔ |
| UNO       | ❔ | ❔ | ❔ |
| LGT8F328P | ❔ | ❔ | ❔ |

Notas: en Windows el puerto serie (E/S solapada) y en macOS la velocidad de puerto solo se han probado hasta
donde llega el CI; el bloque de Simulink se compila para los tres sistemas en CI con MATLAB, pero solo se ha
ejecutado en Linux. El firmware de Pico 2, ESP32, UNO y LGT8F328P compila en CI; su lógica común está probada
en el PC contra una placa simulada, pero ninguna de esas placas se ha probado todavía.
