# protolink en Simulink

Bloque **`sfun_protolink`** (S-Function en C) en `bindings/simulink/`. Funciona en simulación normal
(modo *Normal*), sin toolboxes adicionales, y puede sincronizarse con el reloj real.

> **Estado:** probado en MATLAB/Simulink R2026b (Linux) con una Pico real; el bloque con máscara
> (placas, lista de puertos) se probó con una Pico, sin hardware para las demás placas ni en Windows/macOS.

## 1. Instalar (una vez)

**Desde un paquete listo** (`protolink-matlab-<versión>-<sistema>-<release>.zip`, uno por sistema operativo,
con el bloque ya compilado): descomprime y, en MATLAB,

```matlab
cd protolink-matlab-...      % la carpeta descomprimida
protolink_setup              % añade las carpetas al path y crea la librería de Simulink
```
No hace falta compilador. Si tu MATLAB es más antiguo que el que compiló el bloque, o falta el paquete de tu
sistema: `protolink_setup('rebuild')` lo compila (necesita `mex -setup C`: MinGW-w64 o Visual Studio en Windows,
Xcode en macOS, gcc en Linux).

**Desde el archivo de la librería** (`protolink-X.Y.Z-<sistema>`, sin el bloque compilado):

```matlab
cd('<carpeta>/bindings/simulink')
protolink_setup              % compila sfun_protolink y crea protolink_lib.slx
```

## 2. Probar con el modelo de ejemplo

```matlab
protolink_demo('/dev/ttyACM0', 500)     % o 'COM5', '/dev/cu.usbmodem1101'; 500 Hz, 10 s
```
Crea `protolink_demo_model`: nueve senos → Pico → *Scope*. Con el firmware de referencia (eco) el
*Scope* debe mostrar los mismos senos, retrasados 1–2 pasos; `New` debe valer 1 en casi todos los
pasos y `Pending` mantenerse cerca de 0.

## 3. Usarlo en tu modelo

**Bloque con máscara (lo más cómodo)**: abre el *Library Browser* (o escribe `protolink_lib`) y arrastra
**protolink** a tu modelo; o desde la línea de comandos `protolink_block('mimodelo/protolink')`.
Doble clic y se rellena todo con desplegables:

| Campo | Qué es |
|-------|--------|
| **Board** | Pico / Pico 2, ESP32 (WROOM), Arduino UNO, LGT8F328P o *Custom*. Fija la velocidad del puerto, el manejo de DTR/RTS, la espera tras abrir (el UNO y el LGT8F328P se resetean) y la frecuencia máxima de la placa |
| **Port** | Desplegable con los puertos serie conectados ahora al PC; **Refresh port list** lo actualiza al enchufar o quitar una placa. *(other)* permite escribir uno a mano |
| **Channels to board / from board** | Ancho de la entrada y de la salida del bloque; los fija el firmware de la placa. El botón **Read from board** abre la placa (con la configuración del bloque) y los rellena; con el firmware de pruebas son 9 y 9 |
| **Rate (Hz)** | 10, 50, 100, 200, 250, 500, 1000 o *Custom* (cualquier entero; se rechaza si supera lo que admite la placa) |
| **Read mode** | *Latest value* (la última trama, nunca espera) o *Every frame (FIFO)* (todas, en orden) |
| **FIFO timeout** | espera máxima en modo FIFO |
| **Real time** | frena la simulación al reloj real |
| *Custom board* | solo con *Board = Custom*: baudios, DTR/RTS y espera tras abrir |

El periodo de muestreo es `1/Rate`, y el **paso fijo del modelo debe ser `1/Rate`** (*Model Settings →
Solver → Fixed-step size*); si no, Simulink da el error «All sample times in your model must be an integer
multiple of the fixed-step size» y la máscara avisa. Si cambias *Rate*, cambia también el paso del modelo.
Los parámetros reales de la S-function (tabla de abajo) los calcula la máscara.

**A mano**: añade un bloque **S-Function** (`Simulink → User-Defined Functions`), con *S-function name*
`sfun_protolink` y *S-function parameters*:

```
'COM5', 500, 1/500, 1, 100, 1, 115200, 1, 0, 9, 9
```

