**Programación de Dispositivos Electrónicos**

# Trabajo Práctico 2: El Facón De la Fuente — Protección de Fuente

| **Institución** | Casa Pio IX |
|---|---|
| **Materia** | Programación de Dispositivos Electrónicos |
| **Docentes** | Nicolás Facón y Fernando De La Fuente |
| **Revisión** | 1.0 - Agosto 2026 |

## Tabla de firmas

| **Ejercicios** | **Tema** | **Fecha** | **Firma del docente** |
|---|---|---|---|
| 1 | Estructuras de datos | | |
| 2 | Comunicación UART | | |
| 3 y 4 | Software de PC — comunicación y persistencia | | |
| 5 | Software de PC — panel de estado | | |
| 6 y 7 | ADC y lógica de protección | | |
| 8 | Selector físico de parámetros | | |

| **Alumnos** | **Curso** | **Año** |
|---|---|---|
| Franco Zambrano | | 2026 |

## Índice

- [Introducción](#introducción)
  - [El dispositivo no es autónomo de la PC](#el-dispositivo-no-es-autónomo-de-la-pc)
  - [El armado físico](#el-armado-físico)
  - [Modalidad](#modalidad)
  - [Estructura de archivos del proyecto](#estructura-de-archivos-del-proyecto)
- [**Bloque A — Estructuras de datos**](#bloque-a--estructuras-de-datos)
  - [Ej 1: Configuración, evento y estado](#ej-1-configuración-evento-y-estado)
    - [a) Struct de configuración](#a-struct-de-configuración)
    - [b) Struct de evento](#b-struct-de-evento)
    - [c) Struct de estado](#c-struct-de-estado)
- [**Bloque B — Comunicación UART**](#bloque-b--comunicación-uart)
  - [Ej 2: UART no bloqueante](#ej-2-uart-no-bloqueante)
- [**Bloque C — Software de PC: comunicación y persistencia**](#bloque-c--software-de-pc-comunicación-y-persistencia)
  - [Ej 3: Protocolo de comunicación](#ej-3-protocolo-de-comunicación)
  - [Ej 4: Persistencia en JSON](#ej-4-persistencia-en-json)
- [**Bloque D — Software de PC: panel de estado**](#bloque-d--software-de-pc-panel-de-estado)
  - [Ej 5: Panel de estado en tiempo real](#ej-5-panel-de-estado-en-tiempo-real)
    - [Ejemplo — salida activa](#ejemplo--salida-activa)
    - [Ejemplo — salida cortada por sobrecorriente](#ejemplo--salida-cortada-por-sobrecorriente)
- [**Bloque E — ADC y lógica de protección**](#bloque-e--adc-y-lógica-de-protección)
  - [Ej 6: Lectura de tensión y corriente](#ej-6-lectura-de-tensión-y-corriente)
  - [Ej 7: Lógica de protección](#ej-7-lógica-de-protección)
- [**Bloque F — Selector físico de parámetros**](#bloque-f--selector-físico-de-parámetros)
  - [Ej 8: Selector por pulsador, LEDs y potenciómetro](#ej-8-selector-por-pulsador-leds-y-potenciómetro)

---

## Introducción

En este trabajo práctico vamos a construir **El Facón De la Fuente**, un dispositivo de protección para una fuente de alimentación. Su trabajo es simple de enunciar y exigente de implementar: vigilar constantemente la tensión y la corriente que entrega la fuente y, apenas alguna de las dos se salga de los límites permitidos, cortar la salida antes de que el problema dañe lo que está conectado.

Para lograrlo vamos a combinar tres frentes que ya venimos trabajando por separado a lo largo de la materia:

- **Hardware:** un divisor resistivo para sensar la tensión de salida, una resistencia de shunt junto con un amplificador operacional para sensar la corriente, y un MOSFET o un relé como elemento de corte.
- **Firmware de la Pico 2W:** lectura de ambas magnitudes por ADC, la lógica de decisión (cuándo cortar, cuánto esperar, cuándo reintentar), y la comunicación con la PC por UART a interrupciones.
- **Software de PC:** una aplicación de consola que permite configurar los umbrales de protección, ver en tiempo real lo que está midiendo la fuente, y dejar un registro histórico de todo lo que fue pasando.

El resultado final no es un experimento de laboratorio aislado: es, en una escala reducida, el mismo tipo de protección que llevan las fuentes conmutadas comerciales, las cargadores de batería y buena parte del equipamiento electrónico que ya usás todos los días.

### El dispositivo no es autónomo de la PC

Un punto importante para tener claro desde el principio: **este dispositivo no está pensado para funcionar de forma independiente de la PC**. La Pico no persiste ninguna configuración en su propia memoria: cada vez que se reinicia, arranca sin parámetros útiles y depende de que el software de PC se los transmita.

Mientras haya comunicación activa entre ambos, la PC es quien manda: le indica a la Pico los umbrales de tensión, el umbral de corriente (salvo que se use el potenciómetro para setearlo, ver Bloque F), el tiempo de restablecimiento y la cantidad máxima de reintentos. Si en algún momento se corta la comunicación (se desconecta el cable, se cierra el programa, etc.), la Pico sigue protegiendo la fuente con la **última configuración que haya recibido**, pero no va a poder actualizarla hasta que la comunicación se restablezca.

Cómo se organiza exactamente ese intercambio —quién habla primero, con qué frecuencia— es una decisión de diseño que les corresponde a ustedes, y se las planteamos en el Bloque C.

### El armado físico

El circuito de sensado y de corte es **responsabilidad completa del grupo**: no se entrega ninguna placa armada ni esquemático de referencia. Van a trabajar con las fuentes de alimentación del laboratorio, y el rango de corriente que puedan efectivamente medir y proteger va a depender de qué resistencias usen como carga de prueba. Eso también queda a criterio de cada grupo, según los componentes de los que dispongan.

En total, el hardware que tienen que resolver es:

- Divisor resistivo para sensar la tensión de salida de la fuente.
- Resistencia de shunt + amplificador operacional para sensar la corriente.
- MOSFET o relé como actuador de corte.
- Un pulsador para ciclar entre los parámetros configurables (Bloque F).
- Cinco LEDs indicadores del parámetro seleccionado (Bloque F).
- Un potenciómetro para fijar el valor del parámetro seleccionado (Bloque F).
- Un segundo pulsador de reset, para desbloquear la salida (Bloque F).
- Un LED de estado de salida, sobre el mismo GPIO que maneja el actuador (Bloque F).

> [!IMPORTANT]
> **Importante:** salvo que cuenten con un diodo zener para limitar la tensión, el divisor resistivo tiene que estar calculado considerando la **máxima tensión que pueda entregar la fuente**, no solo el rango que piensan usar en la prueba. La entrada del ADC de la Pico no debe superar los 3,3 V bajo ninguna circunstancia: un divisor mal dimensionado puede terminar quemando la Raspberry.

> [!TIP]
> Antes de programar una sola línea de la lógica de protección, aseguren el sensado: verifiquen con el ADC que las lecturas de tensión y corriente responden de forma coherente a los cambios reales en la fuente y en la carga. Programar sobre un sensado que no está calibrado hace perder mucho más tiempo del que ahorra.

> [!TIP]
> **Sugerencia de diseño:** además de las tres estructuras del Bloque A, es común que convenga definir structs propias para organizar el manejo interno de la UART y del ADC en el firmware (por ejemplo, para el estado de un buffer de recepción, o para los últimos valores convertidos). No es un entregable puntual de este TP, pero es un patrón de diseño que vamos a ver en clase a medida que avancemos con los Bloques B y E.

### Modalidad

El TP se realiza en parejas. A lo largo de las clases, van a ir llamando a los docentes para validar los bloques completados, tal como indica la tabla de firmas de la primera hoja (que deben tener impresa).

La entrega final se realiza comprimiendo todos los archivos del proyecto en un `.zip` o `.rar` con el nombre:

```text
5A_TP2_APELLIDO1_APELLIDO2.zip
```

La foto o escaneo de la hoja de firmas se entrega por separado, con el mismo formato de nombre:

```text
5A_TP2_APELLIDO1_APELLIDO2_firmas.jpg
```

> [!CAUTION]
> **No se considera entregado el TP si los archivos no siguen este formato.**

### Estructura de archivos del proyecto

Este TP incluye dos proyectos separados: el software de PC y el firmware de la Pico. El `.zip` de entrega debe contener ambos, cada uno en su propia carpeta:

```text
facondelafuente/
├── software_pc/
│   ├── includes/
│   │   ├── configuracion.h
│   │   ├── evento.h
│   │   ├── estado.h
│   │   └── panel.h
│   ├── sources/
│   │   ├── main.c
│   │   └── panel.c
│   └── libs/
│       ├── json.h / json.c
│       ├── fields.h
│       ├── serial.h / serial.c
│       └── console.h / console.c
└── firmware_pico/
    ├── app/
    ├── devices/
    ├── drivers/
    ├── sources/
    └── (CMakeLists.txt y demás archivos de configuración del proyecto)
```

Como en los demás trabajos prácticos de la materia, es **obligatorio** separar el código en múltiples archivos según su función. No es necesario modificar el `makefile` del proyecto de PC.

> [!WARNING]
> **Muy importante — qué NO incluir en la entrega:** el proyecto de la Pico genera una carpeta `build/` al compilar, con todos los archivos intermedios y binarios de la compilación. Esa carpeta **no se entrega**: puede pesar cientos de MB y no aporta nada que no se pueda regenerar compilando. Antes de comprimir el `.zip`, verifiquen que no esté incluida ninguna carpeta `build/` dentro de `firmware_pico/`.

---

## Bloque A — Estructuras de datos

En este bloque definimos las tres estructuras principales que va a usar el sistema. Para cada una se indican los campos mínimos que debe tener; los nombres exactos, los tipos de dato y la organización interna quedan a criterio del grupo.

### Ej 1: Configuración, evento y estado

#### a) Struct de configuración

Representa los parámetros de protección vigentes. Es la estructura que la PC persiste en `config.json` y que le transmite a la Pico.

Campos mínimos:

1. Umbral mínimo de tensión.
2. Umbral máximo de tensión.
3. Umbral máximo de corriente.
4. Tiempo de restablecimiento.
5. Cantidad máxima de reintentos antes de bloquear la salida.

#### b) Struct de evento

Representa una entrada del historial de eventos: cada vez que ocurre algo relevante (un corte, un reintento, un bloqueo, etc.) se agrega una entrada nueva. Es la estructura que la PC persiste, como vector de tamaño fijo, en `eventos.json`.

Campos mínimos:

1. El tipo de evento (pensar qué situaciones distintas hay que poder distinguir: sobretensión, baja tensión, sobrecorriente, reintento, bloqueo por reintentos máximos, etc.).
2. Una marca temporal que permita saber cuándo ocurrió, tomada de algún dato que la Pico ya les esté informando (por ejemplo, el tiempo transcurrido desde el arranque del micro). La PC todavía no cuenta con ninguna librería para manejo de fecha u hora en esta materia, así que no corresponde calcular ni generar marcas de tiempo del lado de la PC.
3. El valor medido asociado al evento, cuando corresponda.

#### c) Struct de estado

Representa una "foto" del estado actual del sistema en un momento dado. A diferencia de las dos anteriores, **no se persiste**: vive únicamente en memoria RAM del lado de la PC, reconstruida a partir de lo último que informó la Pico por UART.

Campos mínimos:

1. Tensión actual.
2. Corriente actual (ver la aclaración sobre este campo en el Bloque D, para cuando la salida está cortada).
3. Si la salida está activa o cortada.
4. Tiempo transcurrido desde el último corte, tal como lo informa la Pico con su propio timer — la PC solo lo muestra, no lo calcula por su cuenta.
5. Cantidad de reintentos utilizados en el ciclo actual.
6. La configuración vigente (los cinco campos del punto **a**).

> [!NOTE]
> No hace falta que el struct que usen del lado del firmware sea idéntico al que usan del lado de la PC: en la Pico es habitual necesitar algún campo o contador extra que del lado de la PC no tiene sentido.

---

## Bloque B — Comunicación UART

Este bloque se resuelve enteramente en el firmware de la Pico. Antes de pensar en qué información se intercambia, hay que tener andando el medio por el cual se va a intercambiar.

> [!NOTE]
> Consultar el apunte de puerto serie para microcontroladores para la configuración del periférico UART y el manejo de interrupciones.

### Ej 2: UART no bloqueante

Implementar en el firmware la recepción y el envío por el módulo UART del microcontrolador **enteramente por interrupciones**, tanto para transmisión (TX) como para recepción (RX). No debe haber ningún punto del código que haga _polling_ sobre el estado del periférico.

Al terminar este ejercicio deberían poder enviar y recibir bytes por la UART GPIO (conectada al conversor USBSerie TTL) sin bloquear la ejecución del resto del programa en ningún momento.

---

## Bloque C — Software de PC: comunicación y persistencia

### Ej 3: Protocolo de comunicación

Con la UART funcionando de ambos lados, hace falta definir **qué** se dice y **cuándo**. Acá tienen una decisión de diseño para tomar como grupo, entre dos filosofías posibles:

1. **La Pico publica:** de forma periódica, sin que se lo pidan, la Pico envía su estado actual por UART. La PC escucha y actualiza su pantalla con lo que va llegando.
2. **La Pico responde:** la Pico se queda esperando y solo envía información cuando la PC se lo solicita explícitamente con un comando.

Cualquiera sea la opción elegida, tiene que seguir cumpliéndose lo planteado en la introducción: la PC es la que envía la configuración, y la Pico se queda con la última que recibió si la comunicación se corta.

Definan el protocolo completo: qué mensajes existen, con qué formato, y en qué momento se envía cada uno. **El protocolo debe ser en formato ASCII** (texto legible, no binario). Por el lado de la PC, se comunican a través de la UART GPIO, no de la UART de debugging USB. Trabajar con serialización binaria por UART es posible, pero mucho más compleja de sincronizar que en un socket: acá no hay un framing como el de TCP que les garantice dónde empieza y termina cada mensaje, así que ASCII con un delimitador claro (por ejemplo, fin de línea) es la opción más simple y más razonable para este TP.

### Ej 4: Persistencia en JSON

Utilizando `json.h` y `fields.h`, implementar del lado de la PC:

- Persistencia del struct de **configuración** en `config.json`: se carga al iniciar el programa (si el archivo no existe, se usan valores por defecto) y se guarda cada vez que el usuario modifica algún parámetro desde el menú.
- Persistencia del vector de **eventos** en `eventos.json`: cada evento nuevo que llega desde la Pico (o que se detecta del lado de la PC, según cómo hayan diseñado el protocolo) se agrega al vector y se guarda.

---

## Bloque D — Software de PC: panel de estado

### Ej 5: Panel de estado en tiempo real

Implementar, usando `console.h`, una pantalla que muestre en tiempo real el struct de **estado** completo: tensión actual, corriente actual, si la salida está activa o cortada, tiempo desde el último corte, reintentos utilizados, y la configuración vigente.

Se pide que la pantalla se vea prolija: usar colores para distinguir estados (por ejemplo, un color para "activa" y otro para "cortada"), y organizar la información con alguna estructura visual clara (tabla, cuadro, separación entre secciones). No hace falta limitarse a texto plano en una sola columna.

Además de la visualización, el software de PC tiene que permitir configurar, como mínimo:

1. Umbral mínimo de tensión.
2. Umbral máximo de tensión.
3. Umbral máximo de corriente.
4. Tiempo de restablecimiento.
5. Cantidad máxima de reintentos.
6. **Desbloquear el protector**, para los casos en que se alcanzó la cantidad máxima de reintentos y la salida quedó bloqueada (Ej 7).

A continuación se muestra un **mockup orientativo** de la mitad de la pantalla (la parte de estado en tiempo real). El resto de la pantalla — menú, ingreso de configuración, listado de eventos, etc. — queda a criterio de cada grupo.

#### Ejemplo — salida activa:

```text
══════ EL FACÓN DE LA FUENTE — PANEL DE ESTADO ══════
Tension actual ........... 12.08 V
Corriente actual ......... 0.62 A
Estado de salida ......... ACTIVA
Tiempo desde el corte .... --
Reintentos utilizados .... 0 / 3
── Configuracion vigente ──
Umbral tension minima .... 11.50 V
Umbral tension maxima .... 12.50 V
Umbral corriente maxima .. 1.00 A
Tiempo de restablecimiento 10 s
Reintentos maximos ....... 3
```

#### Ejemplo — salida cortada por sobrecorriente:

> [!NOTE]
> Notar que, apenas se corta la salida, deja de circular corriente: mostrar una "corriente actual" en ese momento sería mostrar `0.00 A`, un dato sin utilidad. Lo que tiene sentido mostrar es la **última corriente medida** antes de cortar, que es lo que disparó la protección.

```text
══════ EL FACÓN DE LA FUENTE — PANEL DE ESTADO ══════
Tension actual ........... 12.03 V
Ultima corriente medida .. 1.24 A
Estado de salida ......... CORTADA (sobrecorriente)
Tiempo desde el corte .... 00:04
Reintentos utilizados .... 1 / 3
── Configuracion vigente ──
Umbral tension minima .... 11.50 V
Umbral tension maxima .... 12.50 V
Umbral corriente maxima .. 1.00 A
Tiempo de restablecimiento 10 s
Reintentos maximos ....... 3
```

---

## Bloque E — ADC y lógica de protección

Este bloque se resuelve en el firmware de la Pico.

> [!NOTE]
> Consultar el apunte de ADC para la configuración del periférico y la conversión de las lecturas crudas a unidades físicas.

### Ej 6: Lectura de tensión y corriente

Configurar el ADC para leer los canales correspondientes a la tensión de salida (vía divisor resistivo) y a la corriente (vía shunt + operacional). Convertir las lecturas crudas del conversor a las unidades físicas correspondientes: volts [V] para la tensión, amperes [A] para la corriente.

> [!TIP]
> Muchas veces conviene trabajar internamente con una escala más chica (por ejemplo, milivolts [mV] o miliamperes [mA]) en lugar de volts o amperes. Así el valor completo entra en un entero, sin perder la parte decimal, y se evita usar `float` en el firmware.

### Ej 7: Lógica de protección

Implementar la lógica central del dispositivo:

- Sensar tensión y corriente de forma continua.
- Si la tensión cae por debajo del umbral mínimo, supera el umbral máximo, o la corriente supera el umbral máximo configurado: cortar la salida (MOSFET o relé) y registrar el evento correspondiente. Una vez cortada la salida, esperar el **tiempo de restablecimiento** configurado y luego reconectar automáticamente.
- Si al reconectar el problema persiste, volver a cortar, registrar el evento, e incrementar el contador de reintentos.
- Si se alcanza la **cantidad máxima de reintentos**, la salida queda bloqueada de forma indefinida (ya no se vuelve a reconectar sola). El desbloqueo, en ese caso, se hace con el pulsador de reset físico (Bloque F) o con un comando específico enviado por UART.

> [!TIP]
> Piensen en el concepto de histéresis al definir cuándo se considera "resuelto" un problema de tensión: un umbral único puede generar oscilaciones de corte/reconexión si el valor medido queda justo en el límite.

---

## Bloque F — Selector físico de parámetros

### Ej 8: Selector por pulsador, LEDs y potenciómetro

Como se mencionó en el Bloque E, no hay forma de setear valores analógicos con el potenciómetro para más de un parámetro a la vez: la Pico 2W solo cuenta con tres entradas ADC, y ya están ocupadas por la tensión, la corriente y el propio potenciómetro. Por eso, el potenciómetro se reutiliza para **los cinco** parámetros configurables, seleccionando de a uno por vez.

Implementar:

- **Un pulsador** que cicla entre los cinco parámetros configurables: umbral mínimo de tensión, umbral máximo de tensión, umbral máximo de corriente, tiempo de restablecimiento y cantidad máxima de reintentos. Cada vez que se presiona, se pasa al siguiente parámetro de la lista (de forma circular).
- **Cinco LEDs**, uno por parámetro, que indiquen cuál está seleccionado en cada momento (un único LED encendido a la vez).
- El **potenciómetro** debe fijar el valor del parámetro actualmente seleccionado en el momento en que se presiona el pulsador para pasar al siguiente (es decir, el valor se "confirma" al salir de esa selección, no de forma continua).
- **Un segundo pulsador de reset**, que desbloquee la salida cuando quedó bloqueada por haber alcanzado la cantidad máxima de reintentos (Ej 7).
- **Un LED de estado de salida**, conectado al mismo GPIO que maneja el actuador (MOSFET o relé), que indique si la salida está activa o cortada.

> [!NOTE]
> El valor específico que se está configurando con el potenciómetro **no tiene forma de verse en el hardware**: solo se puede ver a través del panel de estado de la PC (Bloque D). En el hardware únicamente se ve, mediante los LEDs, qué parámetro se está configurando en cada momento — no su valor.
