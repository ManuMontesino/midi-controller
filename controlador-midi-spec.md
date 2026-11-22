# Controlador USB-MIDI — Especificación v1

> **Objetivo:** pieza de portfolio que muestre hardware + firmware + software.
> **Mensaje:** "Una plataforma probada para hacer controladores a medida en semanas, no meses."

---

## 1. Qué es "terminado" (no se agrega nada fuera de esta lista)

- [ ] PCB propio con RP2040, ensamblado y funcionando
- [ ] 8 encoders con anillo de LEDs + 8 botones con LED RGB
- [ ] Enumera como USB-MIDI + HID (teclado/media) sin drivers en Windows, macOS y Linux
- [ ] Salida MIDI TRS funcionando contra un sinte o interfaz
- [ ] Feedback desde el DAW: cambiar un parámetro en pantalla actualiza el anillo
- [ ] Configurador web funcionando y publicado en GitHub Pages
- [ ] Configuración guardada en flash (sobrevive a desenchufar)
- [ ] Repo con README, fotos, renders, BOM y video demo de 1–2 min

Todo lo demás va a la **lista v2** (sección 9).

---

## 2. Hardware

### Diagrama de bloques

```
USB-C ──► ESD + polyfuse ──► LDO 3.3V ──┬──► RP2040 + flash QSPI + cristal 12 MHz
                                        │       │
                                        │       ├─ SPI ──► 4× 74HC165 ◄── 8 encoders (A, B, SW) + 8 botones
                                        │       ├─ I2C ──► IS31FL3741A ──► 8 anillos (15 LEDs) + 8 LEDs RGB
                                        │       ├─ UART TX ──► jack TRS (MIDI out, Type A)
                                        │       └─ GPIO libres ──► header de expansión
                                        └──► (5V de USB para el driver de LEDs si hace falta)
```

### Componentes principales

| Bloque | Componente | Notas |
|---|---|---|
| MCU | RP2040 (chip suelto, no módulo Pico) | Muestra más oficio de PCB. JLCPCB lo ensambla. |
| Flash | W25Q16JV (2 MB, QSPI) | Sobra para firmware + config |
| Cristal | 12 MHz (el mismo del diseño de referencia) | Con 1k en serie y capacitores de carga según la referencia |
| Regulador | LDO 3.3V de ≥500 mA (ej. AP2112K-3.3) | |
| USB | USB-C receptáculo de 16 pines | 2× 5.1k en CC1/CC2, 27 Ω en serie en D+/D−, USBLC6-2 para ESD, polyfuse de 500 mA |
| Entradas | 4× 74HC165 en cadena = 32 entradas | 8 encoders × 3 (A, B, SW) = 24 + 8 botones = 32, justo |
| Encoders | EC11 / Bourns PEC11R, 24 detents, con pulsador | Filtro RC recomendado por Bourns (10k / 10k / 10nF) en A y B |
| Botones | Switches tipo teclado mecánico (MX o Choc) | LED RGB SMD de montaje invertido debajo, keycap translúcido |
| LEDs | IS31FL3741A (driver matricial I2C, hasta 351 LEDs) | 8 anillos × 15 LEDs = 120 + 8 RGB × 3 = 24 → 144 LEDs. PWM por LED, poco consumo, un solo chip |
| MIDI | Jack TRS 3.5 mm, Type A | Para 3.3V según la especificación MIDI: 33 Ω de 3.3V a Ring (pin 4) y 10 Ω de TX a Tip (pin 5). Sleeve = blindaje |
| Debug | Header SWD (3 pines) + botones BOOTSEL y RESET | |
| Expansión | Header con GPIOs libres + ADC + 3V3 + GND | Clave para la idea de plataforma: variantes sin rediseñar la placa base |

**Para la parte del RP2040:** seguí el documento *"Hardware design with RP2040"* de Raspberry Pi al pie de la letra (desacople, VREG, cristal, flash, USB). Es la parte con menos margen para improvisar.

### Pinout propuesto

| GPIO | Función | Periférico |
|---|---|---|
| GP0 | MIDI TX → TRS | UART0 TX |
| GP6 | SDA → IS31FL3741A | I2C1 |
| GP7 | SCL → IS31FL3741A | I2C1 |
| GP8 | SDB (shutdown) del IS31FL3741A | GPIO |
| GP16 | QH del último 74HC165 | SPI0 RX |
| GP17 | SH/LD̅ (latch) de los 74HC165 | GPIO |
| GP18 | CLK de los 74HC165 | SPI0 SCK |
| GP10–GP15, GP26–GP28 | Header de expansión | Libres (GP26–28 con ADC) |
| GP25 | LED de estado | GPIO |

