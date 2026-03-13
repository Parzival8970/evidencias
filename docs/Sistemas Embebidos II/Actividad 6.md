# Session — Wi-Fi + MQTT: Two ESP32 for LED State and Brightness Control
Implement communication between **two ESP32 boards** using the **MQTT protocol** so that buttons and potentiometers on each board control the **ON/OFF state** and **brightness** of LEDs on the other board.

---

## 1) Activity Goals

- [ ] Connect **two ESP32 boards** through MQTT communication.
- [ ] Use **two push buttons** on each ESP32 to control the ON/OFF state of LEDs on the other ESP32.
- [ ] Use a **potentiometer** on each ESP32 to control the **brightness** of LEDs on the other board.
- [ ] Implement correct **publish/subscribe communication** using MQTT topics.
- [ ] Demonstrate **bidirectional device-to-device control** in an IoT system.
---

## 2) Materials & Setup

### BOM (Bill of Materials)

| # | Item | Qty | Link/Source | Cost (MXN) | Notes |
|---|------|-----|------------|------------|------|
| 1 | ESP32 Dev Board | 2 | Local Store / Amazon / MercadoLibre | ____ | One publisher device and one subscriber device |
| 2 | LED | 4 | Local electronics store | ____ | Connected to GPIO 12 and GPIO 13 |
| 3 | Push Button | 4 | Local electronics store | ____ | Connected to GPIO 4 and GPIO 5 |
| 4 | Potentiometer | 2 | Local electronics store | ____ | Used for brightness control |
| 5 | Resistors | 4 | Local electronics store | ____ | Recommended for LEDs |
| 6 | Jumper wires | Several | Local electronics store | ____ | Circuit connections |

---

### Tools / Software

- **Framework:** ESP-IDF + FreeRTOS  
- **Build / Flash / Monitor:** `idf.py build flash monitor`  
- **MQTT Protocol** used for communication between devices  
- **Operating System:** Windows  

---

### Wiring / Safety

- **LED1 → GPIO 12**
- **LED2 → GPIO 13**
- **Button1 → GPIO 4** 
- **Button2 → GPIO 5** 
- **Potentiometer → ADC Channel 0**
- Ensure both ESP32 boards share **correct wiring and stable power**.

---

## 3) Procedure

1. Configured GPIO pins for LEDs, buttons, and the potentiometer (ADC).
2. Assigned a **device ID** to each ESP32 to differentiate the devices.
3. Defined MQTT topics for button states and brightness values.
4. Configured LEDs as outputs and buttons as inputs with internal pull-up resistors.
5. Configured PWM (LEDC) to control LED brightness.
6. Configured the ADC to read the potentiometer value.
7. Implemented a task that reads the buttons and potentiometer and publishes their values through MQTT.
8. Subscribed to MQTT topics to receive messages from the other ESP32.
9. Used received messages to control the LED state and brightness.
10. Verified that **both ESP32 boards can control each other's LEDs remotely using MQTT**.

---

## 4) Data, Tests & Evidence

## Videos (Evidence)

<iframe width="315" height="560"
src="https://www.youtube.com/embed/zdBQWtp8cvQ"
title="MQTT Dual ESP32 LED Control"
frameborder="0"
allowfullscreen></iframe>

**Video 1.** Demonstration of two ESP32 boards communicating through MQTT to control LED state and brightness remotely.

---

## 5) Analysis

### System Architecture

This system uses **two ESP32 boards connected through MQTT topics**, where **both devices act as publishers and subscribers at the same time**.

The communication works as follows:

- When a **button is pressed on ESP32 A**, a message is published to the MQTT broker.
- ESP32 B receives that message through its subscription and **turns its LED on or off**.
- When the **potentiometer is rotated**, the analog value is sent through MQTT.
- The receiving ESP32 converts that value into a **PWM duty cycle** to control LED brightness.

## 6) Code

### Full Firmware

~~~c
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include "esp_log.h"
#include "esp_event.h"
#include "mqtt_client.h"
#include "driver/gpio.h"
#include "driver/ledc.h"
#include "esp_adc/adc_oneshot.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

static const char *TAG = "MQTT_LAB";

#define GPIO_LED1 GPIO_NUM_12
#define GPIO_LED2 GPIO_NUM_13

#define GPIO_BUTTON1 GPIO_NUM_4
#define GPIO_BUTTON2 GPIO_NUM_5

#define TEAM_ID "teamDyC"

/* CAMBIAR SEGUN EL ESP32 */

#define LOCAL_DEVICE_ID "c6_01"
#define REMOTE_DEVICE_ID "c6_02"

/* Topics */

#define TOPIC_BUTTON1 "ibero/ei2/" TEAM_ID "/" LOCAL_DEVICE_ID "/button1"
#define TOPIC_BUTTON2 "ibero/ei2/" TEAM_ID "/" LOCAL_DEVICE_ID "/button2"
#define TOPIC_POT "ibero/ei2/" TEAM_ID "/" LOCAL_DEVICE_ID "/brightness"

#define REMOTE_BUTTON1 "ibero/ei2/" TEAM_ID "/" REMOTE_DEVICE_ID "/button1"
#define REMOTE_BUTTON2 "ibero/ei2/" TEAM_ID "/" REMOTE_DEVICE_ID "/button2"
#define REMOTE_POT "ibero/ei2/" TEAM_ID "/" REMOTE_DEVICE_ID "/brightness"

