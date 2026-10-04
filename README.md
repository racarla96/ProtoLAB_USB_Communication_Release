# ProtoLink

Enlace USB de datos en tiempo real entre un **PC** (Windows, Linux o macOS) y una placa **Raspberry Pi Pico,
ESP32, Arduino UNO o LGT8F328P**: el PC y la placa se intercambian valores `float32` (9 en cada sentido en el firmware de pruebas, hasta 64 por sentido,
los fija la placa), a 100 Hz por defecto y hasta 1000 Hz. Sirve para hacer **hardware-in-the-loop**, control o adquisición desde **Python,
C/C++, MATLAB o Simulink**, con el código del modelo ejecutándose en la placa o en el PC.

```
 tu programa (Python / C / C++ / MATLAB / Simulink)  <──USB──>  placa con el firmware de protolink
         librería protolink                                      (tu código en pl_app_process)
```

## Empezar en 5 pasos

**1. Descarga** de [Releases](https://github.com/racarla96/ProtoLAB_USB_Communication_Release/releases/latest)
el archivo de tu sistema: `protolink-X.Y.Z-linux-x86_64.tar.gz`, `-macos-universal.tar.gz` o `-windows-x64.zip`.
Descomprímelo: trae la librería, los ejemplos (los de C y C++ ya compilados) y las guías.

**2. Graba el firmware de pruebas en la placa.** Los firmwares publicados (`protolink_fw.uf2`,
`protolink_fw_esp32.bin`, los `.hex` de Arduino) **son los de las pruebas**: devuelven al PC lo que reciben
(eco) para comprobar que todo funciona. Para ejecutar tu propio código en la placa, ve al paso 5.

| Placa | Cómo se graba |
|-------|---------------|
| Raspberry Pi Pico / Pico 2 | Mantén BOOTSEL al enchufar y arrastra `protolink_fw.uf2` a la unidad `RPI-RP2` |
| ESP32 (WROOM) | `esptool --chip esp32 --port <puerto> write-flash 0x0 protolink_fw_esp32.bin` |
| Arduino UNO / LGT8F328P | `avrdude -p atmega328p -c arduino -P <puerto> -b 115200 -U flash:w:protolink_fw_uno.hex:i` (o el sketch en el Arduino IDE) |

**3. Comprueba el enlace** (puerto: `/dev/ttyACM0` en Linux, `COM5` en Windows, `/dev/cu.usbmodem…` en macOS):

```bash
python examples/hw_check.py /dev/ttyACM0                 # Pico
python examples/hw_check.py /dev/ttyUSB0 --esp32         # ESP32
python examples/hw_suite.py /dev/ttyACM0 --board pico    # batería completa de pruebas
```
Los ejemplos de C y C++ también se ejecutan directamente: `examples/bin/example_c /dev/ttyACM0`.

**4. Úsalo desde tu lenguaje** (más abajo, «Uso»).

**5. Pon tu propio código en la placa** con el [device SDK](docs/DeviceSDK.md): escribes una función,
`pl_app_process(in, n_in, out, n_out)`, y la plantilla de la Pico o del ESP32 la ejecuta en cada trama.

## Uso

**Python** — el paquete se instala desde el `.whl` de la release (no está en PyPI: ese nombre pertenece a otro proyecto):
```bash
pip install protolink-*.whl            # o, con uv: uv add --find-links <carpeta con el .whl> protolink
```
```python
import protolink
with protolink.Link("/dev/ttyACM0") as link:     # COM5 en Windows
    link.start(500)                              # tramas por segundo, 1..1000 (100 tras encender)
    to_board, from_board = link.channels         # los fija la placa (9 y 9 en el firmware de pruebas)
    link.send([0.0] * to_board)
    f = link.recv(timeout=0.1)                   # Frame(values, seq, timestamp_us) o None
```
Otras placas: `Link(puerto, baud=protolink.ESP32_BAUD, dtr_rts=False)` (ESP32), `baud=protolink.UNO_BAUD` (UNO).

**C++** — con CMake, apuntando a la carpeta descomprimida ([ejemplo completo](examples/cpp_cmake)):
```cmake
find_package(protolink REQUIRED)               # cmake -DCMAKE_PREFIX_PATH=/ruta/protolink-X.Y.Z-...
target_link_libraries(app PRIVATE protolink::protolink)
```
```cpp
protolink::Link link("/dev/ttyACM0");
link.start(500);
link.send({0,1,2,3,4,5,6,7,8});            // un vector: tantos valores como canales tenga la placa
if (auto f = link.recv_latest()) use(f->values, f->seq, f->timestamp_us);
```

**C** — `#include <protolink/protolink.h>` y enlazar con `lib/libprotolink` (`pl_open`, `pl_start`, `pl_send`, `pl_recv`).

**MATLAB** — descomprime `protolink-matlab-X.Y.Z-<sistema>-<release>.zip`, ejecuta `protolink_setup` una vez y:
```matlab
link = protolink.Link("COM5");
link.start(200);
link.send(single(1:9));
[v, seq, tUs, ok] = link.recvLatest();
```

**Simulink** — tras `protolink_setup`, arrastra el bloque **protolink** de la librería `protolink_lib` a tu modelo.
Placa, puerto (con la lista de los conectados), frecuencia y modo se eligen en desplegables. El paso fijo del
modelo debe ser `1/Rate`. Prueba rápida: `protolink_demo`.

## Tu código en la placa (device SDK)

`protolink-device-sdk.zip` trae las plantillas de **Raspberry Pi Pico / Pico 2** y **ESP32** con la lógica del
enlace ya compilada. Solo escribes tu modelo, controlador o lectura de sensores, y cuántos canales usa tu placa en cada sentido:

```c
void pl_app_process(const float *in, int n_in, float *out, int n_out) { /* in: lo que manda el PC, out: lo que devuelves */ }
```
Guía: [docs/DeviceSDK.md](docs/DeviceSDK.md).

## Guías

[Linux](docs/QuickStartGuide_Linux.md) · [Windows](docs/QuickStartGuide_Windows.md) ·
[macOS](docs/QuickStartGuide_macOS.md) · [ESP32](docs/QuickStartGuide_ESP32.md) ·
[Arduino UNO / LGT8F328P](docs/QuickStartGuide_ArduinoUNO.md) · [Tu código en la placa](docs/DeviceSDK.md) ·
[Simulink](docs/SIMULINK.md)

## Placas y sistemas probados

Leyenda: ✅ probado con la placa real · 🧪 probado en CI contra una placa simulada (sin placa real) ·
🔨 compila / se instala, sin ejecutarlo con una placa · ❔ sin probar (no hay hardware o aún no se ha intentado).

Placas: **Pico** = Raspberry Pi Pico (RP2040), **Pico 2** = RP2350, **ESP32** = ESP32-WROOM (puente CP210x/CH340),
**UNO** = Arduino UNO (≤ 500 Hz), **LGT8F328P** = clon a 32 MHz (≤ 1000 Hz).

### C / C++

| Placa | Linux | Windows | macOS |
|-------|:-----:|:-------:|:-----:|
| Pico      | ✅ | 🔨 | 🧪 |
| Pico 2    | ❔ | ❔ | ❔ |
| ESP32     | ✅ | ❔ | ❔ |
| UNO       | ❔ | ❔ | ❔ |
| LGT8F328P | ❔ | ❔ | ❔ |

### Python

| Placa | Linux | Windows | macOS |
|-------|:-----:|:-------:|:-----:|
| Pico      | ✅ | 🔨 | 🧪 |
| Pico 2    | ❔ | ❔ | ❔ |
| ESP32     | ✅ | ❔ | ❔ |
| UNO       | ❔ | ❔ | ❔ |
| LGT8F328P | ❔ | ❔ | ❔ |

### MATLAB (`protolink.Link`)

| Placa | Linux | Windows | macOS |
|-------|:-----:|:-------:|:-----:|
| Pico      | ✅ | ❔ | ❔ |
| Pico 2    | ❔ | ❔ | ❔ |
| ESP32     | ✅ | ❔ | ❔ |
| UNO       | ❔ | ❔ | ❔ |
| LGT8F328P | ❔ | ❔ | ❔ |

### Simulink (bloque `protolink`)

| Placa | Linux | Windows | macOS |
|-------|:-----:|:-------:|:-----:|
| Pico      | ✅ | 🔨 | 🔨 |
| Pico 2    | ❔ | ❔ | ❔ |
| ESP32     | ✅ | ❔ | ❔ |
| UNO       | ❔ | ❔ | ❔ |
| LGT8F328P | ❔ | ❔ | ❔ |

Notas: en Windows el puerto serie (E/S solapada) y en macOS la velocidad de puerto solo se han probado hasta
donde llega el CI; el bloque de Simulink se compila para los tres sistemas en CI con MATLAB, pero solo se ha
ejecutado en Linux. El firmware de Pico 2, UNO y LGT8F328P compila en CI; su lógica común está probada
en el PC contra una placa simulada, pero esas placas no se han probado todavía.

## Qué hay en cada release

| Fichero | Contenido |
|---------|-----------|
| `protolink-X.Y.Z-linux-x86_64.tar.gz`, `-macos-universal.tar.gz`, `-windows-x64.zip` | Librería (C/C++), bindings de C++, Python, MATLAB y Simulink, ejemplos (los de C y C++ compilados) y guías |
| `protolink-X.Y.Z-py3-none-<plataforma>.whl` | Paquete de Python (`pip install` y listo) |
| `protolink-matlab-X.Y.Z-<sistema>-<release de MATLAB>.zip` | **MATLAB + Simulink listos**: bloque ya compilado, librería con máscara de desplegables, `Link.m`. Descomprimir y ejecutar `protolink_setup` |
| `protolink-device-sdk.zip` | **Tu código en la placa**: librería precompilada (RP2040, RP2350, ESP32) y plantillas de Pico y ESP32 |
| `protolink_fw.uf2` | Firmware **de pruebas** (eco) para Raspberry Pi Pico |
| `protolink_fw_esp32.bin` | Firmware **de pruebas** (eco) para ESP32 clásico (se graba en `0x0`) |
| `SHA256SUMS.txt` | Comprobación de integridad (`sha256sum -c SHA256SUMS.txt`) |

Los firmwares de Pico 2, Arduino UNO y LGT8F328P se publicarán cuando se prueben en hardware.

## Licencia

**protolink** es software propietario con licencia **[PolyForm Noncommercial 1.0.0](LICENSE)**: puedes usarlo,
copiarlo y modificarlo libremente en **proyectos personales, de estudio, de investigación o aficionados** (cualquier
uso no comercial), siempre que **atribuyas su autoría con un enlace a este repositorio**
(<https://github.com/racarla96/ProtoLAB_USB_Communication_Release>) y conserves el aviso `Required Notice` del
fichero [LICENSE](LICENSE). El uso comercial no está incluido: para eso, contacta con el autor. El texto legal
completo (en inglés) está en `LICENSE`; esta explicación es solo un resumen.
