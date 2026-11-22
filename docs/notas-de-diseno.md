# Notas de diseño: el porqué de cada cosa

La idea de este documento no es repetir *qué* conectaste, sino *por qué*. Si entendés el porqué, la próxima placa la diseñás sin guía.

Al final de cada sección hay preguntas para autoevaluarte. Las respuestas están ocultas: pensalas antes de abrirlas.

---

## 0. Cómo está organizado el proyecto

**Esquemático jerárquico.** Una hoja raíz con 5 bloques, y cada bloque es una hoja con una función. Así es más fácil de leer, de revisar y de reutilizar: la hoja de alimentación USB-C te sirve tal cual para el próximo proyecto.

**Tres formas de conectar sin cable:**

| Qué | Tecla | Alcance | Cuándo |
|---|---|---|---|
| Etiqueta local | `L` | Solo dentro de su hoja | Señales internas (cristal, QSPI, `ENC_A1`) |
| Etiqueta jerárquica | `H` | Sale de la hoja hacia el bloque padre | Señales que van a otra hoja (`USB_DP`, `SR_CLK`) |
| Etiqueta global | `Ctrl+L` | Todo el proyecto | Casi nunca: conecta por nombre en cualquier hoja y puede unir cosas sin que te des cuenta |
| Símbolo de power | — | Todo el proyecto | Solo alimentación (+3V3, +5V, GND) |

Una etiqueta jerárquica crea un **pin en el bloque** de la hoja raíz. Ese pin conecta **solo si le tirás un cable** en la raíz: que dos pines se llamen igual no alcanza.

**La grilla importa.** El cable salta de punto en punto de la grilla (50 mil = 1.27 mm). Si un pin queda fuera de grilla, el cable nunca cae justo en el pin. Por eso no te dejaba unir los bloques.

**El campo LCSC.** JLCPCB arma la placa cruzando tu BOM con su stock. El campo `LCSC` le dice exactamente qué parte poner. Sin ese campo, la parte no se ensambla o te piden que elijas a mano.

**Tipos de pin y ERC.** Cada pin de un símbolo tiene un tipo: entrada, salida, bidireccional, pasivo, entrada de potencia, salida de potencia, etc. El ERC usa esos tipos para detectar errores: dos salidas peleándose, una entrada que nadie maneja, una red de alimentación sin fuente. Por eso los símbolos de JLC dan tanto ruido: tienen casi todo en "Unspecified" o mal tipado, como pasó con el USBLC6. Los de la librería oficial de KiCad vienen bien tipados.

<details><summary>¿Por qué usamos los genéricos de KiCad (<code>Device:R</code>) para los pasivos en vez de bajarlos con jlcget?</summary>

Porque una resistencia es una resistencia: lo único que cambia es el valor y el footprint. Si bajás cada valor como símbolo propio, la librería se llena de cientos de símbolos iguales. Con el genérico más el campo LCSC tenés lo mismo, más ordenado.
</details>

---

## 1. Alimentación y USB

### USB-C: las resistencias de 5.1k en CC1 y CC2

Un cargador o puerto USB-C **no entrega 5 V hasta que detecta que hay un dispositivo** conectado. Lo detecta midiendo una resistencia a GND en los pines CC: 5.1k significa "soy un dispositivo que consume (sink)".

**¿Por qué una resistencia por pin y no una sola compartida?** El cable USB-C usa CC1 o CC2 según cómo lo enchufes. Además, los cables "e-marked" (los C-a-C buenos) usan el otro pin. Si compartís una sola resistencia, el cargador ve una resistencia equivalente que no esperaba y no enciende. La Raspberry Pi 4 salió con ese error.

### D+/D− duplicados y SBU sin usar

El conector tiene dos pares de datos, uno para cada orientación del enchufe. Los unís para que funcione en las dos posiciones. SBU es para modos alternativos, como video, y USB 2.0 no los usa.

### USBLC6-2SC6 (ESD)

