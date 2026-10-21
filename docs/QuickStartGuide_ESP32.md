# Guía rápida — ESP32 (ESP32-WROOM / DevKitC)

protolink en un **ESP32 clásico** (ESP32-WROOM-32 / WROVER en placas tipo DevKitC, NodeMCU-32S…),
cuyo USB es un chip puente **CP210x o CH340** conectado al UART0. Todo viene precompilado: no hace falta
instalar ESP-IDF ni compilar nada. Los pasos son para **Windows**; en Linux/macOS cambia solo el nombre del
puerto (ver al final).

> **Estado:** probado en un ESP32-D0WD-V3 (placa con CH340) en Linux: Python, C/C++, MATLAB y Simulink, de 100 a
> 1000 Hz sin pérdidas. Windows y macOS: sin probar con una placa.

## Diferencias con la Pico

| | Pico | ESP32-WROOM |
|---|---|---|
| USB | nativo (USB-CDC) | chip puente CP210x / CH340 → UART0 |
| Velocidad del puerto | da igual | **921600 baudios** (en el PC hay que abrirlo así) |
| DTR / RTS | la Pico necesita DTR | **no tocarlos**: en estas placas resetean el ESP32 |
| Abrir en Python | `Link("COM5")` | `Link("COM7", baud=protolink.ESP32_BAUD, dtr_rts=False)` |
| `hw_check.py` | `hw_check.py COM5` | `hw_check.py COM7 --esp32` |

El protocolo, la frecuencia (1–1000 Hz con `start`) y la API son los mismos.

## 1. Descargar

De la página [Releases](https://github.com/racarla96/ProtoLAB_USB_Communication_Release/releases/latest):

- `protolink_fw_esp32.bin` — imagen completa, se graba en la dirección `0x0`
- `protolink-X.Y.Z-windows-x64.zip` — librería, bindings y ejemplos para Windows x64

## 2. Driver y puerto COM

Conecta la placa y mira en el *Administrador de dispositivos* → **Puertos (COM y LPT)**:

- *Silicon Labs CP210x USB to UART Bridge (COM7)* → CP210x. Windows 10/11 suele instalar el driver solo;
  si no, el *CP210x Universal Windows Driver* de Silicon Labs.
- *USB-SERIAL CH340 (COM7)* → CH340; si no aparece, instala el driver **CH341SER** de WCH.

Si no aparece ningún puerto nuevo: prueba otro cable (muchos son solo de carga).

## 3. Grabar el firmware (sin instalar nada)

Con **Chrome o Edge**, abre <https://espressif.github.io/esptool-js/>:

1. **Connect** → elige el COM de la placa.
2. En *Flash Address* pon **`0x0`** y elige `protolink_fw_esp32.bin`.
3. **Program**. Al terminar, pulsa el botón **EN/RST** de la placa (o desconéctala y vuelve a conectarla).

Si no consigue conectar: mantén pulsado **BOOT**, pulsa y suelta **EN**, suelta **BOOT** y repite *Connect*.

<details><summary>Alternativa con esptool en la terminal</summary>

```powershell
py -m pip install esptool
py -m esptool --chip esp32 --port COM7 write_flash 0x0 protolink_fw_esp32.bin
```
</details>

## 4. Probar

```powershell
# descomprime protolink-dev-windows.zip y entra en la carpeta
py -m venv .venv
.venv\Scripts\python.exe -m pip install .\bindings\python
.venv\Scripts\python.exe examples\hw_check.py COM7 --esp32
.venv\Scripts\python.exe examples\hw_check.py COM7 --esp32 --rate 500
.venv\Scripts\python.exe examples\hw_check.py COM7 --esp32 --rate 1000
```
Resultado esperado: eco correcto, la frecuencia pedida (±5 %) y `lost=0 crc=0 framing=0 overflow=0`.

A 921600 baudios caben de sobra 1000 tramas/s en cada sentido, pero
el chip puente añade latencia; si a 1 kHz aparecen pérdidas y a 500 Hz no, ese es el límite de tu placa.

## 5. En tu código

```python
import protolink
with protolink.Link("COM7", baud=protolink.ESP32_BAUD, dtr_rts=False) as link:
    link.start(500)
    link.send([0.0] * 9)
    f = link.recv(timeout=0.1)
```
- **C**: `pl_open_ex("COM7", PL_ESP32_BAUD, PL_OPEN_NO_DTR_RTS)`.
- **C++**: `protolink::Link link("COM7", PL_ESP32_BAUD, PL_OPEN_NO_DTR_RTS);`
- **MATLAB**: `link = protolink.Link("COM7", 921600, false);`
- **Simulink** (`sfun_protolink`): los parámetros 7 y 8 (obligatorios) son velocidad y DTR/RTS: `'COM7', 500, 1/500, 1, 100, 1, 921600, 0`, o `protolink_demo('COM7', 500, 10, true)`.

El código de tu modelo va en `firmware/common/app.c` (`pl_app_process`), el mismo fichero para Pico y
ESP32. Para compilar tú el firmware: ESP-IDF 5.x y `firmware/esp32/build.sh`.

## Problemas frecuentes

| Síntoma | Causa probable / solución |
|---------|---------------------------|
| `[FAIL] the echo never matched` | Falta `--esp32` (velocidad o DTR/RTS), firmware sin grabar, o la placa sigue en modo descarga: pulsa EN |
| La placa se reinicia al abrir el puerto | Se abrió tocando DTR/RTS: usa `dtr_rts=False` / `--esp32`. Otros programas (monitor serie de Arduino, PuTTY) sí los tocan |
| `could not open or configure the port` | COM equivocado o abierto en otro programa (solo uno a la vez en Windows) |
| Algunos `framing`/`crc` justo al conectar | Mensajes de arranque de la ROM del ESP32 tras un reset; el receptor se resincroniza solo |
| Pérdidas solo a 1 kHz | Latencia del chip puente; usa 500 Hz o una placa con USB nativo (ESP32-S3) |

**Linux / macOS:** el puerto es `/dev/ttyUSB0` (Linux; añade tu usuario al grupo `dialout`) o
`/dev/cu.usbserial-*` / `/dev/cu.wchusbserial*` (macOS); el resto es igual.
