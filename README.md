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

**2. Graba el firmware de ejemplo en la placa.** Cada *device SDK* (`protolink-device-sdk-<placa>-X.Y.Z.zip`, uno
por placa) trae en `firmware/` un ejemplo **ya compilado, listo para grabar**: filtra cada canal, así que lo que envíes
desde el PC vuelve suavizado y comprueba que todo funciona. Para ejecutar tu propio código en la placa, ve al paso 5.

| Placa | Cómo se graba |
|-------|---------------|
| Raspberry Pi Pico | Mantén BOOTSEL al enchufar y arrastra `firmware/protolink_example_pico.uf2` (del SDK de la Pico) a la unidad `RPI-RP2` |
| Raspberry Pi Pico 2 | Igual, con `firmware/protolink_example_pico2.uf2` (del SDK de la Pico 2) |
| ESP32 (WROOM) | `esptool --chip esp32 --port <puerto> write-flash 0x0 firmware/protolink_example_esp32.bin` (del SDK del ESP32) |
| Arduino UNO | `avrdude -p atmega328p -c arduino -P <puerto> -b 115200 -U flash:w:firmware/protolink_example_uno.hex:i` (del SDK del UNO), o el sketch del SDK en el Arduino IDE |
| LGT8F328P | Sin device SDK por ahora (su núcleo de Arduino no admite librerías precompiladas) |

**3. Comprueba el enlace** (puerto: `/dev/ttyACM0` en Linux, `COM5` en Windows, `/dev/cu.usbmodem…` en macOS):

```bash
python examples/hw_check.py /dev/ttyACM0                 # Pico
python examples/hw_check.py /dev/ttyUSB0 --esp32         # ESP32
python examples/hw_suite.py /dev/ttyACM0 --board pico    # batería completa de pruebas
```
Los ejemplos de C y C++ también se ejecutan directamente: `examples/bin/example_c /dev/ttyACM0`.

**4. Úsalo desde tu lenguaje** (más abajo, «Uso»).

**5. Pon tu propio código en la placa** con el [device SDK](docs/DeviceSDK.md) de tu placa: escribes una función,
`pl_app_process(in, n_in, out, n_out)`, y el proyecto de ejemplo (`example/`) la ejecuta en cada trama.

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
Si no sabes la velocidad: `Link(puerto, baud="auto", boot=2.5)` la busca preguntándole a la placa (`link.baud`);
`boot` son los segundos de espera inicial de las placas que se resetean al abrir el puerto (ESP32 ~1, UNO ~2.5).

### Velocidad (baudrate)

| Placa | Conexión | Baud en el PC | Frecuencia máx. |
|-------|----------|---------------|:---------------:|
| Pico / Pico 2 | USB-CDC | no importa | 1000 Hz |
| ESP32 | UART | **921600** | 1000 Hz |
| Arduino UNO | UART | **500000** | 500 Hz |
| LGT8F328P | UART | **1000000** | 1000 Hz |

En un UART hay que abrir el puerto a la misma velocidad que la placa: con otra, todas las tramas fallan el CRC.
La autodetección prueba 115200, 921600, 500000, 1000000, 230400 y 460800 y requiere que la placa conteste a la
consulta de canales (todos los firmwares de protolink lo hacen).

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
link = protolink.Link("COM5");              % o protolink.Link("COM5", "auto", true, 2.5) si no sabes la velocidad
link.start(200);
link.send(single(1:9));
[v, seq, tUs, ok] = link.recvLatest();
```

**Simulink** — tras `protolink_setup`, arrastra el bloque **protolink** de la librería `protolink_lib` a tu modelo.
Placa, puerto (con la lista de los conectados), frecuencia y modo se eligen en desplegables. El paso fijo del
modelo debe ser `1/Rate`. Prueba rápida: `protolink_demo`.

## Tu código en la placa (device SDK)

Hay un device SDK **por placa** (`protolink-device-sdk-pico-X.Y.Z.zip`, `-pico2-`, `-esp32-` y `-uno-`). Trae la lógica del enlace
**ya compilada para esa placa** (una librería, sin código fuente), el **ejemplo ya compilado y listo para grabar**, y
**ese mismo ejemplo como proyecto** para que lo compiles y lo uses de base. Solo escribes tu modelo, controlador o
lectura de sensores, y cuántos canales usa tu placa en cada sentido:

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
| `protolink-device-sdk-pico-X.Y.Z.zip`, `-pico2-`, `-esp32-`, `-uno-` | **Tu código en la placa**: la librería del enlace compilada para esa placa, el ejemplo ya compilado (`.uf2` / `.bin` / `.hex`) y el mismo ejemplo como proyecto (en el UNO, una librería de Arduino y un sketch) |
| `SHA256SUMS.txt` | Comprobación de integridad (`sha256sum -c SHA256SUMS.txt`) |

**Paquetes de MATLAB:** `glnxa64` (Linux), `maca64` (macOS con Apple Silicon) y `win64` (Windows), compilados con
**MATLAB R2024b**: funcionan con esa versión y, en general, con las posteriores. No hay paquete para Mac con
procesador Intel. Con una versión anterior a R2024b, o en un Mac Intel, ejecuta `protolink_setup('rebuild')`
(necesita un compilador de C: `mex -setup C`).

**No incluido:** el device SDK del **LGT8F328P**, porque su núcleo de Arduino (`lgt8fx`) no admite librerías
precompiladas. En el PC funciona igual que el UNO (1000000 baudios, hasta 1000 Hz). Los ejemplos de los SDK están
compilados, pero **ninguno se ha probado todavía en una placa real** (ver «Placas y sistemas probados»).

## Licencia

**protolink** es software propietario con licencia **[PolyForm Noncommercial 1.0.0](LICENSE)**: puedes usarlo,
copiarlo y modificarlo libremente en **proyectos personales, de estudio, de investigación o aficionados** (cualquier
uso no comercial), siempre que **atribuyas su autoría con un enlace a este repositorio**
(<https://github.com/racarla96/ProtoLAB_USB_Communication_Release>) y conserves el aviso `Required Notice` del
fichero [LICENSE](LICENSE). El uso comercial no está incluido: para eso, contacta con el autor. El texto legal
completo (en inglés) está en `LICENSE`; esta explicación es solo un resumen.
