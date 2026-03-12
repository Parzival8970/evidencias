# Lab de Programación Wi-Fi: Control mediante MQTT

**Propósito de la sesión:**  
El objetivo general de esta sesión de laboratorio es desarrollar un sistema embebido basado en ESP32 con conectividad Wi-Fi y comunicación mediante el protocolo MQTT para el control remoto de dispositivos y el monitoreo de sensores.

En esta práctica se utilizó el protocolo MQTT, el cual es un protocolo de mensajería ligero basado en el modelo **publish/subscribe**, donde los dispositivos envían o reciben información a través de un **broker MQTT**.

---

## 1) Objetivos de la Actividad    
_Para este laboratorio se implementará un sistema que permita monitorear variables simuladas y controlar un actuador utilizando el protocolo MQTT._  

1) Lectura de valores de temperatura (simulada con potenciómetro)  
2) Lectura de valores de humedad (simulada con potenciómetro)  
3) Publicación de datos en tópicos MQTT  
4) Control remoto de un motor mediante mensajes MQTT  
---

## 2) Materiales y Configuración

**BOM (Lista de Materiales)**

|#|Artículo|Cantidad|Notas|
|---------|--------|------|--------|
|1|ESP32|1|Microcontrolador con Wi-Fi|
|2|Motor|1|Actuador controlado remotamente|
|3|Potenciómetro (Humedad)|1|Simula sensor de humedad|
|4|Potenciómetro (Temperatura)|1|Simula sensor de temperatura|

---

**_Herramientas / Software_**   

* _SO/Entorno: Windows_  
* _Editores: VS Code / Arduino IDE / ESP-IDF_  
* _Cliente MQTT: MQTT Explorer_  
* _Broker MQTT para la comunicación entre el ESP32 y el cliente_  

**_Cableado / Seguridad_**  

* _Alimentación de la placa: USB 5 V desde la PC host_  
* _Los potenciómetros se conectan a entradas analógicas del ESP32_  
* _El motor se conecta a una salida controlada por el microcontrolador_  
* _Verificar conexiones antes de energizar el sistema_  
---

## 3) Procedimiento   

* _**Paso 1:** Conectar el ESP32 al entorno de desarrollo._  
* _**Paso 2:** Conectar los potenciómetros a entradas analógicas para simular los sensores de temperatura y humedad._  
* _**Paso 3:** Conectar el motor a una salida controlada del ESP32._  
* _**Paso 4:** Programar el ESP32 para conectarse a una red Wi-Fi._  
* _**Paso 5:** Establecer conexión con un broker MQTT._  
* _**Paso 6:** Publicar los valores de temperatura y humedad en tópicos MQTT independientes._  
* _**Paso 7:** Utilizar MQTT Explorer para enviar comandos de control al ESP32._  
---

## 4) Análisis

_La implementación de un sistema basado en ESP32 con MQTT demuestra cómo un dispositivo embebido puede integrarse fácilmente en arquitecturas de Internet de las Cosas._

_El protocolo MQTT permite una comunicación eficiente entre dispositivos mediante un modelo de publicación y suscripción, lo que facilita el intercambio de datos entre sensores, actuadores y aplicaciones externas._

_El uso de potenciómetros como sensores simulados permite probar el sistema sin necesidad de sensores reales, representando variables físicas como temperatura y humedad._

_Además, el uso de una herramienta como **MQTT Explorer** facilita la visualización de los mensajes publicados y permite enviar comandos de control en tiempo real._

_La separación de los datos en diferentes tópicos permite organizar mejor la información del sistema, facilitando el monitoreo individual de cada variable y el control del actuador._

_Este tipo de arquitectura permite desarrollar sistemas de monitoreo y control remoto, donde dispositivos físicos pueden ser gestionados a través de una red utilizando protocolos ligeros diseñados para IoT._

---

## 5) Codigo
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"

#include "esp_system.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_log.h"
#include "esp_netif.h"
#include "nvs_flash.h"
#include "mqtt_client.h"

#include "driver/ledc.h"
#include "driver/gpio.h"
#include "esp_adc/adc_oneshot.h"

