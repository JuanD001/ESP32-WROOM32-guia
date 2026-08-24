"""
=====================================================
 ESP32 - Demostración de IPC en MicroPython
 (Equivalente al ejemplo original en C con FreeRTOS)
=====================================================

Botón 1 -> GPIO13   LED 1 -> GPIO25
Botón 2 -> GPIO14   LED 2 -> GPIO26

Ambos botones usan pull-up interno:
    nivel BAJO  (0) = botón presionado
    nivel ALTO  (1) = botón liberado
"""

from machine import Pin
import time
import _thread

# =====================================================
# DEFINICIÓN DE PINES
# =====================================================

BUTTON1_PIN = 13
BUTTON2_PIN = 14
LED1_PIN = 25
LED2_PIN = 26

button1 = Pin(BUTTON1_PIN, Pin.IN, Pin.PULL_UP)
button2 = Pin(BUTTON2_PIN, Pin.IN, Pin.PULL_UP)

led1 = Pin(LED1_PIN, Pin.OUT)
led2 = Pin(LED2_PIN, Pin.OUT)

led1.value(0)
led2.value(0)


# =====================================================
# OBJETOS DE COMUNICACIÓN ENTRE HILOS
# (equivalentes "manuales" a QueueHandle_t y
#  SemaphoreHandle_t de FreeRTOS)
# =====================================================

sensor_queue = []
queue_lock = _thread.allocate_lock()

# Los "semáforos" de botón se simulan con banderas booleanas
# que la ISR solo enciende (nunca hace trabajo pesado dentro
# de la interrupción, igual que en el código en C).
button1_flag = False
button2_flag = False


# =====================================================
# RUTINAS DE INTERRUPCIÓN (ISR)
# =====================================================

def button1_isr(pin):
    global button1_flag
    button1_flag = True


def button2_isr(pin):
    global button2_flag
    button2_flag = True


button1.irq(trigger=Pin.IRQ_FALLING | Pin.IRQ_RISING, handler=button1_isr)
button2.irq(trigger=Pin.IRQ_FALLING | Pin.IRQ_RISING, handler=button2_isr)


# =====================================================
# TAREA PRODUCTORA
# Genera datos de "sensor" y los coloca en la cola
# =====================================================

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


# =====================================================
# TAREA CONSUMIDORA
# Retira datos de la cola cuando hay disponibles
# =====================================================

def consumer_task():
    while True:
        if sensor_queue:
            queue_lock.acquire()
            value = sensor_queue.pop(0)
            queue_lock.release()

            print("[CONSUMER] Received Sensor Data:", value)

        time.sleep_ms(50)


# =====================================================
# TAREA DE BOTONES
# Revisa las banderas activadas por las ISR y actualiza
# el estado de los LED (equivalente a button1_task /
# button2_task en el código original)
# =====================================================

def button_task():
    global button1_flag, button2_flag

    while True:
        if button1_flag:
            button1_flag = False
            state = button1.value()

            if state == 0:
                led1.value(1)
                print("[BUTTON 1] PRESSED -> LED1 ON")
            else:
                led1.value(0)
                print("[BUTTON 1] RELEASED -> LED1 OFF")

        if button2_flag:
            button2_flag = False
            state = button2.value()

            if state == 0:
                led2.value(1)
                print("[BUTTON 2] PRESSED -> LED2 ON")
            else:
                led2.value(0)
                print("[BUTTON 2] RELEASED -> LED2 OFF")

        time.sleep_ms(20)


# =====================================================
# PROGRAMA PRINCIPAL
# =====================================================

print()
print("========================================")
print(" ESP32 MicroPython IPC Demonstration")
print("========================================")

print("[INIT] GPIO initialized")
print("[INIT] Sensor queue created")
print("[INIT] Button interrupts attached")

# El segundo núcleo ejecuta el hilo productor.
# NOTA: MicroPython en ESP32 permite lanzar un hilo adicional
# con _thread, pero no ofrece tareas con prioridad como FreeRTOS.
_thread.start_new_thread(producer_task, ())
_thread.start_new_thread(consumer_task, ())

print("[INIT] Producer/Consumer threads started")
print("----------------------------------------")
print("Button 1 : GPIO13")
print("Button 2 : GPIO14")
print("LED 1    : GPIO25")
print("LED 2    : GPIO26")
print("----------------------------------------")
print("[READY] System ready!")

# El hilo principal se encarga de los botones
button_task()
