# Guía rápida — macOS

Poner en marcha protolink con una Raspberry Pi Pico real en macOS 11 o posterior (Apple Silicon e
Intel: la librería publicada es universal).

Guías para otros sistemas: [Linux](QuickStartGuide_Linux.md) · [Windows](QuickStartGuide_Windows.md)

## 0. Qué necesitas

- Raspberry Pi Pico (RP2040) o Pico 2 (RP2350) y un cable USB **de datos** (no solo de carga).
- Python ≥ 3.8 para la prueba rápida (de [python.org](https://www.python.org/downloads/macos/) o `brew install python`).
- Para C/C++ (y para MATLAB): las *Command Line Tools* de Xcode: `xcode-select --install`.
- No hace falta instalar drivers.

## 1. Descargar

De la página [Releases](https://github.com/racarla96/ProtoLAB_USB_Communication_Release/releases/latest):

| Fichero | Para qué |
|---------|----------|
| `protolink_fw.uf2` (Pico) o `protolink_fw_pico2.uf2` (Pico 2) | firmware de la placa |
| `protolink-X.Y.Z-macos-universal.tar.gz` | librería C/C++, bindings, ejemplos |
| `protolink-X.Y.Z-py3-none-macosx_11_0_universal2.whl` | paquete de Python |

Comprueba la descarga con `shasum -a 256 -c SHA256SUMS.txt --ignore-missing`.

```bash
tar xzf protolink-X.Y.Z-macos-universal.tar.gz
# La librería no está firmada: quita la marca de cuarentena que pone el navegador,
# o macOS se negará a cargar libprotolink.dylib ("Apple no puede comprobar...").
xattr -dr com.apple.quarantine protolink-X.Y.Z-macos-universal
cd protolink-X.Y.Z-macos-universal
python3 -m venv ~/.venvs/protolink && . ~/.venvs/protolink/bin/activate
pip install ../protolink-*-py3-none-macosx_11_0_universal2.whl
```


## 2. Flashear el firmware

1. Con la placa desconectada, mantén pulsado **BOOTSEL** y conéctala. Suelta el botón.
2. Aparece el volumen `RPI-RP2` (Pico) o `RP2350` (Pico 2). Arrastra el `.uf2` en el Finder, o:
   ```bash
   cp protolink_fw.uf2 /Volumes/RPI-RP2/
   ```
3. La placa se reinicia sola. macOS puede avisar de *"Disco no expulsado correctamente"*: es normal.

## 3. Encontrar el puerto

```bash
ls /dev/cu.usbmodem*        # p. ej. /dev/cu.usbmodem1101
```
Usa siempre el dispositivo **`/dev/cu.*`**, no `/dev/tty.*`. El nombre depende del puerto USB
físico, así que puede cambiar si conectas la placa en otro sitio. No hacen falta permisos especiales.

## 4. Probar

**Prueba automática** (eco, 100 Hz y pérdidas; ~10 s):
```bash
python examples/hw_check.py /dev/cu.usbmodem1101
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
python examples/hw_check.py /dev/cu.usbmodem1101 --rate 1000
```
En tu código: `link.start(1000)` (Python, C++, MATLAB) o `pl_start(link, 1000, 1000)` (C). La Pico
mantiene esa frecuencia con su propio reloj aunque el PC se bloquee; la librería sigue recibiendo en
segundo plano (cola de ~65 s a 1 kHz) y `link.pending()` dice cuántas tramas esperan.

**C / C++** desde la carpeta del release:
```bash
clang examples/example.c -Iinclude -Llib -lprotolink -Wl,-rpath,"$PWD/lib" -o example_c
clang++ -std=c++17 examples/example.cpp -Iinclude -Ibindings/cpp -Llib -lprotolink -Wl,-rpath,"$PWD/lib" -o example_cpp
./example_c /dev/cu.usbmodem1101 5
./example_cpp /dev/cu.usbmodem1101 5
```
Enlazado estático (sin `.dylib` en tiempo de ejecución):
`clang examples/example.c -DPL_STATIC -Iinclude lib/libprotolink_static.a -o example_c`.

**MATLAB** (`loadlibrary` necesita Xcode / Command Line Tools; vale tanto MATLAB nativo de
Apple Silicon como Intel):
```matlab
addpath('bindings/matlab')            % la carpeta del release
link = protolink.Link("/dev/cu.usbmodem1101");
link.send(single(1:9)); pause(0.05);
[v, seq, tUs, ok] = link.recvLatest()
delete(link)
```
Ejemplo completo: `examples/example_matlab.m`.

## 5. Problemas frecuentes

| Síntoma | Causa probable / solución |
|---------|---------------------------|
| No aparece `RPI-RP2` | Cable solo de carga, o no se mantuvo BOOTSEL al conectar |
| No aparece `/dev/cu.usbmodem*` | `.uf2` de otra placa (Pico vs Pico 2); vuelve a flashear |
| *"libprotolink.dylib" no se puede abrir porque Apple no puede comprobar…* | Falta `xattr -dr com.apple.quarantine <carpeta>` (paso 1) |
| `could not open or configure the port` | Ruta incorrecta, o el puerto está abierto por otro programa: `lsof /dev/cu.usbmodem*` |
| Se abre pero no llega nada (`[FAIL] the echo never matched`) | Firmware no flasheado o placa colgada: desconecta y vuelve a conectar |
| Las primeras tramas dan `lost`/`framing` | Normal si se abrió a mitad de trama; `hw_check` ya las descuenta |
| `lost` crece durante la prueba | Hub/adaptador USB-C saturado; prueba conectando directamente y envía la salida de `hw_check` |
| `OSError: could not load the protolink shared library` | Instala la wheel, o define `PROTOLINK_LIB=/ruta/libprotolink.dylib` |
| Al desconectar la placa | Las llamadas devuelven `PL_ERR_IO` (`i/o error or device disconnected`); hay que cerrar y volver a abrir |
