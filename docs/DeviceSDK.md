# Poner tu código en el micro (device SDK)

Los firmwares de ejemplo (`protolink_fw.uf2`, `protolink_fw_esp32.bin`) hacen eco: devuelven lo que reciben.
Para ejecutar **tu** modelo, controlador o lectura de sensores en la placa, usa el **device SDK**
(`protolink-device-sdk.zip` de la release):

```
protolink-device-sdk/
  include/protolink_device.h          toda la API
  lib/{rp2040,rp2350,esp32}/libprotolink_device.a   la lógica del enlace, ya compilada
  templates/pico/                     proyecto de pico-sdk (Pico y Pico 2)
  templates/esp32/                    proyecto de ESP-IDF (ESP32-WROOM / WROVER)
```

## Qué escribes tú

Dos cosas, en `app.c` de la plantilla:

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

La plantilla trae un filtro paso bajo como ejemplo. Lo demás (frecuencia pedida por el PC, confirmaciones,
descarte de tramas sin bloquear nunca) lo hace la librería.

## Raspberry Pi Pico / Pico 2

```bash
export PICO_SDK_PATH=/ruta/al/pico-sdk          # pico-sdk 2.x + gcc-arm-none-eabi
cp -r protolink-device-sdk/templates/pico mi_proyecto   # o trabaja en templates/pico
cmake -S . -B build -G Ninja && cmake --build build     # Pico 2: -DPICO_BOARD=pico2
```
Si copias la plantilla fuera del SDK, indica dónde está con `-DPROTOLINK_SDK=/ruta/protolink-device-sdk`.
Graba `build/my_firmware.uf2` (BOOTSEL, o `stty -F /dev/ttyACM0 1200` en Linux).

## ESP32

```bash
. $IDF_PATH/export.sh                            # ESP-IDF 5.x
cd protolink-device-sdk/templates/esp32          # (o una copia con -DPROTOLINK_SDK=...)
idf.py set-target esp32 && idf.py build && idf.py -p /dev/ttyUSB0 flash
```
En el PC, abre el puerto a 921600 baudios sin tocar DTR/RTS (`--board esp32` en los ejemplos).
No pongas `CONFIG_ESP_CONSOLE_NONE` en tu `sdkconfig`: la placa dejaría de transmitir.

## Desde el PC no cambia nada

El PC usa la misma librería y los mismos ejemplos (Python, C++, MATLAB, Simulink) con tu firmware o con los
de ejemplo; solo cambia lo que devuelve `pl_app_process` y el número de canales.
