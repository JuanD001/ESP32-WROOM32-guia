# Ejemplo en C (ESP-IDF / FreeRTOS) — Demostración de IPC

Archivo: [`ipc_demo.c`](./CODIGO.c)

Programa original en C sobre ESP-IDF que usa **FreeRTOS** para coordinar cuatro tareas concurrentes:

- `producer_task`: genera datos cada segundo y los envía a una cola (`xQueueSend`).
- `consumer_task`: recibe y procesa los datos de la cola (`xQueueReceive`).
- `button1_task` / `button2_task`: esperan una señal de semáforo binario, entregada desde la interrupción de cada botón, para encender o apagar su LED correspondiente.

Usa dos botones (GPIO13, GPIO14) con pull-up interno e interrupciones por ambos flancos, y dos LEDs de salida (GPIO25, GPIO26).

## Simulación

🔗 **Simulación en Wokwi:** [wokwi.com/projects/471963365838170113](https://wokwi.com/projects/471963365838170113)

![Circuito ESP32 con botones y LEDs](../../img/wokwi-circuito-botones-leds.png)

> El mismo circuito (botones en GPIO13/GPIO14, LEDs con resistencia en GPIO25/GPIO26) se usa tanto para esta versión en C como para la versión en MicroPython.

La versión equivalente en MicroPython, con explicación detallada del código, está en [`../micropython/`](../micropython/).
