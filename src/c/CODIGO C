#include <stdio.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "freertos/semphr.h"

#include "driver/gpio.h"


/* =====================================================
 * GPIO DEFINITIONS
 * ===================================================== */

#define BUTTON1_GPIO    GPIO_NUM_13
#define BUTTON2_GPIO    GPIO_NUM_14

#define LED1_GPIO       GPIO_NUM_25
#define LED2_GPIO       GPIO_NUM_26


/* =====================================================
 * IPC OBJECTS
 * ===================================================== */

QueueHandle_t sensorQueue;

SemaphoreHandle_t button1Semaphore;
SemaphoreHandle_t button2Semaphore;


/* =====================================================
 * BUTTON 1 INTERRUPT SERVICE ROUTINE
 * ===================================================== */

static void IRAM_ATTR button1_isr_handler(void *arg)
{
    BaseType_t higherPriorityTaskWoken = pdFALSE;

    xSemaphoreGiveFromISR(
        button1Semaphore,
        &higherPriorityTaskWoken
    );

    if (higherPriorityTaskWoken)
    {
        portYIELD_FROM_ISR();
    }
}


/* =====================================================
 * BUTTON 2 INTERRUPT SERVICE ROUTINE
 * ===================================================== */

static void IRAM_ATTR button2_isr_handler(void *arg)
{
    BaseType_t higherPriorityTaskWoken = pdFALSE;

    xSemaphoreGiveFromISR(
        button2Semaphore,
        &higherPriorityTaskWoken
    );

    if (higherPriorityTaskWoken)
    {
        portYIELD_FROM_ISR();
    }
}


/* =====================================================
 * PRODUCER TASK
 *
 * Generates sensor data and sends it to the queue.
 * ===================================================== */

