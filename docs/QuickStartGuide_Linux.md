# Guía rápida — Linux

Poner en marcha protolink con una Raspberry Pi Pico real en Linux (probado en CI con Ubuntu 22.04;
la librería publicada funciona en cualquier distribución x86_64 con glibc ≥ 2.17).

Guías para otros sistemas: [Windows](QuickStartGuide_Windows.md) · [macOS](QuickStartGuide_macOS.md)

## 0. Qué necesitas

- Raspberry Pi Pico (RP2040) o Pico 2 (RP2350) y un cable USB **de datos** (no solo de carga).
- Python ≥ 3.8 para la prueba rápida; `gcc`/`g++` si vas a usar C/C++.

## 1. Descargar

De la página [Releases](https://github.com/racarla96/ProtoLAB_USB_Communication_Release/releases/latest):

| Fichero | Para qué |
|---------|----------|
| `protolink_fw.uf2` (Pico) o `protolink_fw_pico2.uf2` (Pico 2) | firmware de la placa |
| `protolink-X.Y.Z-linux-x86_64.tar.gz` | librería C/C++, bindings, ejemplos |
| `protolink-X.Y.Z-py3-none-manylinux…_x86_64.whl` | paquete de Python |

Comprueba la descarga con `sha256sum -c SHA256SUMS.txt --ignore-missing`.

```bash
tar xzf protolink-X.Y.Z-linux-x86_64.tar.gz
cd protolink-X.Y.Z-linux-x86_64
python3 -m venv ~/.venvs/protolink && . ~/.venvs/protolink/bin/activate
pip install ../protolink-*-py3-none-manylinux*_x86_64.whl
```


## 2. Flashear el firmware

1. Con la placa desconectada, mantén pulsado **BOOTSEL** y conéctala. Suelta el botón.
2. Aparece una unidad `RPI-RP2` (Pico) o `RP2350` (Pico 2). Copia el `.uf2`:
   ```bash
   cp protolink_fw.uf2 /media/$USER/RPI-RP2/
   ```
3. La placa se reinicia sola y aparece como puerto serie:
   ```bash
   lsusb | grep 2e8a          # ID 2e8a:000a Raspberry Pi
   sudo dmesg | tail          # cdc_acm ...: ttyACM0: USB ACM device
   ls -l /dev/ttyACM*
   ```

## 3. Permisos y ModemManager (una sola vez)

Sin esto el puerto puede no abrirse por permisos, o ModemManager puede ocuparlo unos segundos al
conectar la placa. Esta regla udev resuelve ambas cosas para cualquier placa Raspberry Pi (VID `2e8a`):

```bash
echo 'SUBSYSTEM=="tty", ATTRS{idVendor}=="2e8a", MODE="0666", ENV{ID_MM_DEVICE_IGNORE}="1"' \
  | sudo tee /etc/udev/rules.d/99-protolink.rules
sudo udevadm control --reload && sudo udevadm trigger
```
Desconecta y vuelve a conectar la placa. (Alternativa solo para permisos:
`sudo usermod -aG dialout $USER` y volver a iniciar sesión.)

## 4. Probar

**Prueba automática** (eco, 100 Hz y pérdidas; ~10 s):
```bash
python examples/hw_check.py /dev/ttyACM0
```
Resultado esperado:
```
[OK]   echo matches after 12.3 ms
[OK]   received 1000 frames in 10 s (~100.0 Hz)
[OK]   lost=0 crc=0 framing=0  (tx total=1000)
hw_check: all good
```
El firmware de referencia devuelve el **último vector recibido** (eco). Sustituye `process()` en
`firmware/main.c` por tu modelo.

**Otra frecuencia** (firmware con protocolo v2, entero de 1 a 1000 Hz; tras encender son 100 Hz):
```
python examples/hw_check.py /dev/ttyACM0 --rate 1000
```
En tu código: `link.start(1000)` (Python, C++, MATLAB) o `pl_start(link, 1000, 1000)` (C). La Pico
mantiene esa frecuencia con su propio reloj aunque el PC se bloquee; la librería sigue recibiendo en
segundo plano (cola de ~65 s a 1 kHz) y `link.pending()` dice cuántas tramas esperan.

**C / C++** desde la carpeta del release:
```bash
gcc examples/example.c -Iinclude -Llib -lprotolink -lm -Wl,-rpath,"$PWD/lib" -o example_c
g++ -std=c++17 examples/example.cpp -Iinclude -Ibindings/cpp -Llib -lprotolink -Wl,-rpath,"$PWD/lib" -o example_cpp
./example_c /dev/ttyACM0 5          # imprime seq, t, v0, v8 de cada trama
./example_cpp /dev/ttyACM0 5        # y al final las estadísticas
```
Enlazado estático (sin `.so` en tiempo de ejecución):
`gcc examples/example.c -DPL_STATIC -Iinclude lib/libprotolink_static.a -lm -o example_c`.

**MATLAB** (necesita un compilador C soportado por `loadlibrary`, normalmente `gcc`):
```matlab
addpath('bindings/matlab')            % la carpeta del release
link = protolink.Link("/dev/ttyACM0");
link.send(single(1:9)); pause(0.05);
[v, seq, tUs, ok] = link.recvLatest()
delete(link)
```
Ejemplo completo: `examples/example_matlab.m`.

## 5. Problemas frecuentes

| Síntoma | Causa probable / solución |
|---------|---------------------------|
| No aparece `RPI-RP2` | Cable solo de carga, o no se mantuvo BOOTSEL al conectar |
| No aparece `/dev/ttyACM0` | `.uf2` de otra placa (Pico vs Pico 2); mira `sudo dmesg` |
| `could not open or configure the port` | Permisos (paso 3), o el puerto está abierto por otro programa: `sudo lsof /dev/ttyACM0` |
| Se abre pero no llega nada (`[FAIL] the echo never matched`) | Firmware no flasheado o placa colgada: vuelve a conectarla; envía la salida de `dmesg` |
| Las primeras tramas dan `lost`/`framing` | Normal si se abrió a mitad de trama; `hw_check` ya las descuenta |
| Tasa claramente < 100 Hz o `lost` creciendo | Hub USB saturado o PC muy cargado; prueba otro puerto USB y envía la salida de `hw_check` |
| `OSError: could not load the protolink shared library` | El paquete Python no tiene la librería: instala la wheel, o define `PROTOLINK_LIB=/ruta/libprotolink.so` |
| Al desconectar la placa | Las llamadas devuelven `PL_ERR_IO` (`i/o error or device disconnected`); hay que cerrar y volver a abrir |
