# Poner tu código en el micro (device SDK)

Para ejecutar **tu** modelo, controlador o lectura de sensores en la placa, usa el **device SDK de tu placa**: un zip
por placa en cada release, con la lógica del enlace **ya compilada** (sin código fuente) y un ejemplo de uso:

| Zip | Placa |
|-----|-------|
| `protolink-device-sdk-pico-X.Y.Z.zip` | Raspberry Pi Pico (RP2040) |
| `protolink-device-sdk-pico2-X.Y.Z.zip` | Raspberry Pi Pico 2 (RP2350) |
| `protolink-device-sdk-esp32-X.Y.Z.zip` | ESP32 (WROOM / WROVER) |
| `protolink-device-sdk-uno-X.Y.Z.zip` | Arduino UNO (ATmega328P) |

```
protolink-device-sdk-<placa>-X.Y.Z/
  include/protolink_device.h          toda la API
  lib/libprotolink_device.a           la lógica del enlace, ya compilada para esta placa
  firmware/protolink_example_<placa>.{uf2,bin}   el ejemplo YA COMPILADO, listo para grabar
  example/                            el MISMO ejemplo como proyecto, para compilarlo y usarlo de base
  README.md                           los pasos de esta placa
```

El del **Arduino UNO** tiene otra forma, porque Arduino trabaja con librerías: en vez de `include/` y `lib/`, trae
`library/ProtolinkDevice/` (librería de Arduino ya compilada, que copias a la carpeta `libraries` de tu sketchbook) y el
ejemplo es un sketch (`example/ProtolinkExample/`, con tu código en `app.c`); el ejemplo compilado es un `.hex`.
(El LGT8F328P no tiene device SDK: su núcleo de Arduino no admite librerías precompiladas.)

**Pruébalo sin compilar nada:** graba el fichero de `firmware/` y conecta desde el PC (Python, C++, MATLAB o
Simulink, o `examples/hw_check.py` de la release de la librería). **Y cuando quieras tu código:** copia `example/`,
cambia `pl_app_process()` en `app.c` y compila.

## Qué escribes tú

Dos cosas, en `app.c` del ejemplo:

```c
#include "protolink_device.h"

/* Cuántos valores recibe y cuántos manda la placa en cada trama (los fija tu firmware). */
const unsigned protolink_channels_from_pc = 3;   /* el PC manda 3 valores            */
const unsigned protolink_channels_to_pc = 12;    /* la placa le devuelve 12 valores  */

/* Se llama una vez por trama (100 Hz tras encender; el PC puede pedir hasta 1000 Hz). */
void pl_app_process(const float *in, int n_in, float *out, int n_out) {
    /* in  = los n_in valores que mandó el PC (los que no mandó valen 0)
       out = los n_out valores que se le devuelven */
}
```

**Canales.** El número de canales de cada sentido lo decide la placa, hasta 64 por sentido. El PC los lee
al conectar (`link.channels`, `link.query()`), así que Python, C++, MATLAB y Simulink se adaptan sin tocar nada;
en Simulink, el botón *Read from board* de la máscara los copia al bloque. Cada canal cuesta 4 bytes por trama:
con muchos canales a alta frecuencia hace falta un enlace rápido. En la Pico (USB) no es un problema; en las
placas con UART (ESP32 a 921600 baudios, Arduino) la placa calcula el máximo que cabe (el 90 % del enlace) y lo
anuncia al PC: p. ej. con 20 canales el ESP32 admite hasta 901 Hz y rechaza más con un mensaje claro.

El ejemplo trae un filtro paso bajo. Lo demás (frecuencia pedida por el PC, confirmaciones,
descarte de tramas sin bloquear nunca) lo hace la librería.

## Raspberry Pi Pico / Pico 2

```bash
export PICO_SDK_PATH=/ruta/al/pico-sdk          # pico-sdk 2.x + gcc-arm-none-eabi
cp -r protolink-device-sdk-pico-X.Y.Z/example mi_proyecto
cd mi_proyecto
cmake -S . -B build -G Ninja -DPICO_BOARD=pico   # con el SDK de la Pico 2: -DPICO_BOARD=pico2
cmake --build build
```
Si copias `example/` fuera del SDK, indica dónde está con `-DPROTOLINK_SDK=/ruta/protolink-device-sdk-pico-X.Y.Z`.
Usa el SDK de **tu** placa: la librería de la Pico (RP2040) y la de la Pico 2 (RP2350) son distintas.
Graba `build/my_firmware.uf2` (BOOTSEL, o `stty -F /dev/ttyACM0 1200` en Linux).

## ESP32

```bash
. $IDF_PATH/export.sh                            # ESP-IDF 5.x
cp -r protolink-device-sdk-esp32-X.Y.Z/example mi_proyecto    # (con -DPROTOLINK_SDK=... si lo sacas del SDK)
cd mi_proyecto
idf.py set-target esp32 && idf.py build && idf.py -p /dev/ttyUSB0 flash
```
El ejemplo ya compilado (`firmware/protolink_example_esp32.bin`) es una imagen única para grabar en `0x0`:
`esptool --chip esp32 --port <puerto> write-flash 0x0 firmware/protolink_example_esp32.bin`.
En el PC, abre el puerto a 921600 baudios sin tocar DTR/RTS (`--board esp32` en los ejemplos).
No pongas `CONFIG_ESP_CONSOLE_NONE` en tu `sdkconfig`: la placa dejaría de transmitir.

## Arduino UNO

```bash
# Arduino IDE: copia library/ProtolinkDevice a la carpeta "libraries" de tu sketchbook y abre
# example/ProtolinkExample/ProtolinkExample.ino (placa Arduino UNO).  O, con arduino-cli:
arduino-cli compile --fqbn arduino:avr:uno --library protolink-device-sdk-uno-X.Y.Z/library/ProtolinkDevice \
    protolink-device-sdk-uno-X.Y.Z/example/ProtolinkExample
```
El ejemplo ya compilado (`firmware/protolink_example_uno.hex`) se graba con
`avrdude -p atmega328p -c arduino -P <puerto> -b 115200 -U flash:w:firmware/protolink_example_uno.hex:i`.
Con 2 KB de RAM el máximo es 16 canales por sentido (`PROTOLINK_MAX_CHANNELS`), y el enlace va por UART a 500000
baudios, hasta 500 Hz. En el PC abre el puerto a esa velocidad y espera ~2 s tras abrirlo, que resetea la placa
(`Link(puerto, baud=protolink.UNO_BAUD)` o `baud="auto", boot=2.5`).

## Desde el PC no cambia nada

El PC usa la misma librería y los mismos ejemplos (Python, C++, MATLAB, Simulink) con tu firmware o con los
de ejemplo; solo cambia lo que devuelve `pl_app_process` y el número de canales.