Cuando enchufás un cable, la descarga electrostática de tu mano entra por el conector. El USBLC6 la desvía a GND antes de que llegue al RP2040. Tiene muy poca capacitancia (~1 pF), así que no deforma las señales USB. **Va pegado al conector**: la descarga tiene que encontrar el camino a GND antes de recorrer la placa.

### Polyfuse (500 mA)

Es un fusible que se resetea solo. Con un cortocircuito se calienta, su resistencia sube mucho y corta la corriente. Cuando se enfría, vuelve a conducir.

- **Corriente de hold:** la que aguanta sin cortar. Por eso descartamos el de 250 mA.
- **Corriente de trip:** la que asegura que corte.

Protege el puerto de la computadora si algo en tu placa se pone en corto.

### AP2112K-3.3 (LDO)

Baja de 5 V a 3.3 V. Un LDO es simple y silencioso (sin ruido de conmutación, bueno para audio). Su contra: **toda la diferencia de tensión se disipa como calor**.

> P = (Vin − Vout) × I = (5 − 3.3) × I = 1.7 V × I

| Consumo | Calor en el LDO |
|---|---|
| 100 mA | 0.17 W, tibio |
| 300 mA | 0.51 W, mucho para un SOT-23-5 |

Un SOT-23-5 tiene una resistencia térmica del orden de 200 °C/W o más (el valor exacto está en el datasheet). Con 0.5 W, eso lo pone fácil 100 °C arriba del ambiente. **Por eso la corriente de los LEDs no debería pasar por este regulador.** Lo resolvemos en la hoja de LEDs, alimentándolos desde +5V. Además, en el PCB, dale cobre al pad de GND del LDO para que disipe.

- **EN unido a VIN:** el regulador queda siempre encendido.
- **Capacitores de 10 µF en la entrada y la salida:** un LDO es un lazo de realimentación y necesita un capacitor mínimo en la salida para no oscilar (lo dice el datasheet). El de la entrada ayuda a mantener estable la tensión cuando hay picos de consumo.

### PWR_FLAG

El ERC chequea que toda red con pines "entrada de potencia" tenga algún pin "salida de potencia". Pero un conector o un fusible son pasivos, y el ERC no sabe que por ahí entra energía. El PWR_FLAG le dice "acá hay alimentación, confiá". Por eso hizo falta en VBUS (sale del conector), en +5V (sale del fusible) y en +3V3 (el símbolo de JLC del LDO no marcaba VOUT como salida).

### Test points

Para el bring-up: lo primero que hacés con una placa nueva es medir las tensiones, y un pad dedicado es mucho más cómodo que pinchar la pata de un chip de 0.4 mm de paso.

<details><summary>Enchufás la placa a un cargador USB-C de celular y no prende, pero con un cable USB-A a C sí. ¿Qué revisarías primero?</summary>

Las resistencias de CC. Los cables A-a-C traen la resistencia del lado del cable y siempre dan 5 V; un cargador C-a-C solo entrega 5 V si ve los 5.1k en CC. Si falta una, o si están compartidas, no enciende.
</details>

<details><summary>¿Por qué el USBLC6 va al lado del conector y no al lado del RP2040?</summary>

Porque la descarga entra por el conector. Si la protección está lejos, la descarga recorre la pista (y acopla a las vecinas) antes de llegar a ella.
</details>

---

## 2. MCU (RP2040)

### Por qué el chip suelto y no una Pico

Para un portfolio, un RP2040 bien implementado demuestra que sabés hacer el circuito de un micro: cristal, flash, desacople y USB. Una Pico soldada encima no demuestra nada de eso. Además ocupa menos y sale más barato en serie.

### La flash externa (W25Q16) y BOOTSEL

El RP2040 **no tiene memoria de programa interna**. Ejecuta el código directo desde la flash externa por QSPI (esto se llama XIP, *execute in place*). Por eso no podés escribir la flash mientras corre código desde ella (de ahí `flash_safe_execute()` en el firmware).

**BOOTSEL** aprovecha un truco: al arrancar, la ROM interna del RP2040 mira la línea `QSPI_SS`. Si está en bajo, no arranca desde la flash y aparece como un pendrive USB para cargar firmware. El botón pone `QSPI_SS` a GND.