### Presupuesto de consumo

USB 2.0 da 500 mA. Apuntá a **< 300 mA en el peor caso**: limitá la corriente global del IS31FL3741A por registro y **medila** en el bring-up. Un anillo que muestra un valor tiene pocos LEDs prendidos, así que el caso típico es mucho menor.

### Mecánica

- 2 filas de 4 encoders arriba y 1 fila de 8 botones abajo.
- **Panel frontal hecho como un segundo PCB** (sin cobre, con serigrafía y máscara negra o blanca). Es barato en JLCPCB y queda muy profesional. Base impresa en 3D o de acrílico, con separadores.
- Ojo: en JLCPCB el precio salta arriba de 100 × 100 mm. Calculá el tamaño antes de cerrar el layout.

---

## 3. Firmware

**Stack:** Pico SDK (C, CMake) + TinyUSB. Queda más profesional en el portfolio que Arduino.

### Reparto entre núcleos

```
Core 1 (tiempo real)                     Core 0 (lógica + USB)
─────────────────────                    ─────────────────────────────────
timer 2 kHz                              loop:
 └─ leer 74HC165 por SPI                  ├─ tud_task()            (USB)
 └─ decodificar cuadratura                ├─ consumir eventos de la cola
 └─ debounce de botones                   ├─ mapping → MIDI / HID
 └─ aceleración de encoders               ├─ MIDI IN (USB) → feedback a los LEDs
 └─ push evento ──► queue_t ─────────►    ├─ SysEx → configuración
                                          └─ refresco de LEDs por I2C (~60 Hz)
```

### Módulos

| Módulo | Responsabilidad |
|---|---|
| `input_scan` | Lectura de los 165, decodificación de cuadratura por tabla de estados (robusta al rebote), debounce y aceleración |
| `mapping` | Convierte eventos en mensajes según la configuración de cada control |
| `midi_out` | Envía a USB-MIDI y a UART (TRS) a la vez |
| `midi_in` | Recibe CC/notas del DAW → actualiza el valor interno y el anillo (bidireccional) |
| `hid` | Reportes de teclado + consumer control (teclas de media), con report IDs |
| `leds` | Driver IS31FL3741A + modos de anillo: punto, barra, centrado (pan) |
| `config` | Struct en el último sector de flash, con versión, CRC y valores por defecto |
| `sysex` | Protocolo con el configurador web (sección 4) |

### Configuración por control

- **Tipo:** CC, nota, program change, tecla HID, tecla de media
- **Canal MIDI, número, mínimo y máximo**
- **Modo de encoder:** absoluto (0–127, sincronizado con el feedback del DAW) o relativo (offset binario 64 ± n, para DAWs que lo soportan)
- **Comportamiento del botón:** momentáneo o toggle
- **LED:** color (botones) y estilo de anillo

### USB

- Dispositivo compuesto: **MIDI + HID** (en TinyUSB, `CFG_TUD_MIDI` y `CFG_TUD_HID`). Opcional en builds de desarrollo: **CDC** para logs por puerto serie.
- **VID/PID:** para hardware open source, pedí un PID gratis en **pid.codes** (VID 0x1209). Queda mejor que usar el de Raspberry Pi.
- **Ojo con Windows:** cachea los descriptores. Si cambiás la composición USB, subí `bcdDevice` o cambiá el PID mientras desarrollás, o vas a perseguir errores fantasma.

### Guardado en flash

El RP2040 ejecuta desde la flash (XIP), así que mientras borrás o escribís **no puede correr código desde la flash en ningún núcleo**. Usá `flash_safe_execute()` (Pico SDK 2.x), que se encarga de pausar el otro núcleo y las interrupciones. Guardá solo al recibir `SAVE`, no en cada cambio, para no gastar la flash.

---

## 4. Protocolo SysEx

- **ID de fabricante:** `0x7D` (reservado para uso no comercial o educativo; ideal para el prototipo).
- **Formato:** `F0 7D <device_id> <cmd> <payload…> F7`
- Todos los bytes del payload tienen que ser de 7 bits. Los valores mayores a 127 (por ejemplo keycodes HID) van **partidos en dos bytes** (nibble alto / nibble bajo).

