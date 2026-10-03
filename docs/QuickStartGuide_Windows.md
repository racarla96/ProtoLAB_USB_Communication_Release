# Guía rápida — Windows

Poner en marcha protolink con una Raspberry Pi Pico real en Windows 10/11 x64.

Guías para otros sistemas: [Linux](QuickStartGuide_Linux.md) · [macOS](QuickStartGuide_macOS.md)

## 0. Qué necesitas

- Raspberry Pi Pico (RP2040) o Pico 2 (RP2350) y un cable USB **de datos** (no solo de carga).
- Python ≥ 3.8 x64 para la prueba rápida ([python.org](https://www.python.org/downloads/windows/)).
- Para C/C++: Visual Studio 2022 o las *Build Tools* con la carga de trabajo
  **Desarrollo para el escritorio con C++**.
- No hace falta instalar drivers: Windows 10/11 trae el driver USB-CDC (`usbser.sys`).

## 1. Descargar

De la página [Releases](https://github.com/racarla96/ProtoLAB_USB_Communication_Release/releases/latest):

| Fichero | Para qué |
|---------|----------|
| `protolink_fw.uf2` (Pico) o `protolink_fw_pico2.uf2` (Pico 2) | firmware de la placa |
| `protolink-X.Y.Z-windows-x64.zip` | librería C/C++, bindings, ejemplos |
| `protolink-X.Y.Z-py3-none-win_amd64.whl` | paquete de Python |

Comprobar una descarga (PowerShell) y comparar con `SHA256SUMS.txt`:
```powershell
Get-FileHash .\protolink-X.Y.Z-windows-x64.zip -Algorithm SHA256
```

Descomprime el `.zip` (clic derecho → *Extraer todo*) e instala el paquete de Python:
```powershell
cd protolink-X.Y.Z-windows-x64
py -m venv .venv
.venv\Scripts\python.exe -m pip install ..\protolink-X.Y.Z-py3-none-win_amd64.whl
```
`protolink.dll` lleva el runtime de C enlazado estáticamente: no necesita el *Visual C++ Redistributable*.


## 2. Flashear el firmware

1. Con la placa desconectada, mantén pulsado **BOOTSEL** y conéctala. Suelta el botón.
2. Aparece una unidad `RPI-RP2` (Pico) o `RP2350` (Pico 2) en el Explorador. Arrastra el `.uf2` a ella.
3. La placa se reinicia sola (Windows puede avisar de que la unidad se quitó sin expulsar: es normal)
   y aparece como puerto COM.

## 3. Encontrar el puerto COM

*Administrador de dispositivos* → **Puertos (COM y LPT)** → *Dispositivo serie USB (COM5)*.
O en PowerShell:
```powershell
Get-PnpDevice -Class Ports -PresentOnly | Format-Table FriendlyName, InstanceId
# USB\VID_2E8A&PID_000A... = la Pico
```
El número puede cambiar si conectas la placa en otro puerto USB. Los puertos ≥ COM10 funcionan sin
nada especial (`"COM12"`).

En Windows un puerto COM **solo lo puede abrir un programa a la vez**: cierra Thonny, el monitor serie
de Arduino, PuTTY u otra sesión de MATLAB/Python que lo tenga abierto.

## 4. Probar

**Prueba automática** (eco, 100 Hz y pérdidas; ~10 s):
```powershell
.venv\Scripts\python.exe examples\hw_check.py COM5
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
.venv\Scripts\python.exe examples\hw_check.py COM5 --rate 1000
```
En tu código: `link.start(1000)` (Python, C++, MATLAB) o `pl_start(link, 1000, 1000)` (C). La Pico
mantiene esa frecuencia con su propio reloj aunque el PC se bloquee; la librería sigue recibiendo en
segundo plano (cola de ~65 s a 1 kHz) y `link.pending()` dice cuántas tramas esperan.

**C / C++** desde la carpeta del release, en la *x64 Native Tools Command Prompt for VS 2022*:
```bat
cl /nologo /MD examples\example.c /Iinclude lib\protolink.lib /Fe:example_c.exe
cl /nologo /MD /EHsc /std:c++17 examples\example.cpp /Iinclude /Ibindings\cpp lib\protolink.lib /Fe:example_cpp.exe
copy lib\protolink.dll .
example_c.exe COM5 5
example_cpp.exe COM5 5
```
Enlazado estático (sin DLL): `cl /nologo /MD /DPL_STATIC examples\example.c /Iinclude lib\protolink_static.lib`.
`PL_STATIC` es obligatorio en ese caso (evita `__declspec(dllimport)`).

**MATLAB** (`loadlibrary` necesita un compilador C: el add-on *MATLAB Support for MinGW-w64 C/C++
Compiler* o Visual Studio; compruébalo con `mex -setup C`):
```matlab
addpath('C:\ruta\protolink-X.Y.Z-windows-x64\bindings\matlab')
link = protolink.Link("COM5");
link.send(single(1:9)); pause(0.05);
[v, seq, tUs, ok] = link.recvLatest()
delete(link)
```
Ejemplo completo: `examples\example_matlab.m`. Si cambias de versión de la DLL, ejecuta
`unloadlibrary protolink` (o reinicia MATLAB) para que cargue la nueva.

## 5. Problemas frecuentes

| Síntoma | Causa probable / solución |
|---------|---------------------------|
| No aparece `RPI-RP2` | Cable solo de carga, o no se mantuvo BOOTSEL al conectar |
| No aparece ningún puerto COM nuevo | `.uf2` de otra placa (Pico vs Pico 2); vuelve a flashear |
| `could not open or configure the port` | Número de COM incorrecto, o el puerto está abierto por otro programa |
| Se abre pero no llega nada (`[FAIL] the echo never matched`) | Firmware no flasheado o placa colgada: desconecta y vuelve a conectar |
| Las primeras tramas dan `lost`/`framing` | Normal si se abrió a mitad de trama; `hw_check` ya las descuenta |
| `lost` crece durante la prueba | Hub USB saturado o PC muy cargado; prueba otro puerto USB y envía la salida de `hw_check` |
| `OSError: could not load the protolink shared library` | Instala la wheel `win_amd64` con un Python **x64**, o define `PROTOLINK_LIB=C:\ruta\protolink.dll` |
| MATLAB: `No supported compiler was found` | Instala el add-on MinGW-w64 (Home → Add-Ons) y ejecuta `mex -setup C` |
| Al desconectar la placa | Las llamadas devuelven `PL_ERR_IO` (`i/o error or device disconnected`); hay que cerrar y volver a abrir |