- **La 1k en serie con el botón:** si apretás el botón mientras el micro maneja `QSPI_SS`, sin resistencia sería un cortocircuito directo de una salida a GND.
- **La 10k a +3V3:** durante el encendido, antes de que el RP2040 tome control, asegura que la flash esté deseleccionada.

### El regulador interno y el 1.1 V

El núcleo del RP2040 funciona a 1.1 V. Tiene un regulador interno: entra 3.3 V por `VREG_IN`, sale 1.1 V por `VREG_VOUT`, y vos lo llevás a los pines `DVDD`. Es raro que un chip te haga cablear su propia alimentación por fuera, pero así te deja poner los capacitores donde corresponde.

### Desacople: un 100 nF por pin de alimentación

Un chip digital consume corriente en picos muy rápidos, en cada flanco de reloj. La pista desde el regulador tiene inductancia y no puede entregar esos picos a tiempo. El capacitor pegado al pin es un "tanque" local que los entrega.

**Lo que más importa es la distancia al pin, no el valor exacto.** Un 100 nF a 2 cm sirve mucho menos que uno a 1 mm. Por eso en el PCB va uno al lado de cada pin. Los símbolos que tienen pines apilados, como IOVDD ×6, te esconden eso en el esquemático, pero en la placa son 6 pines físicos.

### Cristal de 12 MHz

- **¿Por qué un cristal?** USB necesita un reloj muy preciso (±0.25 %). El oscilador RC interno del RP2040 varía mucho más con la temperatura y la tensión.
- **¿Por qué 12 MHz?** La ROM de arranque (la que maneja BOOTSEL) asume 12 MHz para levantar el USB. Otro valor rompería el modo de carga.
- **La 1k en serie con XOUT:** limita la potencia que el oscilador le mete al cristal. Si lo excitás de más, envejece rápido o oscila en un armónico.
- **Los 15 pF:** el cristal está especificado para oscilar exacto con cierta capacidad de carga (CL), que figura en el datasheet. La cuenta es:
  > C = 2 × (CL − C_parásita) = 2 × (10 pF − ~2.5 pF) ≈ 15 pF

  Con otro cristal (otro CL) cambian los capacitores. Por eso usamos el mismo cristal que el diseño de referencia: la cuenta ya está probada.

### Las 27 Ω en D+/D−

USB espera líneas de 90 Ω diferencial (unos 45 Ω por línea). El driver USB del RP2040 tiene una impedancia de salida más baja que eso. Los 27 Ω en serie completan la diferencia para que la señal no rebote en los extremos. **Van pegados al RP2040**, porque la terminación tiene que estar en la fuente.

### Lo demás

- **TESTEN a GND:** es un modo de test de fábrica. Si queda flotando, el chip puede entrar en ese modo y no arrancar.
- **RUN:** es el reset. La pull-up lo mantiene en alto y el botón lo baja.
- **SWD:** es el puerto de debug. Con una Pico como "debugprobe" podés poner breakpoints y ver variables. BOOTSEL alcanza para cargar firmware, pero SWD te salva cuando algo se cuelga.
- **Marcadores de sin conexión en los GPIO libres:** le dicen al ERC "lo dejé suelto a propósito". Sin el marcador, no se distingue de un olvido.

<details><summary>La placa nueva no aparece como pendrive al apretar BOOTSEL. Medís +3V3 bien. ¿Qué medirías después y por qué?</summary>

+1V1 (si el regulador interno no arranca, el núcleo no funciona), después que el cristal oscile (con osciloscopio en XOUT: sin cristal no hay USB) y que RUN esté en alto. Por eso pusimos test points en +1V1, RUN y QSPI_SS.
</details>

<details><summary>¿Por qué los 27 Ω van cerca del RP2040 y el ESD cerca del conector, si están en la misma línea?</summary>

Porque hacen cosas distintas. La terminación serie tiene que estar en la fuente de la señal (el driver del RP2040). La protección tiene que estar donde entra la descarga (el conector).
</details>

### Cómo salen las señales del RP2040 (ruteo final)

