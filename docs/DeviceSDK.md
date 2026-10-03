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

Una sola función, en `app.c` de la plantilla, que se llama una vez por trama (100 Hz tras encender;
el PC puede pedir hasta 1000 Hz):

```c
#include "protolink_device.h"

void pl_app_process(const float in[PROTOLINK_N_VALUES], float out[PROTOLINK_N_VALUES]) {
    /* in  = los 9 valores que mandó el PC
       out = los 9 valores que se le devuelven */
}
```
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
de ejemplo; solo cambia lo que devuelve `pl_app_process`.