void producer_task(void *pvParameters)
{
    int count = 0;

    while (1)
    {
        count++;

        printf("[PRODUCER] Sending sensor data: %d\n", count);

        if (xQueueSend(
                sensorQueue,
                &count,
                portMAX_DELAY) == pdPASS)
        {
            printf("[PRODUCER] Data sent successfully\n");
        }

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}


/* =====================================================
 * CONSUMER TASK
 *
 * Receives sensor data from the queue.
 * ===================================================== */

void consumer_task(void *pvParameters)
{
    int received_val;

    while (1)
    {
        if (xQueueReceive(
                sensorQueue,
                &received_val,
                portMAX_DELAY))
        {
            printf(
                "[CONSUMER] Received Sensor Data: %d\n",
                received_val
            );
        }
    }
}


/* =====================================================
 * BUTTON 1 TASK
 *
 * Waits for semaphore from Button 1 ISR.
 * ===================================================== */

void button1_task(void *pvParameters)
{
    while (1)
    {
        /*
         * Wait until Button 1 interrupt gives
         * the semaphore.
         */
        if (xSemaphoreTake(
                button1Semaphore,
                portMAX_DELAY))
        {
            /*
             * Because internal pull-up is used:
             *
             * GPIO13 = LOW  -> button pressed
             * GPIO13 = HIGH -> button released
             */

            int button_state =
                gpio_get_level(BUTTON1_GPIO);

            if (button_state == 0)
            {
                gpio_set_level(LED1_GPIO, 1);

                printf(
                    "[BUTTON 1] PRESSED -> LED1 ON\n"
                );
            }
            else
            {
                gpio_set_level(LED1_GPIO, 0);

                printf(
                    "[BUTTON 1] RELEASED -> LED1 OFF\n"
                );
            }
        }
    }
}


/* =====================================================
 * BUTTON 2 TASK
 *
 * Waits for semaphore from Button 2 ISR.
 * ===================================================== */

void button2_task(void *pvParameters)
{
    while (1)
    {
        /*
         * Wait until Button 2 interrupt gives
         * the semaphore.
         */
        if (xSemaphoreTake(
                button2Semaphore,
                portMAX_DELAY))
        {
            /*
             * Internal pull-up:
             *
             * GPIO14 = LOW  -> pressed
             * GPIO14 = HIGH -> released
             */

            int button_state =
                gpio_get_level(BUTTON2_GPIO);

            if (button_state == 0)
            {
                gpio_set_level(LED2_GPIO, 1);

                printf(
                    "[BUTTON 2] PRESSED -> LED2 ON\n"
                );
            }
            else
            {
                gpio_set_level(LED2_GPIO, 0);

                printf(
                    "[BUTTON 2] RELEASED -> LED2 OFF\n"
                );
            }
        }
    }
}


/* =====================================================
 * GPIO INITIALIZATION
 * ===================================================== */

void gpio_init(void)
{
    /*
     * -------------------------------------------------
     * Configure LEDs
     * -------------------------------------------------
     */

    gpio_config_t led_config = {
        .pin_bit_mask =
            (1ULL << LED1_GPIO) |
            (1ULL << LED2_GPIO),

        .mode = GPIO_MODE_OUTPUT,

        .pull_up_en = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,

        .intr_type = GPIO_INTR_DISABLE
    };

    gpio_config(&led_config);


    /* Initially turn both LEDs OFF */

    gpio_set_level(LED1_GPIO, 0);
    gpio_set_level(LED2_GPIO, 0);


    /*
     * -------------------------------------------------
     * Configure buttons
     * -------------------------------------------------
     *
     * Button 1:
     *
     * GPIO13 ---- BUTTON ---- GND
     *
     * Button 2:
     *
     * GPIO14 ---- BUTTON ---- GND
     *
     * Internal pull-ups are enabled.
     */

    gpio_config_t button_config = {
        .pin_bit_mask =
            (1ULL << BUTTON1_GPIO) |
            (1ULL << BUTTON2_GPIO),

        .mode = GPIO_MODE_INPUT,

        .pull_up_en = GPIO_PULLUP_ENABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,

        /*
         * Interrupt on BOTH edges:
         *
         * HIGH -> LOW  = button pressed
         * LOW  -> HIGH = button released
         */
        .intr_type = GPIO_INTR_ANYEDGE
    };

    gpio_config(&button_config);
}


/* =====================================================
 * MAIN APPLICATION
 * ===================================================== */

void app_main(void)
{
    printf("\n");
    printf("========================================\n");
    printf(" ESP32 FreeRTOS IPC Demonstration\n");
    printf("========================================\n");


    /* -------------------------------------------------
     * Initialize GPIO
     * ------------------------------------------------- */

    gpio_init();

    printf("[INIT] GPIO initialized\n");


    /* -------------------------------------------------
     * Create Queue
     * ------------------------------------------------- */

    sensorQueue = xQueueCreate(
        10,
        sizeof(int)
    );

    if (sensorQueue == NULL)
    {
        printf("[ERROR] Queue creation failed!\n");
        return;
    }

    printf("[INIT] Sensor queue created\n");


    /* -------------------------------------------------
     * Create Semaphores
     * ------------------------------------------------- */

    button1Semaphore = xSemaphoreCreateBinary();

    button2Semaphore = xSemaphoreCreateBinary();

    if (button1Semaphore == NULL ||
        button2Semaphore == NULL)
    {
        printf("[ERROR] Semaphore creation failed!\n");
        return;
    }

    printf("[INIT] Button semaphores created\n");


    /* -------------------------------------------------
     * Install GPIO ISR Service
     * ------------------------------------------------- */

    esp_err_t ret = gpio_install_isr_service(0);

    if (ret != ESP_OK)
    {
        printf(
            "[ERROR] Failed to install ISR service\n"
        );
        return;
    }

    printf("[INIT] GPIO ISR service installed\n");


    /* -------------------------------------------------
     * Attach Button 1 ISR
     * ------------------------------------------------- */

    ret = gpio_isr_handler_add(
        BUTTON1_GPIO,
        button1_isr_handler,
        NULL
    );

    if (ret != ESP_OK)
    {
        printf(
            "[ERROR] Failed to attach Button 1 ISR\n"
        );
        return;
    }


    /* -------------------------------------------------
     * Attach Button 2 ISR
     * ------------------------------------------------- */

    ret = gpio_isr_handler_add(
        BUTTON2_GPIO,
        button2_isr_handler,
        NULL
    );

    if (ret != ESP_OK)
    {
        printf(
            "[ERROR] Failed to attach Button 2 ISR\n"
        );
        return;
    }

    printf("[INIT] Button interrupts attached\n");


    /* -------------------------------------------------
     * Create Producer Task
     * Priority = 3
     * ------------------------------------------------- */

    xTaskCreate(
        producer_task,
        "Producer Task",
        2048,
        NULL,
        3,
        NULL
    );


    /* -------------------------------------------------
     * Create Consumer Task
     * Priority = 2
     * ------------------------------------------------- */

    xTaskCreate(
        consumer_task,
        "Consumer Task",
        2048,
        NULL,
        2,
        NULL
    );


    /* -------------------------------------------------
     * Create Button 1 Task
     * Priority = 4
     * ------------------------------------------------- */

    xTaskCreate(
        button1_task,
        "Button1 Task",
        2048,
        NULL,
        4,
        NULL
    );


    /* -------------------------------------------------
     * Create Button 2 Task
     * Priority = 4
     * ------------------------------------------------- */

    xTaskCreate(
        button2_task,
        "Button2 Task",
        2048,
        NULL,
        4,
        NULL
    );


    printf("[INIT] All FreeRTOS tasks created\n");

    printf("----------------------------------------\n");
    printf("Button 1 : GPIO13\n");
    printf("Button 2 : GPIO14\n");
    printf("LED 1    : GPIO25\n");
    printf("LED 2    : GPIO26\n");
    printf("----------------------------------------\n");

    printf("[READY] System ready!\n");


    /* -------------------------------------------------
     * Keep app_main alive
     * ------------------------------------------------- */

    while (1)
    {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
