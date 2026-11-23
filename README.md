# Práctica 3: Controlador de Máquina Expendedora en Arduino UNO

## 1. Introducción y Objetivo
El objetivo de esta práctica es diseñar e implementar el controlador software en tiempo real para una máquina expendedora automática sobre **Arduino UNO**. El sistema gestiona de forma concurrente y no bloqueante la interacción con el usuario (pantalla LCD 16x2, joystick analógico, pulsador externo e indicadores LED) y la lectura de sensores ambientales y de proximidad (DHT11 y ultrasonidos HC-SR04), cumpliendo estrictamente con los modos de operación **Arranque**, **Servicio** y **Administración**.

## 2. Esquema de Conexiones y Montaje Hardware

### Tabla de Pines (Arduino UNO)
| Componente | Pin del Módulo | Pin Arduino UNO | Tipo de Señal / Configuración |
| :--- | :--- | :--- | :--- |
| **Pulsador Externo** | Señal | `D2` (INT0) | Entrada digital con `INPUT_PULLUP` (Interrupción Hardware) |
| **Pantalla LCD 16x2** | RS, E, D4, D5, D6, D7 | `D12, D11, D5, D4, D3, D7` | Salidas digitales (Bus de 4 bits) |
| **LED2 (Progreso/Admin)** | Ánodo (+) | `D6` (~PWM) | Salida analógica PWM (Timer0, 0-255) |
| **Ultrasonidos HC-SR04** | TRIG | `D8` | Salida digital |
| **Ultrasonidos HC-SR04** | ECHO | `D9` | Entrada digital (`pulseIn`) |
| **Sensor Clima DHT11** | DATA | `D10` | Entrada/Salida digital (Librería DHT) |
| **Joystick (Botón)** | SW / SEL | `D13` | Entrada digital con `INPUT_PULLUP` |
| **Joystick (Eje X)** | VRx | `A0` | Entrada analógica (ADC 10 bits: 0-1023) |
| **Joystick (Eje Y)** | VRy | `A1` | Entrada analógica (ADC 10 bits: 0-1023) |
| **LED1 (Estado/Arranque)** | Ánodo (+) | `A3` | Salida digital (`HIGH` / `LOW`) |

### Esquemático del Circuito (Fritzing / Tinkercad)
<img width="2160" height="2160" alt="maquina-expendedora" src="https://github.com/user-attachments/assets/a23d55b0-ff57-4c0e-b86a-a25e848f7738" />

### Fotografías del Montaje Real
<p align="center">
  <img width="200" alt="Montaje 1" src="https://github.com/user-attachments/assets/c67ae388-9dc1-4d29-ae26-a75401d28639" />
  <img width="200" alt="Montaje 2" src="https://github.com/user-attachments/assets/d678e304-b04a-44a0-bfd2-f5bfbca45cff" />
  <img width="200" alt="Montaje 3" src="https://github.com/user-attachments/assets/02a130bf-a9cd-4c46-81d2-c31f9ddef1f2" />
  <img width="200" alt="Montaje 4" src="https://github.com/user-attachments/assets/514d1f86-ed1e-459c-86b3-67e56df81103" />
</p>
<p align="center">
  <img width="410" alt="Detalle LCD y Protoboard" src="https://github.com/user-attachments/assets/5f43d36e-42f6-4c65-ab2a-699efb3f2b30" />
  <img width="410" alt="Vista general superior" src="https://github.com/user-attachments/assets/a228ff42-333e-4399-9a91-376318eb260e" />
</p>

## 3. Arquitectura Software y Recursos Utilizados

El software se ha modularizado en ficheros independientes (`Maquina.ino`, `Sistema.h`, `Clima.cpp`, `Ultrasonidos.cpp`, `Joystick.cpp` y `MaquinaEstados.cpp`) para separar la gestión de cada sensor de la lógica principal y evitar bloqueos de la CPU.