#define WIFI_SSID      "iPhone de Jose maria"
#define WIFI_PASS      "123456789"

#define TEAM_ID    "ASUS"
#define DEVICE_ID  "c6_01"
#define TOPIC_CMD   "ibero/ei2/" TEAM_ID "/" DEVICE_ID "/cmd"
#define TOPIC_TLM   "ibero/ei2/" TEAM_ID "/" DEVICE_ID "/telemetry"
#define TOPIC_STAT  "ibero/ei2/" TEAM_ID "/" DEVICE_ID "/status"

#define TOPIC_MOTOR   "ibero/cmd/motor"
#define TOPIC_SPEED   "ibero/cmd/speed"

#define ENA_GPIO   4
#define IN1_GPIO   18
#define IN2_GPIO   19

#define TEMP_ADC_CHANNEL ADC_CHANNEL_0
#define HUM_ADC_CHANNEL  ADC_CHANNEL_1

static EventGroupHandle_t wifi_event_group;
#define WIFI_CONNECTED_BIT BIT0

static const char *TAG = "IBERO_MQTT";

static esp_mqtt_client_handle_t client = NULL;

static int current_speed = 0;
static int motor_state = 0;

static void wifi_event_handler(void* arg,
                               esp_event_base_t event_base,
                               int32_t event_id,
                               void* event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START)
        esp_wifi_connect();
    else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED)
        esp_wifi_connect();
    else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP)
        xEventGroupSetBits(wifi_event_group, WIFI_CONNECTED_BIT);
}

void wifi_init_sta(void)
{
    wifi_event_group = xEventGroupCreate();

    esp_netif_init();
    esp_event_loop_create_default();
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    esp_wifi_init(&cfg);

    esp_event_handler_instance_register(WIFI_EVENT,
                                        ESP_EVENT_ANY_ID,
                                        &wifi_event_handler,
                                        NULL,
                                        NULL);

    esp_event_handler_instance_register(IP_EVENT,
                                        IP_EVENT_STA_GOT_IP,
                                        &wifi_event_handler,
                                        NULL,
                                        NULL);

    wifi_config_t wifi_config = {
        .sta = {
            .ssid = WIFI_SSID,
            .password = WIFI_PASS
        }
    };

    esp_wifi_set_mode(WIFI_MODE_STA);
    esp_wifi_set_config(WIFI_IF_STA, &wifi_config);
    esp_wifi_start();

    xEventGroupWaitBits(wifi_event_group,
                        WIFI_CONNECTED_BIT,
                        pdFALSE,
                        pdFALSE,
                        portMAX_DELAY);
}

static void motor_init()
{
    gpio_set_direction(IN1_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_direction(IN2_GPIO, GPIO_MODE_OUTPUT);

    ledc_timer_config_t timer = {
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .timer_num = LEDC_TIMER_0,
        .duty_resolution = LEDC_TIMER_8_BIT,
        .freq_hz = 5000,
        .clk_cfg = LEDC_AUTO_CLK
    };

    ledc_timer_config(&timer);

    ledc_channel_config_t channel = {
        .gpio_num = ENA_GPIO,
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .channel = LEDC_CHANNEL_0,
        .timer_sel = LEDC_TIMER_0,
        .duty = 0
    };

    ledc_channel_config(&channel);
}

static void set_motor_speed(int speed)
{
    ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, speed);
    ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
}

static void motor_forward()
{
    gpio_set_level(IN1_GPIO, 1);
    gpio_set_level(IN2_GPIO, 0);
}

static void motor_stop()
{
    gpio_set_level(IN1_GPIO, 0);
    gpio_set_level(IN2_GPIO, 0);
    set_motor_speed(0);
}