| # | Parámetro | Significado |
|---|-----------|-------------|
| 1 | `PORT` | puerto serie (texto entre comillas simples) |
| 2 | `RATE_HZ` | frecuencia que se pide a la Pico al arrancar (entero 1..1000); `0` = no tocarla (conserva la que tuviera: 100 Hz tras encender) |
| 3 | `TS` | periodo de muestreo del bloque en segundos (normalmente `1/RATE_HZ`) |
| 4 | `MODE` | `1` = *latest*: en cada paso, la trama más reciente (no espera; las anteriores se descartan). `2` = *fifo*: la siguiente trama en orden, esperando hasta `TIMEOUT_MS` (no se descarta nada; la Pico marca el ritmo) |
| 5 | `TIMEOUT_MS` | espera máxima en modo 2 |
| 6 | `REALTIME` | `1` = frenar la simulación al reloj real; `0` = tan rápido como pueda |
| 7 | `BAUD` | velocidad del puerto: `115200` para la Pico, `921600` para el firmware del ESP32 |
| 8 | `DTR_RTS` | `1` para la Pico; `0` para placas ESP32 (no tocar las líneas que resetean la placa) |
| 9 | `BOOT_MS` | espera en ms tras abrir el puerto, antes del primer `start`: `0` para Pico y ESP32, `2500` para Arduino UNO y LGT8F328P (se resetean al abrir) |
| 10 | `N_TO` | canales hacia la placa = ancho de la entrada del bloque (1..64) |
| 11 | `N_FROM` | canales desde la placa = ancho de la salida 1 (1..64) |

`N_TO` y `N_FROM` los fija el firmware de la placa (9 y 9 en el firmware de pruebas). Al arrancar, el bloque le
pregunta a la placa y se detiene con un mensaje claro si no coinciden con lo configurado.

Entrada: vector de `N_TO` valores que se envía a la placa en cada paso (después de calcular las salidas,
así que el bloque no tiene paso directo y no crea bucles algebraicos).
Salidas: (1) `N_FROM` valores de la última trama, (2) `seq`, (3) `timestamp_us` de la Pico, (4) `new` (1 si
llegó trama en este paso), (5) tramas pendientes en la cola de recepción.

Configura el modelo con *Solver* `FixedStepDiscrete` y *Fixed-step size* = `TS`.

**Elegir modo.** Para un lazo de control usa `MODE = 1` con `REALTIME = 1`: cada paso trabaja con el
dato más fresco. Si lo que quieres es registrar todas las muestras sin perder ninguna, `MODE = 2` con
`REALTIME = 0`: Simulink avanza al ritmo de la Pico.

**Si el PC se cuelga.** La Pico sigue a su frecuencia y la librería sigue recibiendo en su propio hilo
(cola de ~65 s a 1 kHz). Con `MODE = 1`, al recuperarse el bloque salta a la trama más reciente; con
`MODE = 2` las va entregando en orden. La salida 5 (`pending`) muestra el retraso acumulado.

**Rendimiento.** En simulación normal, 1 kHz es alcanzable con un modelo ligero; si `pending` crece de
forma sostenida (modo 2) o `new` vale 0 a menudo (modo 1), el modelo no da abasto a esa frecuencia.

## Otras vías (anotadas, para cuando hagan falta)

| Opción | Cuándo | Qué haría falta |
|--------|--------|-----------------|
| **Bloque MATLAB System** sobre `protolink.Link` | Sin compilador MEX; hasta ~200 Hz | Una clase `matlab.System` que llame a `Link.m` (ejecución interpretada) |
| **Simulink Desktop Real-Time** | Tiempo real estricto en el PC (kernel de SLDRT) | Comprobar si el kernel admite S-Functions de usuario con E/S USB; si no, usar el bloque *Packet Input/Output* de SLDRT sobre el puerto serie, implementando la trama en el modelo |
| **Simulink Coder / Embedded Coder** | Generar un ejecutable del modelo | Un fichero TLC para `sfun_protolink` (o reescribir el bloque con *Legacy Code Tool*) y enlazar `protolink_static` |