El problema del QFN es que entre pad y pad (0.4 mm de paso) no pasa ninguna pista, y abajo del chip hay desacoples y la pista ancha de +5V en B.Cu. Cada grupo sale por donde puede:

| Grupo | Salida | Camino |
|---|---|---|
| EXP_GP10–15 | pines de la derecha y de abajo, en F.Cu | Bus de 6 pistas hacia la derecha, sube por la zona libre (x≈106), cruza por arriba del IS31 y pasa J2 entre sus agujeros. Rodea J3 **por abajo** y entra por el lado derecho. |
| LED_SDB | abajo, F.Cu | Viaja con el bus y salta a B.Cu para llegar a R14 por debajo del cruce. |
| I2C_SDA / SCL, MIDI_TX | abajo, vía a B.Cu | Corren por B.Cu entre la pista de +5V y el riel CS8. SDA y SCL vuelven a F.Cu al lado de C28. MIDI_TX sigue hasta x=172 y sube a R16. |
| EXP_ADC0–2, GPIO25 | arriba, F.Cu | Suben entre C11 y C3, abren en abanico hacia el medio de SW1 y bajan por vías. GPIO25 pasa entre los pines 1 y 2 de J1 hacia R7. Los ADC van por B.Cu pegados al borde de arriba, bajan bajo la muesca de J2 y rodean J3 **por arriba**. |

**Por qué rodear J3:** el bus llega con el orden GP15…GP10 (de arriba a abajo), justo al revés de como están los pines del header. Entrando de frente, todas las pistas tendrían que cruzarse. Si das la vuelta al conector, el orden se invierte solo, sin vías. Los ADC rodean por arriba en B.Cu y los GP por abajo en F.Cu, así que no se pisan.

**Cambios hechos para que entre todo:**
- C9 se movió junto al MCU. RUN, SWD y SWCLK salen en abanico por la esquina superior derecha.
- C5 bajó 0.25 mm para que MIDI_TX salga del pin 2.
- C6 pasó a la fila de C4/C7/C8 (x 90.6). El pin 10 (IOVDD) baja directo a esa fila de +3V3, que ya tiene sus capacitores al lado.
- La pista de +5V en B.Cu bajo el MCU subió 0.3 mm (y=59.3) para dejar 3 carriles entre ella y el riel CS8.
- Las reglas `RP2040 fine pitch` (0.15 mm alrededor de U4) y `RP2040 +1V1 vs R8` en `midi-controller.kicad_dru`.

---

## 3. Entradas

### ¿Por qué registros de desplazamiento?

Tenés 32 entradas (8 encoders × 3 + 8 botones) y el RP2040 tiene 30 GPIO, y muchos ya están ocupados. Hay tres opciones:

| Opción | Pines | Contra |
|---|---|---|
| Directo a GPIO | 32 | No hay tantos |
| Matriz (filas × columnas) | ~12 | Se lee por barrido; con encoders, que cambian rápido y a la vez, se pierden pasos; necesita diodos para evitar "ghosting" |
| **74HC165 en cadena** | **3** | Una lectura serie, pero el SPI por hardware lee 32 bits en microsegundos |

El 74HC165 es un registro **paralelo-a-serie (PISO)**:

1. **`~{PL}` en bajo:** copia las 8 entradas a la vez (es una "foto").
2. **Clock:** las saca una por una por `Q7`.
3. Encadenando `Q7` de un chip con `DS` del siguiente, 4 chips se leen como un solo registro de 32 bits.

### Agrupar por zona de la placa, no por tipo de señal

Al principio repartimos las entradas por tipo: un chip para todas las A, otro para las B, otro para los switches y otro para los botones. En el esquemático quedaba prolijo, pero en el PCB cada chip juntaba señales de los 8 encoders, repartidos en 115 mm de ancho. Eso daba cuatro buses cruzando la placa entera.

El firmware lee 32 bits y los traduce con una tabla, así que le da igual qué bit es cuál. **La agrupación la decide la geometría:**