### 3.1. Multitarea Cooperativa (`ArduinoThread`)
* **Recurso:** Librerías `Thread.h` y `ThreadController.h`.
* **Justificación:** Al disponer de un único núcleo (*mono-core*), el uso de `delay()` congelaría el sistema impidiendo atender el botón de cancelación o el joystick mientras se prepara una bebida. Con `ThreadController` se ejecutan **cuatro hilos periódicos** basados en `millis()`, agrupados según el ritmo físico de cada tarea:
  * `tDht` (**2000 ms**): Consulta temperatura y humedad en el DHT11 respetando su tiempo mínimo de muestreo de 2 segundos.
  * `tUltra` (**120 ms**): Lanza el pulso de disparo de $10\ \mu\text{s}$ en `TRIG`, mide el eco con `pulseIn(ECHO_PIN, HIGH)` y calcula la distancia multiplicando por `0.01715`.
  * `tJoy` (**60 ms**): Lee los ejes analógicos X/Y mediante el ADC de 10 bits (con calibración inicial y zona muerta de reposo para evitar saltos múltiples en los menús) y el botón central del joystick.
  * `tSis` (**100 ms**): Ejecuta la máquina de estados principal (`switch`) y actualiza la pantalla LCD y los LEDs. No se han creado hilos adicionales para la LCD o los LEDs porque son salidas pasivas que dependen del estado actual; centralizarlas en `tSis` evita condiciones de carrera al escribir en pantalla y ahorra memoria RAM.

### 3.2. Interrupciones Hardware y Antirrebote (`attachInterrupt`)
* **Recurso:** Interrupción externa en el pin **D2 (INT0)** en modo `CHANGE` junto con variables calificadas como `volatile` (`flagCancelar`, `flagAdmin`, `tPulsacion`, `tUltimoCambio`, `botonPulsado`).
* **Justificación:** El pulsador externo permite cancelar una operación (pulsación de **2 a 3 segundos**) o entrar/salir del modo Administración desde cualquier estado (pulsación de **$\ge 5$ segundos**). La interrupción captura el flanco exacto al instante sin depender del ciclo de los hilos:
  1. Aplica un filtro antirrebote (*debounce*) por software de **40 ms** comparando `millis() - tUltimoCambio`.
  2. En el flanco de bajada (`LOW`, con `INPUT_PULLUP`), registra `tPulsacion = millis()`.
  3. En el flanco de subida (`HIGH`), calcula la duración y activa únicamente el *flag* booleano `volatile` correspondiente, dejando que el hilo `tSis` realice el cambio de estado y actualice la LCD de forma segura.

### 3.3. Temporizador Watchdog (`avr/wdt.h`)
* **Recurso:** Watchdog Timer configurado a **4 segundos** (`wdt_disable()`, `wdt_enable(WDTO_4S)` y `wdt_reset()` en cada iteración del `loop()`).
* **Justificación:** Actúa como mecanismo interno anti-bloqueos. Mientras el programa se ejecuta con normalidad, `wdt_reset()` reinicia la cuenta atrás de 4 segundos en cada vuelta del `loop()`. Si un fallo en un sensor dejara el código detenido más de ese tiempo, el temporizador llega a cero y reinicia automáticamente el microcontrolador para recuperar el servicio.


## 4. Máquina de Estados Implementada
1. **Modo Arranque (`ARRANQUE`):** Muestra `CARGANDO` en la LCD mientras hace parpadear `LED1` 3 veces en intervalos de 1 segundo (`1000 ms`). Al finalizar, apaga `LED1` y pasa a `SERVICIO`.
2. **Modo Servicio (`SERVICIO`):**
   * `ESPERANDO`: Muestra `ESPERANDO CLIENTE` hasta que el ultrasonidos detecta presencia a menos de `100 cm`.
   * `CLIMA`: Muestra temperatura y humedad durante `5000 ms`.
   * `CATALOGO`: Permite desplazarse con el joystick (Arriba/Abajo) por los 5 productos y confirmar con clic.
   * `PREPARANDO`: Calcula una duración aleatoria entre `4000` y `8000 ms` mediante `4000 + (millis() % 4001)` e incrementa el brillo de `LED2` en el pin PWM `6` sumando `5` cada `100 ms` hasta el máximo (`255`).
   * `RETIRAR`: Muestra `RETIRE BEBIDA` durante `3000 ms` antes de volver a `ESPERANDO`.
3. **Modo Administración (`ADMIN`):** Enciende `LED1` y `LED2` al máximo y despliega el menú técnico (Temperatura/Humedad en vivo, Distancia en vivo, Contador de segundos desde el arranque mediante `millis() / 1000`, y Edición de precios en tramos de `0.05 €` con opción de guardar con clic o descartar moviendo el joystick a la izquierda).