| Cmd | Dirección | Uso |
|---|---|---|
| `0x01 GET_INFO` | web → dispositivo | Pide versión de firmware, versión de config y cantidad de controles |
| `0x02 INFO` | dispositivo → web | Respuesta |
| `0x03 GET_CONTROL <id>` | web → dispositivo | Pide la configuración de un control |
| `0x04 CONTROL <id> <datos>` | ambos | Configuración de un control (respuesta o escritura) |
| `0x05 SAVE` | web → dispositivo | Escribe la configuración en flash |
| `0x06 FACTORY_RESET` | web → dispositivo | Vuelve a los valores por defecto |
| `0x07 IDENTIFY <id>` | web → dispositivo | Hace parpadear el LED de ese control |
| `0x08 EVENT <id>` | dispositivo → web | "Se tocó este control" (para que la web lo resalte en vivo) |

Documentalo en `docs/protocol.md`: que el protocolo esté escrito muestra oficio.

---

## 5. Configurador web

- **Tecnología:** Web MIDI API con `sysex: true` + página estática (vanilla JS, o Svelte/React con Vite).
- **Hosting:** GitHub Pages (requiere HTTPS, que Pages ya da). **El link al configurador es parte del portfolio.**
- **Compatibilidad:** Chrome, Edge y Firefox. **Safari no soporta Web MIDI**: avisalo en la página.

**Funciones de la v1:**
1. Detectar el controlador y leer su configuración
2. Dibujo del layout real del controlador. Clic en un control → panel de edición
3. Resaltado en vivo: girás una perilla física y se ilumina en pantalla (`EVENT`)
4. Guardar en el dispositivo
5. Exportar e importar presets en JSON
6. **Modo demo sin hardware**, para que un cliente pueda probarlo desde el portfolio

---

## 6. Estructura del repo

```
midi-controller/
├── README.md            ← fotos, video, features, cómo compilar
├── hardware/            ← proyecto KiCad, Gerbers, BOM, pick & place, renders
├── firmware/            ← Pico SDK (CMake)
├── configurator/        ← web app
└── docs/
    ├── protocol.md      ← protocolo SysEx
    ├── bringup.md       ← mediciones, problemas encontrados y cómo se resolvieron
    └── img/
```

`bringup.md` vale oro para el portfolio: muestra cómo depurás, que es lo que un cliente quiere saber.

---

## 7. Plan (≈ 3 semanas)

El envío de PCBs a Argentina es el cuello de botella, así que **el pedido sale lo antes posible y el firmware se hace en paralelo.**

| Semana | Hardware | Firmware / web |
|---|---|---|
| 1 | Esquemático + layout + revisión. **Pedido a JLCPCB a fin de semana** (placa principal + panel frontal) | Protoboard: Raspberry Pi Pico + 1 74HC165 + 2 encoders + breakout del IS31FL3741 (Adafruit tiene uno). USB-MIDI enumerando, encoders leyendo |
| 2 | (esperando el envío) | Mapping, HID, MIDI TRS, feedback del DAW, config en flash, SysEx. Configurador web contra la protoboard |
| 3 | Bring-up de la placa: alimentación → RP2040 → USB → entradas → LEDs → MIDI. Medir consumo | Ajustes finales, fotos, video, README, publicar en GitHub Pages |

**Regla:** si la semana 3 se estira, se publica lo que hay y lo pendiente va a "known issues" en el README.

---

## 8. Riesgos conocidos

| Riesgo | Mitigación |
|---|---|
| Error en el circuito del RP2040 que impide arrancar | Copiar el diseño de referencia; poner test points en 3V3, 1V1, RUN y QSPI_SS |
| Consumo de LEDs excede USB | Límite global de corriente por registro + medición en el bring-up |
| Encoders saltando valores | Filtro RC + tabla de estados + escaneo a 2 kHz |
| Windows no reconoce el cambio de descriptores | Cambiar `bcdDevice` o PID en cada cambio de composición |
| Cuelgue al guardar en flash | `flash_safe_execute()` |
| Demora o aduana del envío | Pedir en la semana 1; desarrollar todo en la protoboard |

---

## 9. Lista v2 (no tocar hasta terminar la v1)

- MIDI de 14 bits (NRPN / CC pares)
- Bancos (varias páginas de configuración)
- MIDI IN por TRS + thru
- Entrada para pedal de expresión (ya está el ADC en el header)
- Scripts o perfiles para DAWs específicos (Ableton, Bitwig, Reaper)
- Variantes con el header de expansión: button box para sim racing, panel de streaming