| Chip | Entradas | Ubicación |
|---|---|---|
| U6 | A/B de ENC1, 2, 5, 6 | entre las columnas de ENC1 y ENC2 |
| U7 | A/B de ENC3, 4, 7, 8 | entre las columnas de ENC3 y ENC4 |
| U8 | switches de ENC1, 2, 5, 6 + botones 1–4 | hueco entre ENC5 y ENC6 (debajo de U6) |
| U9 | switches de ENC3, 4, 7, 8 + botones 5–8 | hueco entre ENC7 y ENC8 (debajo de U7) |

Cadena: RP2040 ← U6 ← U8 ← U9 ← U7 (DS de U7 a GND). De paso, A y B de cada encoder quedan en bits vecinos del mismo byte, que es justo lo que necesita la decodificación de cuadratura.

**Lección general:** el esquemático se ordena para que se lea fácil. El PCB se ordena para que las pistas sean cortas. Cuando chocan, manda el PCB, y se vuelve al esquemático a reasignar pines (pin swapping).

**Pin swap aplicado en U6/U7 (ruteo de entradas).** Cada señal entra al 165 desde el lado de donde viene su encoder, por debajo del cuerpo del chip, y las redes de pull-up se rotaron 180° para que sus pads de señal queden enfrentados a los pines (una pista corta por señal). Las 4 resistencias de cada red son intercambiables, así que también se reasignaron:

| Chip | Pin 14 · 13 · 12 · 11 (D3·D2·D1·D0, de izquierda a derecha en la placa) | Pin 3 · 4 · 5 · 6 (D4·D5·D6·D7) |
|---|---|---|
| U6 | B1 · A1 · B2 · A2 | A5 · B5 · A6 · B6 |
| U7 | B3 · A3 · B4 · A4 | A7 · B7 · A8 · B8 |

RN1/RN3: pads 1–4 = B·A·B·A del encoder de la izquierda y después el de la derecha (1/3 y 2/4). RN2/RN4: pads 4·3·2·1 = A·B·A·B de los encoders 5/6 y 7/8.

**U8/U9: reubicados y reagrupados por zona.** Al principio estaban en el centro, encima del tronco de las líneas SW en In2 y entre los rieles CS de B.Cu: no se podía poner ni una vía debajo, y por F.Cu los costados estaban tapados por las pistas de cátodo de los anillos 6 y 7. Se movieron a los huecos libres de la fila de abajo (entre ENC5/ENC6 y entre ENC7/ENC8), donde In2 está libre, y cada uno junta las entradas de su mitad de la placa:

| Chip | Pin 14·13·12·11 (D3–D0), fila de arriba, de izquierda a derecha | Pin 3·4·5·6 (D4–D7), fila de abajo |
|---|---|---|
| U8 | SW1 · SW2 · SW5 · SW6 | BTN1 · BTN2 · BTN3 · BTN4 |
| U9 | SW3 · SW4 · SW7 · SW8 | BTN5 · BTN6 · BTN7 · BTN8 |

**Reasignación final por cercanía.** Al rutear, las salidas A/B de los anillos 6 y 8 quedaban bloqueadas por U8/U9, así que cada entrada se mandó al 165 más cercano. Esta tabla reemplaza a las dos de arriba:

| Chip | Fila de arriba: pin 14 · 13 · 12 · 11 | Fila de abajo: pin 3 · 4 · 5 · 6 |
|---|---|---|
| U6 | B1 · A1 · B2 · A2 | A5 · B5 · SW1 · SW2 |
| U7 | B3 · A3 · B4 · A4 | A7 · B7 · SW3 · SW4 |
| U8 | A6 · B6 · SW5 · SW6 | BTN1 · BTN2 · BTN3 · BTN4 |
| U9 | A8 · B8 · SW7 · SW8 | BTN5 · BTN6 · BTN7 · BTN8 |

A y B de cada encoder siguen en bits vecinos del mismo byte. El firmware tiene que leer el mapa de esta tabla.