static void sensor_task(void *pv)
{
    adc_oneshot_unit_handle_t adc_handle;

    adc_oneshot_unit_init_cfg_t init_config = {
        .unit_id = ADC_UNIT_1,
    };

    adc_oneshot_new_unit(&init_config, &adc_handle);

    adc_oneshot_chan_cfg_t config = {
        .bitwidth = ADC_BITWIDTH_DEFAULT,
        .atten = ADC_ATTEN_DB_12,
    };

    adc_oneshot_config_channel(adc_handle, TEMP_ADC_CHANNEL, &config);
    adc_oneshot_config_channel(adc_handle, HUM_ADC_CHANNEL, &config);

    int last_temp = -100;
    int last_hum = -100;

    while (1)
    {
        int raw_temp;
        int raw_hum;

        adc_oneshot_read(adc_handle, TEMP_ADC_CHANNEL, &raw_temp);
        adc_oneshot_read(adc_handle, HUM_ADC_CHANNEL, &raw_hum);

        int temperature = (raw_temp * 50) / 4095;
        int humidity = (raw_hum * 100) / 4095;

        if (abs(temperature - last_temp) >= 3 || abs(humidity - last_hum) >= 3)
        {
            printf("Temperatura: %d °C\n", temperature);
            printf("Humedad: %d %%\n", humidity);

            char msg[64];
            sprintf(msg, "temp:%d,hum:%d", temperature, humidity);

            if (client != NULL)
                esp_mqtt_client_publish(client, TOPIC_TLM, msg, 0, 1, 0);

            last_temp = temperature;
            last_hum = humidity;
        }

        vTaskDelay(pdMS_TO_TICKS(300));
    }
}

static void mqtt_event_handler(void *handler_args,
                               esp_event_base_t base,
                               int32_t event_id,
                               void *event_data)
{
    esp_mqtt_event_handle_t event = event_data;

    switch (event->event_id)
    {

    case MQTT_EVENT_CONNECTED:

        printf("MQTT Conectado\n");

        esp_mqtt_client_subscribe(client, TOPIC_CMD, 1);
        esp_mqtt_client_subscribe(client, TOPIC_MOTOR, 1);
        esp_mqtt_client_subscribe(client, TOPIC_SPEED, 1);

        esp_mqtt_client_publish(client, TOPIC_STAT, "online", 0, 1, 0);

        break;

    case MQTT_EVENT_DATA:

        printf("Topic=%.*s Data=%.*s\n",
               event->topic_len, event->topic,
               event->data_len, event->data);

        if (strncmp(event->topic, TOPIC_MOTOR, event->topic_len) == 0)
        {
            if (strncmp(event->data, "ON", event->data_len) == 0)
            {
                motor_state = 1;
                motor_forward();
                set_motor_speed(current_speed);
            }
            else if (strncmp(event->data, "OFF", event->data_len) == 0)
            {
                motor_state = 0;
                motor_stop();
            }
        }

        if (strncmp(event->topic, TOPIC_SPEED, event->topic_len) == 0)
        {
            int speed = atoi(event->data);

            if (speed >= 0 && speed <= 255)
            {
                current_speed = speed;

                if (motor_state)
                    set_motor_speed(speed);
            }
        }

        break;

    case MQTT_EVENT_DISCONNECTED:

        printf("MQTT Desconectado\n");

        break;

    default:
        break;
    }
}

void mqtt_app_start(const char *broker_uri)
{
    motor_init();

    xTaskCreate(sensor_task, "sensor_task", 4096, NULL, 5, NULL);

    esp_mqtt_client_config_t cfg = {
        .broker.address.uri = broker_uri,
        .session.keepalive = 30
    };

    client = esp_mqtt_client_init(&cfg);

    esp_mqtt_client_register_event(client,
                                   ESP_EVENT_ANY_ID,
                                   mqtt_event_handler,
                                   NULL);

    esp_mqtt_client_start(client);
}

void app_main(void)
{
    nvs_flash_init();
    wifi_init_sta();
    mqtt_app_start("mqtt://test.mosquitto.org:1883");
}
```
## 6) Archivos y Multimedia

<div style="text-align:center; margin-top:20px;">
  <iframe 
    width="560" 
    height="315" 
    src="https://www.youtube.com/embed/14tYYR74VIg" 
    title="Demostración del sistema"
    frameborder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen>
  </iframe>
</div>