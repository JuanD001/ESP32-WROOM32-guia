# Ejemplo MicroPython — Demostración de IPC (botones, LEDs y cola productor/consumidor)

Archivo: [`CODIGO.py`](./CODIGO.py)

Este ejemplo es la **versión en MicroPython** del programa original escrito en C con ESP-IDF/FreeRTOS que se encuentra en [`../c/CODIGO.c`](../c/CODIGO.c). Reproduce el mismo comportamiento: dos botones controlan dos LEDs mediante interrupciones, mientras un hilo "productor" genera datos y un hilo "consumidor" los procesa a través de una cola compartida.

## Conexiones de hardware

| Función | Pin (GPIO) |
|---|---|
| Botón 1 (a GND, con pull-up interno) | GPIO13 |
| Botón 2 (a GND, con pull-up interno) | GPIO14 |
| LED 1 | GPIO25 |
| LED 2 | GPIO26 |

> Con pull-up interno activado: nivel **0 = botón presionado**, nivel **1 = botón liberado**.

## Simulación

🔗 **Simulación en Wokwi:** [wokwi.com/projects/473197046830440449](https://wokwi.com/projects/473197046830440449)

![Circuito ESP32 con botones y LEDs](../../img/wokwi-circuito-botones-leds.png)

> El mismo circuito se usa también en la versión en C, disponible en [`../c/`](../c/).

---

## Explicación del código por bloques

### 1. Importaciones

```python
from machine import Pin
import time
import _thread
```

- `machine.Pin`: es el módulo estándar de MicroPython para controlar pines GPIO (entradas, salidas, resistencias de pull-up/pull-down, interrupciones).
- `time`: para las pausas (`sleep`, `sleep_ms`), equivalente a `vTaskDelay()` en FreeRTOS.
- `_thread`: permite lanzar una función en un hilo aparte, aprovechando el segundo núcleo del ESP32. Es lo más cercano que tiene MicroPython a `xTaskCreate()`.

### 2. Configuración de pines

```python
button1 = Pin(BUTTON1_PIN, Pin.IN, Pin.PULL_UP)
button2 = Pin(BUTTON2_PIN, Pin.IN, Pin.PULL_UP)
led1 = Pin(LED1_PIN, Pin.OUT)
led2 = Pin(LED2_PIN, Pin.OUT)
```

Configura los botones como entradas con resistencia de pull-up interna (así no se necesita una resistencia externa) y los LEDs como salidas. Esto reemplaza toda la función `gpio_init()` del código en C, donde había que llenar una estructura `gpio_config_t` manualmente para cada grupo de pines.

### 3. "Cola" y "semáforos" simulados

```python
sensor_queue = []
queue_lock = _thread.allocate_lock()

button1_flag = False
button2_flag = False
```

MicroPython no incluye de fábrica una cola (`QueueHandle_t`) ni semáforos binarios (`SemaphoreHandle_t`) como los de FreeRTOS. Se simulan así:

- **Cola →** una lista de Python (`sensor_queue`) protegida por un candado (`_thread.allocate_lock()`), para que el productor y el consumidor no la modifiquen al mismo tiempo.
- **Semáforo de botón →** una variable booleana (`button1_flag` / `button2_flag`). La interrupción solo la enciende; la tarea de botones la apaga cuando ya procesó el evento.

### 4. Rutinas de interrupción (ISR)

```python
def button1_isr(pin):
    global button1_flag
    button1_flag = True
```

Se ejecuta automáticamente cuando cambia el nivel eléctrico del pin del botón. **A propósito, no hace nada más que cambiar una bandera**: en MicroPython (igual que con `IRAM_ATTR` en C) las interrupciones deben ser lo más cortas posible, sin `print()` ni operaciones pesadas, para no bloquear el sistema.

```python
button1.irq(trigger=Pin.IRQ_FALLING | Pin.IRQ_RISING, handler=button1_isr)
```

Esto reemplaza tanto a `gpio_install_isr_service()` como a `gpio_isr_handler_add()` del código en C: en una sola línea se registra la interrupción para ambos flancos (subida y bajada), equivalente a `GPIO_INTR_ANYEDGE`.

### 5. Tarea productora

```python
def producer_task():
    count = 0
    while True:
        count += 1
        print("[PRODUCER] Sending sensor data:", count)
        queue_lock.acquire()
        sensor_queue.append(count)
        queue_lock.release()
        print("[PRODUCER] Data sent successfully")
        time.sleep(1)
```

Cada segundo incrementa un contador y lo agrega a la lista compartida, usando el candado para evitar condiciones de carrera. Es el equivalente directo a `producer_task()` y `xQueueSend()` en el código original.

### 6. Tarea consumidora

```python
def consumer_task():
    while True:
        if sensor_queue:
            queue_lock.acquire()
            value = sensor_queue.pop(0)
            queue_lock.release()
            print("[CONSUMER] Received Sensor Data:", value)
        time.sleep_ms(50)
```

Revisa constantemente si hay datos disponibles en la lista y, de haberlos, retira el primero (`pop(0)`, comportamiento FIFO, igual que una cola real). Equivale a `consumer_task()` y `xQueueReceive()`.

### 7. Tarea de botones

```python
def button_task():
    global button1_flag, button2_flag
    while True:
        if button1_flag:
            button1_flag = False
            state = button1.value()
            if state == 0:
                led1.value(1)
                ...
```

Revisa continuamente las banderas activadas por las interrupciones. Cuando detecta que una bandera está encendida, la apaga y lee el estado real del botón para decidir si el LED correspondiente debe encender o apagar. Esto reemplaza `button1_task()`/`button2_task()`, que en el código en C esperaban (`xSemaphoreTake`) a que la ISR les diera la señal.

### 8. Programa principal

```python
_thread.start_new_thread(producer_task, ())
_thread.start_new_thread(consumer_task, ())
...
button_task()
```

Lanza el productor y el consumidor cada uno en su propio hilo (usando el segundo núcleo), y deja la tarea de botones corriendo en el hilo principal. Es la versión simplificada de la sección final de `app_main()`, donde se creaban las cuatro tareas con `xTaskCreate()` y prioridades específicas (3, 2, 4 y 4).

---

## Diferencias clave frente a la versión en C

| Aspecto | Versión en C (FreeRTOS) | Versión en MicroPython |
|---|---|---|
| Cola de datos | `xQueueCreate()` / `xQueueSend()` / `xQueueReceive()` nativos | Lista de Python + `_thread.allocate_lock()` |
| Semáforo de botón | `xSemaphoreCreateBinary()` / `xSemaphoreGiveFromISR()` | Variable booleana simple |
| Tareas concurrentes | `xTaskCreate()` con **prioridades** (2, 3, 4, 4) | `_thread.start_new_thread()`, **sin prioridades** |
| Ejecución paralela real | Sí, en ambos núcleos de forma independiente | Limitada por el GIL de MicroPython; menos paralelismo real |
| Registro de interrupciones | `gpio_install_isr_service()` + `gpio_isr_handler_add()` | Una sola línea: `Pin.irq()` |
| Cantidad de código | Más extenso y detallado | Más corto y directo |

## Cómo ejecutarlo

1. Flashea el firmware de MicroPython en tu ESP32-WROOM-32 (ver la sección de referencias del README principal del repositorio).
2. Sube este archivo (`CODIGO.py`) a la placa, por ejemplo con **Thonny**, **ampy** o **rshell**.
3. Ejecútalo o renómbralo como `main.py` para que corra automáticamente al iniciar la placa.
4. Abre el monitor serie para ver los mensajes de `[PRODUCER]`, `[CONSUMER]` y `[BUTTON]`.