RN5/RN7 (arriba) y RN6/RN8 (abajo) tienen cada pad de señal frente a su pin. **Acceso a las entradas:** cada pin tiene lugar para una vía debajo del cuerpo del chip y una pista por In2 que sale del grupo hacia arriba (fila de arriba) o hacia abajo (fila de abajo). LATCH y CLK bajan desde U6/U7 por In2 hasta vías debajo del cuerpo; Q7/DS salen por arriba en F.Cu. Todo verificado con DRC en una copia simulada. La cadena no cambia (RP2040 ← U6 ← U8 ← U9 ← U7): U8–U6 y U9–U7 quedan pegados, y el enlace largo es U8 ← U9.

### Pull-ups y lógica activa en bajo

Cada entrada tiene una pull-up a +3V3, y el botón o el encoder la llevan a GND. ¿Por qué al revés (activo en bajo)? Porque así el cable que sale al encoder o al botón es GND, y un GND suelto o tocado no hace daño. Además es la convención de casi todos los encoders.

### Debounce en firmware y no con capacitores

Los contactos mecánicos rebotan durante unos milisegundos. Para los encoders, la decodificación por **tabla de estados** ignora los rebotes: solo acepta transiciones válidas de la secuencia de cuadratura. Para los botones alcanza con pedir que el estado se mantenga unos ms. Sin capacitores la placa es más barata y la lectura más rápida.

<details><summary>¿Por qué <code>SR_CLK</code> puede necesitar una resistencia en serie y <code>SR_LATCH</code> no?</summary>

El clock cuenta flancos: si la pista larga hace que el flanco rebote (ringing), un chip puede ver dos flancos donde hubo uno y correr un bit. El latch es un nivel ("copiá ahora"), un rebote no cambia el resultado.
</details>

---

## 4. LEDs (IS31FL3741A)

### Para qué sirven

Los encoders giran sin tope, así que mirando la perilla no sabés en qué valor está. **El anillo de 15 LEDs muestra el valor**, como la aguja de un VU. Es bidireccional: si el DAW cambia el parámetro (cambiaste de canal, lo moviste con el mouse), le manda el valor nuevo al controlador por MIDI y el anillo salta a la posición correcta.

El LED RGB de cada botón muestra su estado con colores: prendido/apagado, mute, solo o el modo activo.

### Una matriz en vez de 144 pines

El IS31 tiene **9 pines SW**, que entregan corriente a los ánodos, y **39 pines CS**, que la absorben desde los cátodos. Prende una fila SW por vez, miles de veces por segundo, y el ojo ve todo encendido a la vez (persistencia de la visión). Con 8 + 18 pines manejás 144 LEDs.

| Fila | CS1–CS15 | CS16–CS18 |
|---|---|---|
| `LED_SWk` | anillo k (D**k**01–D**k**15) | RGB k (D**k**16) |

La numeración D305 = anillo 3, LED 5 no es casual: el script que ubique los LEDs en círculos en el PCB la va a usar.

**Consecuencia del barrido:** cada LED está prendido solo 1/8 (o 1/9) del tiempo. Su corriente promedio es la pico dividida por 8, y por eso la corriente pico es alta (19 mA) aunque el brillo sea moderado.

### Por qué +5V y no +3V3

Según el datasheet, el chip "come" hasta 0.75 V en el SW y 0.6 V en el CS. Un LED blanco tiene ~3 V de Vf, así que hacen falta 3 + 1.35 ≈ **4.35 V como mínimo**. Con 3.3 V no prenden. Además, así la corriente de los LEDs no pasa por el LDO, que no la soportaría (sección 1).

### Por qué la lógica sigue a 3.3 V

Las entradas del IS31 reconocen como "1" cualquier tensión **≥ 1.4 V**, aunque el chip esté alimentado a 5 V. El RP2040 lo maneja directo. Las **pull-ups del I2C van a +3V3**: si fueran a +5V, meterían 5 V en los pines del RP2040, que no los toleran.

### ISET = 20k (y no los 10k del datasheet)

> I_pico = 383 / R_ISET(kΩ) → 10k = 38 mA, 20k = 19 mA