static esp_mqtt_client_handle_t client = NULL;
static adc_oneshot_unit_handle_t adc1_handle;

/* ESTADO DE LOS LEDS */

static int led1_state = 0;
static int led2_state = 0;

/* ============================= */

static void publisher_task(void *pvParameters)
{
    int last_button1 = 1;
    int last_button2 = 1;

    while (1)
    {
        int button1 = !gpio_get_level(GPIO_BUTTON1);
        int button2 = !gpio_get_level(GPIO_BUTTON2);

        if (button1 != last_button1)
        {
            if (button1)
                esp_mqtt_client_publish(client, REMOTE_BUTTON1, "1", 0, 0, 0);
            else
                esp_mqtt_client_publish(client, REMOTE_BUTTON1, "0", 0, 0, 0);

            last_button1 = button1;
        }

        if (button2 != last_button2)
        {
            if (button2)
                esp_mqtt_client_publish(client, REMOTE_BUTTON2, "1", 0, 0, 0);
            else
                esp_mqtt_client_publish(client, REMOTE_BUTTON2, "0", 0, 0, 0);

            last_button2 = button2;
        }

        int adc_raw;
        adc_oneshot_read(adc1_handle, ADC_CHANNEL_0, &adc_raw);

        char msg[10];
        sprintf(msg, "%d", adc_raw);

        esp_mqtt_client_publish(client, REMOTE_POT, msg, 0, 0, 0);

        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

/* ============================= */

static void mqtt_event_handler(void *handler_args, esp_event_base_t base,
                               int32_t event_id, void *event_data)
{
    esp_mqtt_event_handle_t event = event_data;

    switch (event_id)
    {

    case MQTT_EVENT_CONNECTED:

        ESP_LOGI(TAG, "MQTT connected");

        esp_mqtt_client_subscribe(client, TOPIC_BUTTON1, 0);
        esp_mqtt_client_subscribe(client, TOPIC_BUTTON2, 0);
        esp_mqtt_client_subscribe(client, TOPIC_POT, 0);

        break;

    case MQTT_EVENT_DATA:
    {

        if (event->topic_len == strlen(TOPIC_BUTTON1) &&
            memcmp(event->topic, TOPIC_BUTTON1, event->topic_len) == 0)
        {
            if (event->data[0] == '1')
                led1_state = 1;
            else
                led1_state = 0;
        }

        else if (event->topic_len == strlen(TOPIC_BUTTON2) &&
                 memcmp(event->topic, TOPIC_BUTTON2, event->topic_len) == 0)
        {
            if (event->data[0] == '1')
                led2_state = 1;
            else
                led2_state = 0;
        }

        else if (event->topic_len == strlen(TOPIC_POT) &&
                 memcmp(event->topic, TOPIC_POT, event->topic_len) == 0)
        {
            char data_str[10];

            memcpy(data_str, event->data, event->data_len);
            data_str[event->data_len] = '\0';

            int brightness = atoi(data_str);

            int duty = (brightness * 8191) / 4095;

            if (led1_state)
                ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, duty);
            else
                ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 0);

            ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);

            if (led2_state)
                ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_1, duty);
            else
                ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_1, 0);

            ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_1);
        }

        break;
    }

    default:
        break;
    }
}

/* ============================= */

esp_err_t mqtt_app_start(const char *broker_uri)
{

    gpio_config_t led_conf = {
        .pin_bit_mask = (1ULL << GPIO_LED1) | (1ULL << GPIO_LED2),
        .mode = GPIO_MODE_OUTPUT};

    gpio_config(&led_conf);

    gpio_config_t button_conf = {
        .pin_bit_mask = (1ULL << GPIO_BUTTON1) | (1ULL << GPIO_BUTTON2),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE};

    gpio_config(&button_conf);

    ledc_timer_config_t ledc_timer = {
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .timer_num = LEDC_TIMER_0,
        .duty_resolution = LEDC_TIMER_13_BIT,
        .freq_hz = 5000};

    ledc_timer_config(&ledc_timer);

    ledc_channel_config_t ledc_channel1 = {
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .channel = LEDC_CHANNEL_0,
        .timer_sel = LEDC_TIMER_0,
        .gpio_num = GPIO_LED1,
        .duty = 0};

    ledc_channel_config(&ledc_channel1);

    ledc_channel_config_t ledc_channel2 = {
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .channel = LEDC_CHANNEL_1,
        .timer_sel = LEDC_TIMER_0,
        .gpio_num = GPIO_LED2,
        .duty = 0};

    ledc_channel_config(&ledc_channel2);

    adc_oneshot_unit_init_cfg_t init_config1 = {
        .unit_id = ADC_UNIT_1};

    adc_oneshot_new_unit(&init_config1, &adc1_handle);

    adc_oneshot_chan_cfg_t config = {
        .bitwidth = ADC_BITWIDTH_DEFAULT,
        .atten = ADC_ATTEN_DB_12};

    adc_oneshot_config_channel(adc1_handle, ADC_CHANNEL_0, &config);

    esp_mqtt_client_config_t cfg = {
        .broker.address.uri = broker_uri};

    client = esp_mqtt_client_init(&cfg);

    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);

    esp_mqtt_client_start(client);

    xTaskCreate(publisher_task, "publisher", 4096, NULL, 5, NULL);
    
    return ESP_OK;
}

~~~