Peor caso: una fila entera prendida al máximo, es decir, 18 canales × 19 mA ≈ **345 mA**. Con 10k serían 690 mA: un bug del firmware que prenda todo al arrancar haría saltar el polyfuse. **Es una protección por hardware contra errores de software.** El brillo se ajusta después por registro (GCC) sin tocar la placa.

### El resto
- **SDB con 100k a GND:** mientras el RP2040 arranca, sus pines flotan. El pull-down mantiene el chip apagado hasta que el firmware lo habilita.
- **ADDR a GND:** dirección I2C 0x30.
- **Desacople:** 1 µF + 100 nF en cada pin de alimentación, y 10 µF a la entrada. En cada cambio de fila la corriente salta de 0 a cientos de mA. El de 10 µF es el "tanque" que la entrega sin que caiga el +5V.
- **PCB:** el pad central lleva entre 16 y 25 vías a GND (lo pide el datasheet para disipar), y las pistas SW son más anchas que las CS, porque cada SW lleva la corriente de toda su fila.

<details><summary>Si el ojo ve los 144 LEDs prendidos, ¿por qué el consumo no es 144 × 19 mA?</summary>

Porque nunca están prendidos todos a la vez: en cada instante conduce una sola fila (como mucho 18 LEDs). Las otras 7 están apagadas. El consumo máximo es el de una fila.
</details>

<details><summary>Cambiás los LEDs blancos por rojos (Vf ≈ 2 V). ¿Qué pasa con el calor del IS31?</summary>

Sobran 5 − 2 − (caída del chip) ≈ 2 V más, que el CS disipa como calor. Con muchos rojos al máximo, el chip se calienta más. El datasheet sugiere resistencias en serie en esos casos, o bajar la corriente por registro.
</details>

---

## 5. La hoja raíz: cómo se conectan las hojas

```
Alimentación ──USB_DP/DM──► MCU ──SR_*──────► Entradas
                             ├──I2C, LED_SDB──► LEDs
                             └──MIDI_TX, EXP_*─► MIDI y expansión
```

- **Alimentación:** +5V, +3V3 y GND no aparecen como cables porque son símbolos globales.
- **Pines del MCU:** están agrupados por destino y alineados con su bloque, así cada cable es una línea recta. Un esquemático donde se ve qué habla con qué, sin cruces, se revisa en minutos.

---

## 6. Costo estimado

Con las tarifas publicadas de JLCPCB (economic PCBA): preparación ~USD 8, ~USD 0.0016 por soldadura y USD 1.5–3 por cada parte "Extended". El resto son estimaciones, no una cotización:

| Primer pedido: 5 PCB, 2 armadas | USD |
|---|---|
| PCB 4 capas 170×110 | 30–50 |
| Preparación + partes Extended | 25–40 |
| Componentes (2 placas) | ~20 |
| Panel frontal (5) | 10–20 |
| Envío | 20–35 |
| Encoders, teclas, perillas (a mano) | 25–40 |
| **Total** | **~150–210**, unos USD 75–100 por unidad |

En 20 a 50 unidades, la placa armada baja a USD 20–30. La mayor parte del costo de un prototipo son **costos fijos**: preparación, partes Extended y envío. El precio exacto sale al subir el BOM y el CPL a JLCPCB.

---

## 7. Cosas pendientes que salieron en el camino

- **Firmware, MIDI TRS:** el lazo MIDI toma unos 5 mA de GPIO0. Hay que subir la corriente de ese pin a 12 mA (por defecto entrega 4 mA). Si no, algunos sintes no leen bien.
- **Header de expansión:** los ADC aceptan como máximo 3.3 V. Un fader o un pote va entre +3V3 y GND, nunca a +5V.
- **Advertencias del ERC:** los símbolos de JLC del IS31 (y algunos otros) traen los tipos de pin sin definir. No es un error eléctrico; se limpia corrigiendo los tipos en la librería, como se hizo con el USBLC6.
- **Si volvés a bajar C7519 o C2940604 con `jlcget`,** sus símbolos vuelven a la versión original de JLC.
- **Colores del LED RGB:** no se sabe qué pin es R, G o B. No hace falta para el esquemático; se resuelve en el firmware durante el bring-up.
- **En el PCB:** desacoples pegados a sus pines, los 27 Ω pegados al RP2040, el USBLC6 pegado al conector, cobre en el pad de GND del LDO, y el par USB ruteado junto y con impedancia controlada (JLCPCB tiene calculadora para su stackup de 4 capas).

---

## 8. Notas para el firmware (salieron de la revisión final)

### Pines del RP2040

| GPIO | Señal | Periférico |
|---|---|---|
| 0 | MIDI_TX | UART0 TX, 31250 baudios. Subir la corriente del pin a 12 mA (ver sección 7). |
| 6 / 7 | I2C_SDA / I2C_SCL | I2C1, con pull-ups de 4k7 (R12, R13) en la placa. |
| 8 | LED_SDB | En alto enciende el IS31. R14 (100k a GND) lo mantiene apagado hasta que el firmware lo prende. |
| 10–15 | EXP_GP10–15 | Header de expansión J3. |
| 16 | SR_DATA | Entrada: sale del Q7 de U6. |
| 17 | SR_LATCH | Pulso en bajo = captura las 32 entradas (~PL de los 74HC165). |
| 18 | SR_CLK | Cada flanco de subida corre la cadena un bit. |
| 25 | LED de estado | GPIO25 → R7 (1k) → D2 (rojo) → GND. En alto = prendido. |
| 26–28 | EXP_ADC0–2 | ADC del header. Máximo 3.3 V. |

### Lectura de las entradas (74HC165)

- **Orden de la cadena:** `RP2040 ← U6 ← U8 ← U9 ← U7 ← GND`. No sigue el orden de los números: quedó así por el ruteo.
- **Leer exactamente 32 bits.** Después del bit 32 entra el GND del DS de U7, que se lee como "apretado".
- **Activo en bajo:** 0 = contacto cerrado. Todas las entradas tienen pull-up de 10k (RN1–RN8).
- **Secuencia:** LATCH en bajo y de nuevo en alto → el primer bit ya está en SR_DATA, se lee **antes** del primer clock → después, clock y lectura 31 veces más.
- **Orden de los bits** (Q7 saca primero D7):

| Bits | Chip | D7 → D0 |
|---|---|---|
| 0–7 | U6 | ENC_SW2, ENC_SW1, ENC_B5, ENC_A5, ENC_B1, ENC_A1, ENC_B2, ENC_A2 |
| 8–15 | U8 | BTN_4, BTN_3, BTN_2, BTN_1, ENC_A6, ENC_B6, ENC_SW5, ENC_SW6 |
| 16–23 | U9 | BTN_8, BTN_7, BTN_6, BTN_5, ENC_A8, ENC_B8, ENC_SW7, ENC_SW8 |
| 24–31 | U7 | ENC_SW4, ENC_SW3, ENC_B7, ENC_A7, ENC_B3, ENC_A3, ENC_B4, ENC_A4 |

  Conviene armar una tabla en el código que traduzca "bit N → encoder/botón X". Así el resto del firmware no depende del ruteo.

### IS31FL3741A

- **Dirección I2C: 0x30** (7 bits, ADDR a GND). Algunos ejemplos la escriben como 0x60, que es la misma corrida un bit con el bit de R/W. El SDK del RP2040 usa la de 7 bits.
- **Primero, un escaneo I2C:** tiene que aparecer 0x30. Si no aparece, revisar SDB (tiene que estar en alto), los pull-ups y la soldadura del QFN.
- **Usamos SW1–SW8 de los 9 que tiene.** Buscar en la hoja de datos si el registro de configuración permite barrer solo 8: si sigue barriendo la SW9 vacía, cada LED queda prendido menos tiempo y se ve más tenue. *(Sin verificar: confirmarlo en la hoja de datos.)*
- **Matriz:** SW = fila, entrega corriente y se barre. CS = columna, absorbe corriente con PWM propio. Cada LED es un par (SWx, CSy). Por ejemplo, D301 = (SW3, CS1).
- **LED RGB:** no se sabe qué pin es R, G o B. Se averigua en el arranque prendiendo un CS a la vez.
