# BLE Programming Lab.

**Session Purpose:**  
The general goal of this lab session is to develop a system based on the ESP32-C6 using Bluetooth Low Energy (BLE), implementing communication through GATT services and characteristics to enable data exchange, device control, and real-time interaction with a client device.

---

## Lab 1

---

### Exercise 1.1 — Echo the Write Back on Read

#### 1) Activity Objectives    
_For this exercise, a system will be implemented that allows storing received data and returning it in a subsequent read._  

1) Declare a global buffer to store data  
2) Store the data received from the client  
3) Implement the handling of the read event  
4) Respond with the stored data  

---

#### 2) Materials & Setup

**BOM (Bill of Materials)**

|#|Item|Qty|Notes|
|---------|--------|------|--------|
|1|ESP32-C6|1|Microcontroller with BLE|
|2|USB Cable|1|Power supply and programming|
|3|Smartphone|1|BLE client device|

---

**_Tools / Software_**   

* _OS/Environment: Windows_  
* _Editors: VS Code / ESP-IDF_  
* _BLE Scanner: nRF Connect or similar_  

**_Wiring / Safety_**  

* _Board power supply: USB connection_  
* _Verify the connection before loading the system_  

---

#### 3) Procedure   

* _**Step 1:** Declare and initialize a global buffer._  
* _**Step 2:** Store the data received in the write event._  
* _**Step 3:** Implement the read event to return the data._  
* _**Step 4:** Compile, flash, and test with a BLE client._  

---

#### 4) Analysis

_The system allows storing the data sent by a BLE client and using it as a response in subsequent reads._  

_The implementation is based on the use of a global buffer where the data received during the write event is stored._  

_The read event accesses this buffer to build the response sent to the client._  

_This behavior allows replacing the initial characteristic value with dynamically sent data._  

---

#### 5) Code  

```c
#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "esp_bt.h"
#include "esp_gap_ble_api.h"
#include "esp_gatts_api.h"
#include "esp_bt_main.h"
#include "esp_gatt_common_api.h"

static const char *TAG = "BLE_CHAT";

#define DEVICE_NAME              "CHEM_ASUS"
#define GATTS_SERVICE_UUID_TEST  0x00FF
#define GATTS_CHAR_UUID_TEST     0xFF01
#define GATTS_NUM_HANDLE_TEST    4
#define ESP_APP_ID               0x55

static char stored_value[300] = "Hello \n";

static uint16_t service_handle;
static uint16_t char_handle;
static esp_gatt_if_t gatts_if_for_profile = 0;

static esp_gatt_char_prop_t char_property =
    ESP_GATT_CHAR_PROP_BIT_READ | ESP_GATT_CHAR_PROP_BIT_WRITE;

static esp_ble_adv_data_t adv_data = {
    .set_scan_rsp = false,
    .include_name = true,
    .include_txpower = false,
    .min_interval = 0x20,
    .max_interval = 0x40,
    .appearance = 0x00,
    .manufacturer_len = 0,
    .p_manufacturer_data = NULL,
    .service_data_len = 0,
    .p_service_data = NULL,
    .service_uuid_len = 0,
    .p_service_uuid = NULL,
    .flag = ESP_BLE_ADV_FLAG_GEN_DISC | ESP_BLE_ADV_FLAG_BREDR_NOT_SPT,
};

static esp_ble_adv_params_t adv_params = {
    .adv_int_min = 0x20,
    .adv_int_max = 0x40,
    .adv_type = ADV_TYPE_IND,
    .own_addr_type = BLE_ADDR_TYPE_PUBLIC,
    .channel_map = ADV_CHNL_ALL,
    .adv_filter_policy = ADV_FILTER_ALLOW_SCAN_ANY_CON_ANY,
};

// GAP
static void gap_event_handler(esp_gap_ble_cb_event_t event, esp_ble_gap_cb_param_t *param)
{
    switch (event) {
    case ESP_GAP_BLE_ADV_DATA_SET_COMPLETE_EVT:
        esp_ble_gap_start_advertising(&adv_params);
        break;

    case ESP_GAP_BLE_ADV_START_COMPLETE_EVT:
        if (param->adv_start_cmpl.status == ESP_BT_STATUS_SUCCESS) {
            ESP_LOGI(TAG, "Advertising started successfully");
        } else {
            ESP_LOGE(TAG, "Advertising start failed");
        }
        break;

    default:
        break;
    }
}

// GATT
static void gatts_profile_event_handler(esp_gatts_cb_event_t event,
                                        esp_gatt_if_t gatts_if,
                                        esp_ble_gatts_cb_param_t *param)
{
    switch (event) {

    case ESP_GATTS_REG_EVT: {
        ESP_LOGI(TAG, "GATT server registered");
        gatts_if_for_profile = gatts_if;

        esp_ble_gap_set_device_name(DEVICE_NAME);
        esp_ble_gap_config_adv_data(&adv_data);

        esp_gatt_srvc_id_t service_id = {
            .is_primary = true,
            .id.inst_id = 0x00,
            .id.uuid.len = ESP_UUID_LEN_16,
            .id.uuid.uuid.uuid16 = GATTS_SERVICE_UUID_TEST,
        };

        esp_ble_gatts_create_service(gatts_if, &service_id, GATTS_NUM_HANDLE_TEST);
        break;
    }

    case ESP_GATTS_CREATE_EVT: {
        ESP_LOGI(TAG, "Service created");
        service_handle = param->create.service_handle;
        esp_ble_gatts_start_service(service_handle);

        esp_bt_uuid_t char_uuid = {
            .len = ESP_UUID_LEN_16,
            .uuid.uuid16 = GATTS_CHAR_UUID_TEST,
        };

        esp_attr_value_t char_val = {
            .attr_max_len = sizeof(stored_value),
            .attr_len = strlen(stored_value),
            .attr_value = (uint8_t *)stored_value,
        };

        esp_attr_control_t char_control = {
            .auto_rsp = ESP_GATT_AUTO_RSP,
        };

        esp_ble_gatts_add_char(service_handle,
                               &char_uuid,
                               ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE,
                               char_property,
                               &char_val,
                               &char_control);
        break;
    }

    case ESP_GATTS_ADD_CHAR_EVT:
        ESP_LOGI(TAG, "Characteristic added");
        char_handle = param->add_char.attr_handle;
        break;

    case ESP_GATTS_WRITE_EVT:
        if (!param->write.is_prep) {

            int current_len = strlen(stored_value);
            int new_len = param->write.len;

            if (current_len + new_len + 3 < sizeof(stored_value)) {

                strcat(stored_value, "\n- ");
                strncat(stored_value, (char *)param->write.value, new_len);

                ESP_LOGI(TAG, "Historial actualizado:\n%s", stored_value);
                esp_ble_gatts_set_attr_value(char_handle,
                                             strlen(stored_value),
                                             (uint8_t *)stored_value);
            }
        }
        break;

    case ESP_GATTS_CONNECT_EVT:
        ESP_LOGI(TAG, "Central connected");
        break;

    case ESP_GATTS_DISCONNECT_EVT:
        ESP_LOGI(TAG, "Central disconnected, restarting advertising");
        esp_ble_gap_start_advertising(&adv_params);
        break;

    default:
        break;
    }
}

static void gatts_event_handler(esp_gatts_cb_event_t event,
                               esp_gatt_if_t gatts_if,
                               esp_ble_gatts_cb_param_t *param)
{
    if (event == ESP_GATTS_REG_EVT) {
        if (param->reg.status == ESP_GATT_OK) {
            gatts_if_for_profile = gatts_if;
        } else {
            ESP_LOGE(TAG, "GATT app register failed: %d", param->reg.status);
            return;
        }
    }

    gatts_profile_event_handler(event, gatts_if, param);
}
void app_main(void)
{
    esp_err_t ret;

    ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    ESP_ERROR_CHECK(esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT));

    esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_bt_controller_init(&bt_cfg));
    ESP_ERROR_CHECK(esp_bt_controller_enable(ESP_BT_MODE_BLE));

    ESP_ERROR_CHECK(esp_bluedroid_init());
    ESP_ERROR_CHECK(esp_bluedroid_enable());

    ESP_ERROR_CHECK(esp_ble_gap_register_callback(gap_event_handler));
    ESP_ERROR_CHECK(esp_ble_gatts_register_callback(gatts_event_handler));
    ESP_ERROR_CHECK(esp_ble_gatts_app_register(ESP_APP_ID));

    ESP_ERROR_CHECK(esp_ble_gatt_set_local_mtu(500));

    ESP_LOGI(TAG, "BLE Chat initialization complete");
}
```

---

#### 6) Files & Media

<div style="text-align:center; margin-top:20px;">
  <iframe 
    width="315" 
    height="560" 
    src="https://www.youtube.com/embed/xIgWSNoviEY" 
    title="Demostración del sistema"
    frameborder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen>
  </iframe>
</div>

---

### Exercise 1.2 — LED Control via BLE

---

#### 1) Activity Objectives    
_For this exercise, a system will be implemented that allows controlling an LED through commands sent via BLE._  

1) Configure a GPIO as output  
2) Receive commands from a BLE client  
3) Interpret the commands "ON" and "OFF"  
4) Control the LED state  

---

#### 2) Materials & Setup

**BOM (Bill of Materials)**

|#|Item|Qty|Notes|
|---------|--------|------|--------|
|1|ESP32-C6|1|Microcontroller with BLE|
|2|USB Cable|1|Power supply and programming|
|3|Smartphone|1|BLE client device|
|4|Led|1|Controlled hardware|

---

**_Tools / Software_**   

* _OS/Environment: Windows_  
* _Editors: VS Code / ESP-IDF_  
* _BLE Scanner: nRF Connect or similar_  

**_Wiring / Safety_**  

* _Board power supply: USB connection_  
* _Verify the connection before loading the system_  

---

#### 3) Procedure   

* _**Step 1:** Include the GPIO driver and define the LED pin._  
* _**Step 2:** Configure the pin as output and initialize it as off._  
* _**Step 3:** Read the data received in the write event._  
* _**Step 4:** Compare the commands "ON" and "OFF" and update the LED._  
* _**Step 5:** Compile, flash, and test with a BLE client._  

---

#### 4) Analysis

_The system allows controlling an LED through commands sent from a BLE client._  
_The data received in the write event is compared to determine the action to perform._  
_If the command is "ON", the LED turns on; if it is "OFF", the LED turns off._  
_Any other command is ignored and logged as unrecognized._  
_This behavior allows basic control of an actuator through BLE._  

---

#### 5) Code  

```c
#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "esp_bt.h"
#include "esp_gap_ble_api.h"
#include "esp_gatts_api.h"
#include "esp_bt_main.h"
#include "esp_gatt_common_api.h"
#include "driver/gpio.h"

static const char *TAG = "BLE_DEMO";

#define DEVICE_NAME              "ASUS"
#define GATTS_SERVICE_UUID_TEST  0x00FF
#define GATTS_CHAR_UUID_TEST     0xFF01
#define GATTS_NUM_HANDLE_TEST    4
#define PROFILE_NUM              1
#define PROFILE_APP_IDX          0
#define ESP_APP_ID               0x55

#define MAX_CHAR_LEN 128
#define LED_GPIO 2   //  usamos GPIO 2

/* ---------- BUFFER GLOBAL ---------- */
static char stored_value[MAX_CHAR_LEN] = "Hello from ESP32-C6";
static uint16_t stored_len = sizeof("Hello from ESP32-C6") - 1;

static uint16_t service_handle;
static esp_gatt_char_prop_t char_property = ESP_GATT_CHAR_PROP_BIT_READ
                                          | ESP_GATT_CHAR_PROP_BIT_WRITE;
static uint16_t char_handle;
static esp_gatt_if_t gatts_if_for_profile = 0;

/* ---------- BLE Advertising ---------- */
static esp_ble_adv_data_t adv_data = {
    .set_scan_rsp        = false,
    .include_name        = true,
    .include_txpower     = false,
    .min_interval        = 0x20,
    .max_interval        = 0x40,
    .appearance          = 0x00,
    .manufacturer_len    = 0,
    .p_manufacturer_data = NULL,
    .service_data_len    = 0,
    .p_service_data      = NULL,
    .service_uuid_len    = 0,
    .p_service_uuid      = NULL,
    .flag = ESP_BLE_ADV_FLAG_GEN_DISC | ESP_BLE_ADV_FLAG_BREDR_NOT_SPT,
};

static esp_ble_adv_params_t adv_params = {
    .adv_int_min        = 0x20,
    .adv_int_max        = 0x40,
    .adv_type           = ADV_TYPE_IND,
    .own_addr_type      = BLE_ADDR_TYPE_PUBLIC,
    .channel_map        = ADV_CHNL_ALL,
    .adv_filter_policy  = ADV_FILTER_ALLOW_SCAN_ANY_CON_ANY,
};

/* ====================== GAP ====================== */
static void gap_event_handler(esp_gap_ble_cb_event_t event,
                              esp_ble_gap_cb_param_t *param)
{
    switch (event) {
    case ESP_GAP_BLE_ADV_DATA_SET_COMPLETE_EVT:
        esp_ble_gap_start_advertising(&adv_params);
        break;

    case ESP_GAP_BLE_ADV_START_COMPLETE_EVT:
        if (param->adv_start_cmpl.status == ESP_BT_STATUS_SUCCESS) {
            ESP_LOGI(TAG, "Advertising started successfully");
        } else {
            ESP_LOGE(TAG, "Advertising start failed");
        }
        break;

    default:
        break;
    }
}

/* ====================== GATTS ====================== */
static void gatts_profile_event_handler(esp_gatts_cb_event_t event,
                                        esp_gatt_if_t gatts_if,
                                        esp_ble_gatts_cb_param_t *param)
{
    switch (event) {

    case ESP_GATTS_REG_EVT:
        ESP_LOGI(TAG, "GATT server registered");

        esp_ble_gap_set_device_name(DEVICE_NAME);
        esp_ble_gap_config_adv_data(&adv_data);

        esp_gatt_srvc_id_t service_id = {
            .is_primary   = true,
            .id.inst_id   = 0x00,
            .id.uuid.len  = ESP_UUID_LEN_16,
            .id.uuid.uuid.uuid16 = GATTS_SERVICE_UUID_TEST,
        };

        esp_ble_gatts_create_service(gatts_if, &service_id, GATTS_NUM_HANDLE_TEST);
        break;

    case ESP_GATTS_CREATE_EVT:
        ESP_LOGI(TAG, "Service created");
        service_handle = param->create.service_handle;
        esp_ble_gatts_start_service(service_handle);

        esp_bt_uuid_t char_uuid = {
            .len = ESP_UUID_LEN_16,
            .uuid.uuid16 = GATTS_CHAR_UUID_TEST,
        };

        esp_attr_value_t char_val = {
            .attr_max_len = MAX_CHAR_LEN,
            .attr_len = stored_len,
            .attr_value = (uint8_t *)stored_value,
        };

        esp_attr_control_t char_control = {
            .auto_rsp = ESP_GATT_RSP_BY_APP,
        };

        esp_ble_gatts_add_char(service_handle,
                               &char_uuid,
                               ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE,
                               char_property,
                               &char_val,
                               &char_control);
        break;

    case ESP_GATTS_ADD_CHAR_EVT:
        ESP_LOGI(TAG, "Characteristic added");
        char_handle = param->add_char.attr_handle;
        break;

    /* ---------- WRITE ---------- */
    case ESP_GATTS_WRITE_EVT:
        ESP_LOGI(TAG, "Write event received");

        if (param->write.len > 0) {
            uint16_t len = param->write.len;
            if (len > MAX_CHAR_LEN - 1) len = MAX_CHAR_LEN - 1;

            memcpy(stored_value, param->write.value, len);
            stored_value[len] = '\0';
            stored_len = len;

            ESP_LOGI(TAG, "Data received: %s", stored_value);

            /* CONTROL LED */
            if (strcmp(stored_value, "ON") == 0) {
                gpio_set_level(LED_GPIO, 1);
                ESP_LOGI(TAG, "LED ON");
            }
            else if (strcmp(stored_value, "OFF") == 0) {
                gpio_set_level(LED_GPIO, 0);
                ESP_LOGI(TAG, "LED OFF");
            }
            else {
                ESP_LOGW(TAG, "Unknown command: %s", stored_value);
            }
        }

        if (param->write.need_rsp) {
            esp_ble_gatts_send_response(gatts_if,
                                        param->write.conn_id,
                                        param->write.trans_id,
                                        ESP_GATT_OK,
                                        NULL);
        }
        break;

    /* ---------- READ ---------- */
    case ESP_GATTS_READ_EVT: {
        ESP_LOGI(TAG, "Read event");

        esp_gatt_rsp_t rsp;
        memset(&rsp, 0, sizeof(rsp));

        rsp.attr_value.handle = param->read.handle;
        rsp.attr_value.len = stored_len;
        memcpy(rsp.attr_value.value, stored_value, stored_len);

        esp_ble_gatts_send_response(gatts_if,
                                    param->read.conn_id,
                                    param->read.trans_id,
                                    ESP_GATT_OK,
                                    &rsp);
        break;
    }

    case ESP_GATTS_CONNECT_EVT:
        ESP_LOGI(TAG, "Central connected");
        break;

    case ESP_GATTS_DISCONNECT_EVT:
        ESP_LOGI(TAG, "Central disconnected");
        esp_ble_gap_start_advertising(&adv_params);
        break;

    default:
        break;
    }
}

/* ====================== DISPATCHER ====================== */
static void gatts_event_handler(esp_gatts_cb_event_t event,
                                esp_gatt_if_t gatts_if,
                                esp_ble_gatts_cb_param_t *param)
{
    if (event == ESP_GATTS_REG_EVT && param->reg.status != ESP_GATT_OK) {
        ESP_LOGE(TAG, "GATT register failed");
        return;
    }

    gatts_profile_event_handler(event, gatts_if, param);
}

/* ====================== MAIN ====================== */
void app_main(void)
{
    esp_err_t ret;

    /* INIT LED */
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(LED_GPIO, 0);

    /* ---------- BLE INIT ---------- */
    ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES ||
        ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    ESP_ERROR_CHECK(esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT));

    esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_bt_controller_init(&bt_cfg));
    ESP_ERROR_CHECK(esp_bt_controller_enable(ESP_BT_MODE_BLE));

    ESP_ERROR_CHECK(esp_bluedroid_init());
    ESP_ERROR_CHECK(esp_bluedroid_enable());

    ESP_ERROR_CHECK(esp_ble_gap_register_callback(gap_event_handler));
    ESP_ERROR_CHECK(esp_ble_gatts_register_callback(gatts_event_handler));
    ESP_ERROR_CHECK(esp_ble_gatts_app_register(ESP_APP_ID));

    ESP_ERROR_CHECK(esp_ble_gatt_set_local_mtu(500));

    ESP_LOGI(TAG, "BLE initialization complete");
}
```

---

#### 6) Files & Media

<div style="text-align:center; margin-top:20px;">
  <iframe 
    width="315" 
    height="560" 
    src="https://www.youtube.com/embed/e6mVLkYyew0" 
    title="Demostración del sistema"
    frameborder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen>
  </iframe>
</div>

---

### Exercise 1.3 — Write-Length Guard

---

#### 1) Activity Objectives    
_For this exercise, a system will be implemented that allows validating the length of the received data and rejecting those that exceed the allowed limit._  

1) Validate the length of the received data  
2) Reject data greater than 64 bytes  
3) Send an appropriate error code  
4) Prevent processing of invalid data  

---

#### 2) Materials & Setup

**BOM (Bill of Materials)**

|#|Item|Qty|Notes|
|---------|--------|------|--------|
|1|ESP32-C6|1|Microcontroller with BLE|
|2|USB Cable|1|Power supply and programming|
|3|Smartphone|1|BLE client device|

---

**_Tools / Software_**   

* _OS/Environment: Windows_  
* _Editors: VS Code / ESP-IDF_  
* _BLE Scanner: nRF Connect or similar_  

**_Wiring / Safety_**  

* _Board power supply: USB connection_  
* _Verify the connection before loading the system_  

---

#### 3) Procedure   

* _**Step 1:** Verify the length of the data at the beginning of the write event._  
* _**Step 2:** If it exceeds 64 bytes, send an error and stop execution._  
* _**Step 3:** Ensure that validation occurs before any processing._  
* _**Step 4:** Compile, flash, and test with different data lengths._  

---

#### 4) Analysis

_The system validates the length of the received data before processing it._  
_If the data exceeds the allowed limit, an error code is sent and it is not processed._  
_This prevents handling invalid data within the system._  
_Validation is performed at the beginning of the write event to ensure that only valid data is processed._  

---

#### 5) Code  

```c
#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "esp_bt.h"
#include "esp_gap_ble_api.h"
#include "esp_gatts_api.h"
#include "esp_bt_main.h"
#include "esp_gatt_common_api.h"
#include "driver/gpio.h"

static const char *TAG = "BLE_DEMO";

#define DEVICE_NAME              "ASUS"
#define GATTS_SERVICE_UUID_TEST  0x00FF
#define GATTS_CHAR_UUID_TEST     0xFF01
#define GATTS_NUM_HANDLE_TEST    4
#define ESP_APP_ID               0x55

#define MAX_CHAR_LEN 128
#define LED_GPIO 2

/* ---------- BUFFER GLOBAL ---------- */
static char stored_value[MAX_CHAR_LEN] = "Hello from ESP32-C6";
static uint16_t stored_len = sizeof("Hello from ESP32-C6") - 1;

static uint16_t service_handle;
static esp_gatt_char_prop_t char_property =
    ESP_GATT_CHAR_PROP_BIT_READ | ESP_GATT_CHAR_PROP_BIT_WRITE;

static uint16_t char_handle;

/* ---------- BLE Advertising ---------- */
static esp_ble_adv_data_t adv_data = {
    .set_scan_rsp        = false,
    .include_name        = true,
    .include_txpower     = false,
    .min_interval        = 0x20,
    .max_interval        = 0x40,
    .appearance          = 0x00,
    .manufacturer_len    = 0,
    .p_manufacturer_data = NULL,
    .service_data_len    = 0,
    .p_service_data      = NULL,
    .service_uuid_len    = 0,
    .p_service_uuid      = NULL,
    .flag = ESP_BLE_ADV_FLAG_GEN_DISC | ESP_BLE_ADV_FLAG_BREDR_NOT_SPT,
};

static esp_ble_adv_params_t adv_params = {
    .adv_int_min        = 0x20,
    .adv_int_max        = 0x40,
    .adv_type           = ADV_TYPE_IND,
    .own_addr_type      = BLE_ADDR_TYPE_PUBLIC,
    .channel_map        = ADV_CHNL_ALL,
    .adv_filter_policy  = ADV_FILTER_ALLOW_SCAN_ANY_CON_ANY,
};

/* ====================== GAP ====================== */
static void gap_event_handler(esp_gap_ble_cb_event_t event,
                              esp_ble_gap_cb_param_t *param)
{
    switch (event) {
    case ESP_GAP_BLE_ADV_DATA_SET_COMPLETE_EVT:
        esp_ble_gap_start_advertising(&adv_params);
        break;

    case ESP_GAP_BLE_ADV_START_COMPLETE_EVT:
        if (param->adv_start_cmpl.status == ESP_BT_STATUS_SUCCESS) {
            ESP_LOGI(TAG, "Advertising started successfully");
        } else {
            ESP_LOGE(TAG, "Advertising start failed");
        }
        break;

    default:
        break;
    }
}

/* ====================== GATTS ====================== */
static void gatts_profile_event_handler(esp_gatts_cb_event_t event,
                                        esp_gatt_if_t gatts_if,
                                        esp_ble_gatts_cb_param_t *param)
{
    switch (event) {

    case ESP_GATTS_REG_EVT:
        ESP_LOGI(TAG, "GATT server registered");

        esp_ble_gap_set_device_name(DEVICE_NAME);
        esp_ble_gap_config_adv_data(&adv_data);

        esp_gatt_srvc_id_t service_id = {
            .is_primary = true,
            .id.inst_id = 0x00,
            .id.uuid.len = ESP_UUID_LEN_16,
            .id.uuid.uuid.uuid16 = GATTS_SERVICE_UUID_TEST,
        };

        esp_ble_gatts_create_service(gatts_if, &service_id, GATTS_NUM_HANDLE_TEST);
        break;

    case ESP_GATTS_CREATE_EVT:
        ESP_LOGI(TAG, "Service created");
        service_handle = param->create.service_handle;
        esp_ble_gatts_start_service(service_handle);

        esp_bt_uuid_t char_uuid = {
            .len = ESP_UUID_LEN_16,
            .uuid.uuid16 = GATTS_CHAR_UUID_TEST,
        };

        esp_attr_value_t char_val = {
            .attr_max_len = MAX_CHAR_LEN,
            .attr_len = stored_len,
            .attr_value = (uint8_t *)stored_value,
        };

        esp_attr_control_t char_control = {
            .auto_rsp = ESP_GATT_RSP_BY_APP,
        };

        esp_ble_gatts_add_char(service_handle,
                               &char_uuid,
                               ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE,
                               char_property,
                               &char_val,
                               &char_control);
        break;

    case ESP_GATTS_ADD_CHAR_EVT:
        ESP_LOGI(TAG, "Characteristic added");
        char_handle = param->add_char.attr_handle;
        break;

    /* ---------- WRITE ---------- */
    case ESP_GATTS_WRITE_EVT:
        ESP_LOGI(TAG, "Write event received");

        /* PROTECCIÓN >64 bytes */
        if (param->write.len > 64) {
            ESP_LOGW(TAG, "Write rejected, length too long: %d", param->write.len);

            if (param->write.need_rsp) {
                esp_ble_gatts_send_response(gatts_if,
                                            param->write.conn_id,
                                            param->write.trans_id,
                                            ESP_GATT_INVALID_ATTR_LEN,
                                            NULL);
            }
            break;
        }

        if (param->write.len > 0) {
            uint16_t len = param->write.len;
            if (len > MAX_CHAR_LEN - 1) len = MAX_CHAR_LEN - 1;

            memcpy(stored_value, param->write.value, len);
            stored_value[len] = '\0';
            stored_len = len;

            ESP_LOGI(TAG, "Data received: %s", stored_value);

            /* CONTROL LED */
            if (strcmp(stored_value, "ON") == 0) {
                gpio_set_level(LED_GPIO, 1);
                ESP_LOGI(TAG, "LED ON");
            }
            else if (strcmp(stored_value, "OFF") == 0) {
                gpio_set_level(LED_GPIO, 0);
                ESP_LOGI(TAG, "LED OFF");
            }
            else {
                ESP_LOGW(TAG, "Unknown command: %s", stored_value);
            }
        }

        if (param->write.need_rsp) {
            esp_ble_gatts_send_response(gatts_if,
                                        param->write.conn_id,
                                        param->write.trans_id,
                                        ESP_GATT_OK,
                                        NULL);
        }
        break;

    /* ---------- READ ---------- */
    case ESP_GATTS_READ_EVT: {
        ESP_LOGI(TAG, "Read event");

        esp_gatt_rsp_t rsp;
        memset(&rsp, 0, sizeof(rsp));

        rsp.attr_value.handle = param->read.handle;
        rsp.attr_value.len = stored_len;
        memcpy(rsp.attr_value.value, stored_value, stored_len);

        esp_ble_gatts_send_response(gatts_if,
                                    param->read.conn_id,
                                    param->read.trans_id,
                                    ESP_GATT_OK,
                                    &rsp);
        break;
    }

    case ESP_GATTS_CONNECT_EVT:
        ESP_LOGI(TAG, "Central connected");
        break;

    case ESP_GATTS_DISCONNECT_EVT:
        ESP_LOGI(TAG, "Central disconnected");
        esp_ble_gap_start_advertising(&adv_params);
        break;

    default:
        break;
    }
}

/* ====================== DISPATCHER ====================== */
static void gatts_event_handler(esp_gatts_cb_event_t event,
                                esp_gatt_if_t gatts_if,
                                esp_ble_gatts_cb_param_t *param)
{
    if (event == ESP_GATTS_REG_EVT && param->reg.status != ESP_GATT_OK) {
        ESP_LOGE(TAG, "GATT register failed");
        return;
    }

    gatts_profile_event_handler(event, gatts_if, param);
}

/* ====================== MAIN ====================== */
void app_main(void)
{
    esp_err_t ret;

    /* INIT LED */
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(LED_GPIO, 0);

    /* ---------- BLE INIT ---------- */
    ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES ||
        ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    ESP_ERROR_CHECK(esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT));

    esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_bt_controller_init(&bt_cfg));
    ESP_ERROR_CHECK(esp_bt_controller_enable(ESP_BT_MODE_BLE));

    ESP_ERROR_CHECK(esp_bluedroid_init());
    ESP_ERROR_CHECK(esp_bluedroid_enable());

    ESP_ERROR_CHECK(esp_ble_gap_register_callback(gap_event_handler));
    ESP_ERROR_CHECK(esp_ble_gatts_register_callback(gatts_event_handler));
    ESP_ERROR_CHECK(esp_ble_gatts_app_register(ESP_APP_ID));

    ESP_ERROR_CHECK(esp_ble_gatt_set_local_mtu(500));

    ESP_LOGI(TAG, "BLE initialization complete");
}
```

---

#### 6) Files & Media

<div style="text-align:center; margin-top:20px;">
  <iframe 
    width="315" 
    height="560" 
    src="https://www.youtube.com/embed/f8gzU8DepJs" 
    title="Demostración 3"
    frameborder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen>
  </iframe>
</div>

---

## Lab 2

---

### Exercise 2.1 — Add a Third Service

#### 1) Activity Objectives    
_For this exercise, a system will be implemented that allows adding a new BLE service with a characteristic to store a PIN code._  

1) Define a new BLE service  
2) Add a read/write characteristic  
3) Store a 4-character PIN code  
4) Allow reading and writing of the PIN  

---

#### 2) Materials & Setup

**BOM (Bill of Materials)**

|#|Item|Qty|Notes|
|---------|--------|------|--------|
|1|ESP32-C6|1|Microcontroller with BLE|
|2|USB Cable|1|Power supply and programming|
|3|Smartphone|1|BLE client device|

---

**_Tools / Software_**   

* _OS/Environment: Windows_  
* _Editors: VS Code / ESP-IDF_  
* _BLE Scanner: nRF Connect or similar_  

**_Wiring / Safety_**  

* _Board power supply: USB connection_  
* _Verify the connection before loading the system_  

---

#### 3) Procedure   

* _**Step 1:** Define the UUIDs for the new service and characteristic._  
* _**Step 2:** Declare variables for the service, characteristic, and PIN._  
* _**Step 3:** Create and start the new service._  
* _**Step 4:** Add the PIN characteristic with read and write permissions._  
* _**Step 5:** Implement write and read handling for the PIN._  
* _**Step 6:** Compile, flash, and test with a BLE client._  

---

#### 4) Analysis

_The system allows adding a new BLE service that stores a PIN code._  
_The characteristic enables both reading and writing, allowing dynamic modification of the stored value._  
_The use of a default value ensures that the system has a valid initial state._  
_This approach demonstrates how multiple services can coexist within the same BLE device._  

---

#### 5) Code  
```c
#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "esp_bt.h"
#include "esp_gap_ble_api.h"
#include "esp_gatts_api.h"
#include "esp_bt_main.h"
#include "esp_gatt_common_api.h"

static const char *TAG = "BLE_DEMO";

#define DEVICE_NAME              "ESPROY32"
#define PROFILE_NUM              1
#define PROFILE_APP_IDX          0
#define ESP_APP_ID               0x55

/* ── Service A: Device Info ── */
#define SVC_A_UUID               0xAA00
#define CHAR_A1_UUID             0xAA01
#define CHAR_A2_UUID             0xAA02
#define SVC_A_NUM_HANDLES        6

/* ── Service B: Sensor ── */
#define SVC_B_UUID               0xBB00
#define CHAR_B1_UUID             0xBB01
#define CHAR_B2_UUID             0xBB02
#define SVC_B_NUM_HANDLES        6

/* ── Service C: PIN ── */
#define SVC_C_UUID               0xCC00
#define CHAR_C1_UUID             0xCC01
#define SVC_C_NUM_HANDLES        4

/* ── Default values ── */
static const char firmware_version[] = "1.0.0";
static char device_name[32] = "ESP32C6_BLE_DEMO";
static char temperature[8]  = "25.0";
static char pin_code[5]     = "0000";   // 4 chars + '\0'

/* ── Handles ── */
static uint16_t svc_a_handle;
static uint16_t char_a1_handle;
static uint16_t char_a2_handle;

static uint16_t svc_b_handle;
static uint16_t char_b1_handle;
static uint16_t char_b2_handle;

static uint16_t svc_c_handle;
static uint16_t char_c1_handle;

static int setup_step = 0;
static esp_gatt_if_t gatts_if_for_profile = 0;

/* ── Advertising ── */
static esp_ble_adv_data_t adv_data = {
    .set_scan_rsp        = false,
    .include_name        = true,
    .include_txpower     = false,
    .min_interval        = 0x20,
    .max_interval        = 0x40,
    .appearance          = 0x00,
    .manufacturer_len    = 0,
    .p_manufacturer_data = NULL,
    .service_data_len    = 0,
    .p_service_data      = NULL,
    .service_uuid_len    = 0,
    .p_service_uuid      = NULL,
    .flag = ESP_BLE_ADV_FLAG_GEN_DISC | ESP_BLE_ADV_FLAG_BREDR_NOT_SPT,
};

static esp_ble_adv_params_t adv_params = {
    .adv_int_min        = 0x20,
    .adv_int_max        = 0x40,
    .adv_type           = ADV_TYPE_IND,
    .own_addr_type      = BLE_ADDR_TYPE_PUBLIC,
    .channel_map        = ADV_CHNL_ALL,
    .adv_filter_policy  = ADV_FILTER_ALLOW_SCAN_ANY_CON_ANY,
};

/* ====================== GAP callback ====================== */
static void gap_event_handler(esp_gap_ble_cb_event_t event,
                              esp_ble_gap_cb_param_t *param)
{
    switch (event) {
    case ESP_GAP_BLE_ADV_DATA_SET_COMPLETE_EVT:
        esp_ble_gap_start_advertising(&adv_params);
        break;

    case ESP_GAP_BLE_ADV_START_COMPLETE_EVT:
        if (param->adv_start_cmpl.status == ESP_BT_STATUS_SUCCESS) {
            ESP_LOGI(TAG, "Advertising started successfully");
        } else {
            ESP_LOGE(TAG, "Advertising start failed");
        }
        break;

    default:
        break;
    }
}

/* ====================== Helpers ====================== */
static void create_service(esp_gatt_if_t gif, uint16_t uuid, uint16_t num_handles)
{
    esp_gatt_srvc_id_t id = {
        .is_primary  = true,
        .id.inst_id  = 0x00,
        .id.uuid.len = ESP_UUID_LEN_16,
        .id.uuid.uuid.uuid16 = uuid,
    };
    esp_ble_gatts_create_service(gif, &id, num_handles);
}

static void add_char(uint16_t svc_handle, uint16_t uuid,
                     esp_gatt_perm_t perm, esp_gatt_char_prop_t prop,
                     uint8_t *value, uint16_t val_len, uint16_t max_len,
                     bool auto_rsp)
{
    esp_bt_uuid_t cuuid = {
        .len         = ESP_UUID_LEN_16,
        .uuid.uuid16 = uuid,
    };

    esp_attr_value_t val = {
        .attr_max_len = max_len,
        .attr_len     = val_len,
        .attr_value   = value,
    };

    esp_attr_control_t ctrl = {
        .auto_rsp = auto_rsp ? ESP_GATT_AUTO_RSP : ESP_GATT_RSP_BY_APP,
    };

    esp_ble_gatts_add_char(svc_handle, &cuuid, perm, prop, &val, &ctrl);
}

/* ====================== GATTS profile callback ====================== */
static void gatts_profile_event_handler(esp_gatts_cb_event_t event,
                                        esp_gatt_if_t gatts_if,
                                        esp_ble_gatts_cb_param_t *param)
{
    switch (event) {

    /* ---------- Registration ---------- */
    case ESP_GATTS_REG_EVT:
        ESP_LOGI(TAG, "GATT server registered");
        gatts_if_for_profile = gatts_if;

        esp_ble_gap_set_device_name(DEVICE_NAME);
        esp_ble_gap_config_adv_data(&adv_data);

        create_service(gatts_if, SVC_A_UUID, SVC_A_NUM_HANDLES);
        break;

    /* ---------- Service created ---------- */
    case ESP_GATTS_CREATE_EVT: {
        uint16_t handle = param->create.service_handle;
        uint16_t uuid   = param->create.service_id.id.uuid.uuid.uuid16;

        esp_ble_gatts_start_service(handle);

        if (uuid == SVC_A_UUID) {
            ESP_LOGI(TAG, "Service A (Device Info) created");
            svc_a_handle = handle;

            add_char(svc_a_handle, CHAR_A1_UUID,
                     ESP_GATT_PERM_READ,
                     ESP_GATT_CHAR_PROP_BIT_READ,
                     (uint8_t *)firmware_version, strlen(firmware_version),
                     sizeof(firmware_version), true);

        } else if (uuid == SVC_B_UUID) {
            ESP_LOGI(TAG, "Service B (Sensor) created");
            svc_b_handle = handle;

            add_char(svc_b_handle, CHAR_B1_UUID,
                     ESP_GATT_PERM_READ,
                     ESP_GATT_CHAR_PROP_BIT_READ,
                     (uint8_t *)temperature, strlen(temperature),
                     sizeof(temperature), true);

        } else if (uuid == SVC_C_UUID) {
            ESP_LOGI(TAG, "Service C (PIN) created");
            svc_c_handle = handle;

            add_char(svc_c_handle, CHAR_C1_UUID,
                     ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE,
                     ESP_GATT_CHAR_PROP_BIT_READ | ESP_GATT_CHAR_PROP_BIT_WRITE,
                     (uint8_t *)pin_code, strlen(pin_code),
                     sizeof(pin_code), false);
        }
        break;
    }

    /* ---------- Characteristic added — chain to next ---------- */
    case ESP_GATTS_ADD_CHAR_EVT: {
        setup_step++;
        ESP_LOGI(TAG, "Characteristic added (step %d), handle = %d",
                 setup_step, param->add_char.attr_handle);

        switch (setup_step) {
        case 1:
            char_a1_handle = param->add_char.attr_handle;
            add_char(svc_a_handle, CHAR_A2_UUID,
                     ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE,
                     ESP_GATT_CHAR_PROP_BIT_READ | ESP_GATT_CHAR_PROP_BIT_WRITE,
                     (uint8_t *)device_name, strlen(device_name),
                     sizeof(device_name), false);
            break;

        case 2:
            char_a2_handle = param->add_char.attr_handle;
            create_service(gatts_if_for_profile, SVC_B_UUID, SVC_B_NUM_HANDLES);
            break;

        case 3:
            char_b1_handle = param->add_char.attr_handle;
            add_char(svc_b_handle, CHAR_B2_UUID,
                     ESP_GATT_PERM_WRITE,
                     ESP_GATT_CHAR_PROP_BIT_WRITE,
                     NULL, 0, 64, false);
            break;

        case 4:
            char_b2_handle = param->add_char.attr_handle;
            create_service(gatts_if_for_profile, SVC_C_UUID, SVC_C_NUM_HANDLES);
            break;

        case 5:
            char_c1_handle = param->add_char.attr_handle;
            ESP_LOGI(TAG, "All services and characteristics ready");
            break;
        }
        break;
    }

    /* ---------- Write events ---------- */
    case ESP_GATTS_WRITE_EVT: {
        uint16_t handle = param->write.handle;

        char buf[param->write.len + 1];
        memcpy(buf, param->write.value, param->write.len);
        buf[param->write.len] = '\0';

        if (handle == char_a2_handle) {
            strncpy(device_name, buf, sizeof(device_name) - 1);
            device_name[sizeof(device_name) - 1] = '\0';
            ESP_LOGI(TAG, "[Device Info] Name updated to: %s", device_name);

        } else if (handle == char_b2_handle) {
            ESP_LOGI(TAG, "[Sensor] Command received: %s", buf);

        } else if (handle == char_c1_handle) {
            if (param->write.len != 4) {
                ESP_LOGW(TAG, "[PIN] Invalid length: %d (must be 4)", param->write.len);

                if (param->write.need_rsp) {
                    esp_ble_gatts_send_response(gatts_if,
                                                param->write.conn_id,
                                                param->write.trans_id,
                                                ESP_GATT_INVALID_ATTR_LEN,
                                                NULL);
                }
                break;
            }

            memcpy(pin_code, param->write.value, 4);
            pin_code[4] = '\0';

            ESP_LOGI(TAG, "[PIN] Updated to: %s", pin_code);

        } else {
            ESP_LOGW(TAG, "Write to unknown handle %d", handle);
        }

        if (param->write.need_rsp) {
            esp_ble_gatts_send_response(gatts_if,
                                        param->write.conn_id,
                                        param->write.trans_id,
                                        ESP_GATT_OK, NULL);
        }
        break;
    }

    /* ---------- Read events (manual response chars only) ---------- */
    case ESP_GATTS_READ_EVT: {
        esp_gatt_rsp_t rsp = {0};
        rsp.attr_value.handle = param->read.handle;

        if (param->read.handle == char_a2_handle) {
            rsp.attr_value.len = strlen(device_name);
            memcpy(rsp.attr_value.value, device_name, rsp.attr_value.len);

            esp_ble_gatts_send_response(gatts_if,
                                        param->read.conn_id,
                                        param->read.trans_id,
                                        ESP_GATT_OK, &rsp);

        } else if (param->read.handle == char_c1_handle) {
            rsp.attr_value.len = strlen(pin_code);
            memcpy(rsp.attr_value.value, pin_code, rsp.attr_value.len);

            esp_ble_gatts_send_response(gatts_if,
                                        param->read.conn_id,
                                        param->read.trans_id,
                                        ESP_GATT_OK, &rsp);
        }
        break;
    }

    /* ---------- Connection / Disconnection ---------- */
    case ESP_GATTS_CONNECT_EVT:
        ESP_LOGI(TAG, "Central connected");
        break;

    case ESP_GATTS_DISCONNECT_EVT:
        ESP_LOGI(TAG, "Central disconnected, restarting advertising");
        esp_ble_gap_start_advertising(&adv_params);
        break;

    default:
        break;
    }
}

/* ====================== GATTS dispatcher ====================== */
static void gatts_event_handler(esp_gatts_cb_event_t event,
                                esp_gatt_if_t gatts_if,
                                esp_ble_gatts_cb_param_t *param)
{
    if (event == ESP_GATTS_REG_EVT) {
        if (param->reg.status == ESP_GATT_OK) {
            gatts_if_for_profile = gatts_if;
        } else {
            ESP_LOGE(TAG, "GATT app register failed: %d", param->reg.status);
            return;
        }
    }

    gatts_profile_event_handler(event, gatts_if, param);
}

/* ====================== Entry point ====================== */
void app_main(void)
{
    esp_err_t ret;

    ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES ||
        ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    ESP_ERROR_CHECK(esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT));

    esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_bt_controller_init(&bt_cfg));
    ESP_ERROR_CHECK(esp_bt_controller_enable(ESP_BT_MODE_BLE));

    ESP_ERROR_CHECK(esp_bluedroid_init());
    ESP_ERROR_CHECK(esp_bluedroid_enable());

    ESP_ERROR_CHECK(esp_ble_gap_register_callback(gap_event_handler));
    ESP_ERROR_CHECK(esp_ble_gatts_register_callback(gatts_event_handler));
    ESP_ERROR_CHECK(esp_ble_gatts_app_register(ESP_APP_ID));

    ESP_ERROR_CHECK(esp_ble_gatt_set_local_mtu(500));

    ESP_LOGI(TAG, "BLE initialization complete");
}
```

---

### 6) Files & Media

---

### Exercise 2.2 — Notify on Temperature Change

#### 1) Activity Objectives    
_For this exercise, a system will be implemented that allows sending notifications to the client when the temperature value changes._  

1) Enable notifications in a BLE characteristic  
2) Configure a descriptor for notification control  
3) Detect a command from the client  
4) Send updated data automatically  

---

#### 2) Materials & Setup

**BOM (Bill of Materials)**

|#|Item|Qty|Notes|
|---------|--------|------|--------|
|1|ESP32-C6|1|Microcontroller with BLE|
|2|USB Cable|1|Power supply and programming|
|3|Smartphone|1|BLE client device|

---

**_Tools / Software_**   

* _OS/Environment: Windows_  
* _Editors: VS Code / ESP-IDF_  
* _BLE Scanner: nRF Connect or similar_  

**_Wiring / Safety_**  

* _Board power supply: USB connection_  
* _Verify the connection before loading the system_  

---

#### 3) Procedure   

* _**Step 1:** Enable the notify property in the temperature characteristic._  
* _**Step 2:** Add a descriptor to allow enabling notifications._  
* _**Step 3:** Detect configuration writes from the client._  
* _**Step 4:** When receiving "UPDATE", modify the temperature value._  
* _**Step 5:** Send a notification if enabled._  
* _**Step 6:** Compile, flash, and test with a BLE client._  

---

#### 4) Analysis

_The system allows sending automatic notifications to the client when a value changes._  
_The descriptor enables the client to activate or deactivate notifications._  
_The command received triggers an update in the temperature value._  
_This behavior allows real-time communication without requiring manual reads._  

---

#### 5) Code  
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "esp_bt.h"
#include "esp_gap_ble_api.h"
#include "esp_gatts_api.h"
#include "esp_bt_main.h"
#include "esp_gatt_common_api.h"

static const char *TAG = "BLE_DEMO";

#define DEVICE_NAME              "ESPROY32"
#define PROFILE_NUM              1
#define PROFILE_APP_IDX          0
#define ESP_APP_ID               0x55

/* ── Service A: Device Info ── */
#define SVC_A_UUID               0xAA00
#define CHAR_A1_UUID             0xAA01
#define CHAR_A2_UUID             0xAA02
#define SVC_A_NUM_HANDLES        6

/* ── Service B: Sensor ── */
#define SVC_B_UUID               0xBB00
#define CHAR_B1_UUID             0xBB01
#define CHAR_B2_UUID             0xBB02
#define SVC_B_NUM_HANDLES        8   // antes 6, ahora 8 por descriptor

/* ── Service C: PIN ── */
#define SVC_C_UUID               0xCC00
#define CHAR_C1_UUID             0xCC01
#define SVC_C_NUM_HANDLES        4

/* ── Default values ── */
static const char firmware_version[] = "1.0.0";
static char       device_name[32]    = "ESP32C6_BLE_DEMO";
static char       temperature[8]     = "25.0";
static char       pin_code[5]        = "0000";
static float      temp_value         = 25.0;

/* ── Handles ── */
static uint16_t svc_a_handle;
static uint16_t char_a1_handle;
static uint16_t char_a2_handle;

static uint16_t svc_b_handle;
static uint16_t char_b1_handle;
static uint16_t char_b2_handle;
static uint16_t temp_cccd_handle;   // NUEVO

static uint16_t svc_c_handle;
static uint16_t char_c1_handle;

static int setup_step = 0;
static esp_gatt_if_t gatts_if_for_profile = 0;
static uint16_t current_conn_id = 0;      // NUEVO
static bool temp_notify_enabled = false;  // NUEVO

/* ── Advertising ── */
static esp_ble_adv_data_t adv_data = {
    .set_scan_rsp        = false,
    .include_name        = true,
    .include_txpower     = false,
    .min_interval        = 0x20,
    .max_interval        = 0x40,
    .appearance          = 0x00,
    .manufacturer_len    = 0,
    .p_manufacturer_data = NULL,
    .service_data_len    = 0,
    .p_service_data      = NULL,
    .service_uuid_len    = 0,
    .p_service_uuid      = NULL,
    .flag = ESP_BLE_ADV_FLAG_GEN_DISC | ESP_BLE_ADV_FLAG_BREDR_NOT_SPT,
};

static esp_ble_adv_params_t adv_params = {
    .adv_int_min        = 0x20,
    .adv_int_max        = 0x40,
    .adv_type           = ADV_TYPE_IND,
    .own_addr_type      = BLE_ADDR_TYPE_PUBLIC,
    .channel_map        = ADV_CHNL_ALL,
    .adv_filter_policy  = ADV_FILTER_ALLOW_SCAN_ANY_CON_ANY,
};

/* ====================== GAP callback ====================== */
static void gap_event_handler(esp_gap_ble_cb_event_t event,
                              esp_ble_gap_cb_param_t *param)
{
    switch (event) {
    case ESP_GAP_BLE_ADV_DATA_SET_COMPLETE_EVT:
        esp_ble_gap_start_advertising(&adv_params);
        break;

    case ESP_GAP_BLE_ADV_START_COMPLETE_EVT:
        if (param->adv_start_cmpl.status == ESP_BT_STATUS_SUCCESS) {
            ESP_LOGI(TAG, "Advertising started successfully");
        } else {
            ESP_LOGE(TAG, "Advertising start failed");
        }
        break;

    default:
        break;
    }
}

/* ====================== Helpers ====================== */
static void create_service(esp_gatt_if_t gif, uint16_t uuid, uint16_t num_handles)
{
    esp_gatt_srvc_id_t id = {
        .is_primary  = true,
        .id.inst_id  = 0x00,
        .id.uuid.len = ESP_UUID_LEN_16,
        .id.uuid.uuid.uuid16 = uuid,
    };
    esp_ble_gatts_create_service(gif, &id, num_handles);
}

static void add_char(uint16_t svc_handle, uint16_t uuid,
                     esp_gatt_perm_t perm, esp_gatt_char_prop_t prop,
                     uint8_t *value, uint16_t val_len, uint16_t max_len,
                     bool auto_rsp)
{
    esp_bt_uuid_t cuuid = {
        .len         = ESP_UUID_LEN_16,
        .uuid.uuid16 = uuid,
    };

    esp_attr_value_t val = {
        .attr_max_len = max_len,
        .attr_len     = val_len,
        .attr_value   = value,
    };

    esp_attr_control_t ctrl = {
        .auto_rsp = auto_rsp ? ESP_GATT_AUTO_RSP : ESP_GATT_RSP_BY_APP,
    };

    esp_ble_gatts_add_char(svc_handle, &cuuid, perm, prop, &val, &ctrl);
}

/* ====================== GATTS profile callback ====================== */
static void gatts_profile_event_handler(esp_gatts_cb_event_t event,
                                        esp_gatt_if_t gatts_if,
                                        esp_ble_gatts_cb_param_t *param)
{
    switch (event) {

    /* ---------- Registration ---------- */
    case ESP_GATTS_REG_EVT:
        ESP_LOGI(TAG, "GATT server registered");
        gatts_if_for_profile = gatts_if;

        esp_ble_gap_set_device_name(DEVICE_NAME);
        esp_ble_gap_config_adv_data(&adv_data);

        create_service(gatts_if, SVC_A_UUID, SVC_A_NUM_HANDLES);
        break;

    /* ---------- Service created ---------- */
    case ESP_GATTS_CREATE_EVT: {
        uint16_t handle = param->create.service_handle;
        esp_ble_gatts_start_service(handle);

        if (param->create.service_id.id.uuid.uuid.uuid16 == SVC_A_UUID) {
            ESP_LOGI(TAG, "Service A (Device Info) created");
            svc_a_handle = handle;

            add_char(svc_a_handle, CHAR_A1_UUID,
                     ESP_GATT_PERM_READ,
                     ESP_GATT_CHAR_PROP_BIT_READ,
                     (uint8_t *)firmware_version, strlen(firmware_version),
                     sizeof(firmware_version), true);

        } else if (param->create.service_id.id.uuid.uuid.uuid16 == SVC_B_UUID) {
            ESP_LOGI(TAG, "Service B (Sensor) created");
            svc_b_handle = handle;

            add_char(svc_b_handle, CHAR_B1_UUID,
                     ESP_GATT_PERM_READ,
                     ESP_GATT_CHAR_PROP_BIT_READ | ESP_GATT_CHAR_PROP_BIT_NOTIFY,
                     (uint8_t *)temperature, strlen(temperature),
                     sizeof(temperature), true);

        } else if (param->create.service_id.id.uuid.uuid.uuid16 == SVC_C_UUID) {
            ESP_LOGI(TAG, "Service C (PIN) created");
            svc_c_handle = handle;

            add_char(svc_c_handle, CHAR_C1_UUID,
                     ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE,
                     ESP_GATT_CHAR_PROP_BIT_READ | ESP_GATT_CHAR_PROP_BIT_WRITE,
                     (uint8_t *)pin_code, strlen(pin_code),
                     sizeof(pin_code), false);
        }
        break;
    }

    /* ---------- Characteristic added ---------- */
    case ESP_GATTS_ADD_CHAR_EVT: {
        setup_step++;
        ESP_LOGI(TAG, "Characteristic added (step %d), handle = %d",
                 setup_step, param->add_char.attr_handle);

        switch (setup_step) {
        case 1:
            char_a1_handle = param->add_char.attr_handle;
            add_char(svc_a_handle, CHAR_A2_UUID,
                     ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE,
                     ESP_GATT_CHAR_PROP_BIT_READ | ESP_GATT_CHAR_PROP_BIT_WRITE,
                     (uint8_t *)device_name, strlen(device_name),
                     sizeof(device_name), false);
            break;

        case 2:
            char_a2_handle = param->add_char.attr_handle;
            create_service(gatts_if_for_profile, SVC_B_UUID, SVC_B_NUM_HANDLES);
            break;

        case 3:
            char_b1_handle = param->add_char.attr_handle;

            /* CCCD para notifications de temperatura */
            {
                esp_bt_uuid_t descr_uuid = {
                    .len = ESP_UUID_LEN_16,
                    .uuid.uuid16 = ESP_GATT_UUID_CHAR_CLIENT_CONFIG
                };

                esp_ble_gatts_add_char_descr(
                    svc_b_handle,
                    &descr_uuid,
                    ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE,
                    NULL,
                    NULL
                );
            }
            break;

        case 4:
            char_b2_handle = param->add_char.attr_handle;
            create_service(gatts_if_for_profile, SVC_C_UUID, SVC_C_NUM_HANDLES);
            break;

        case 5:
            char_c1_handle = param->add_char.attr_handle;
            ESP_LOGI(TAG, "All services and characteristics ready");
            break;
        }
        break;
    }

    /* ---------- Descriptor added ---------- */
    case ESP_GATTS_ADD_CHAR_DESCR_EVT:
        ESP_LOGI(TAG, "Descriptor added, handle = %d", param->add_char_descr.attr_handle);
        temp_cccd_handle = param->add_char_descr.attr_handle;

        /* después del CCCD, agregar Command */
        add_char(svc_b_handle, CHAR_B2_UUID,
                 ESP_GATT_PERM_WRITE,
                 ESP_GATT_CHAR_PROP_BIT_WRITE,
                 NULL, 0, 64, false);
        break;

    /* ---------- Write events ---------- */
    case ESP_GATTS_WRITE_EVT: {
        uint16_t handle = param->write.handle;

        char buf[param->write.len + 1];
        memcpy(buf, param->write.value, param->write.len);
        buf[param->write.len] = '\0';

        /* --- CCCD de temperatura --- */
        if (handle == temp_cccd_handle && param->write.len == 2) {
            uint16_t cccd_value = param->write.value[1] << 8 | param->write.value[0];

            if (cccd_value == 0x0001) {
                temp_notify_enabled = true;
                ESP_LOGI(TAG, "[Temperature] Notifications ENABLED");
            } else if (cccd_value == 0x0000) {
                temp_notify_enabled = false;
                ESP_LOGI(TAG, "[Temperature] Notifications DISABLED");
            } else {
                ESP_LOGW(TAG, "[Temperature] Unsupported CCCD value: 0x%04X", cccd_value);
            }

        } else if (handle == char_a2_handle) {
            strncpy(device_name, buf, sizeof(device_name) - 1);
            device_name[sizeof(device_name) - 1] = '\0';
            ESP_LOGI(TAG, "[Device Info] Name updated to: %s", device_name);

        } else if (handle == char_b2_handle) {
            ESP_LOGI(TAG, "[Sensor] Command received: %s", buf);

            if (strcmp(buf, "UPDATE") == 0) {
                temp_value += 0.5;
                snprintf(temperature, sizeof(temperature), "%.1f", temp_value);

                ESP_LOGI(TAG, "[Sensor] Temperature updated to: %s", temperature);

                /* actualiza el atributo interno */
                esp_ble_gatts_set_attr_value(char_b1_handle,
                                             strlen(temperature),
                                             (uint8_t *)temperature);

                /* manda notify si está habilitado */
                if (temp_notify_enabled) {
                    esp_ble_gatts_send_indicate(
                        gatts_if,
                        current_conn_id,
                        char_b1_handle,
                        strlen(temperature),
                        (uint8_t *)temperature,
                        false
                    );
                    ESP_LOGI(TAG, "[Sensor] Notification sent");
                } else {
                    ESP_LOGI(TAG, "[Sensor] Notification not sent (disabled)");
                }
            }

        } else if (handle == char_c1_handle) {
            if (param->write.len != 4) {
                ESP_LOGW(TAG, "[PIN] Invalid length: %d (must be 4)", param->write.len);

                if (param->write.need_rsp) {
                    esp_ble_gatts_send_response(gatts_if,
                                                param->write.conn_id,
                                                param->write.trans_id,
                                                ESP_GATT_INVALID_ATTR_LEN,
                                                NULL);
                }
                break;
            }

            memcpy(pin_code, param->write.value, 4);
            pin_code[4] = '\0';

            ESP_LOGI(TAG, "[PIN] Updated to: %s", pin_code);

        } else {
            ESP_LOGW(TAG, "Write to unknown handle %d", handle);
        }

        if (param->write.need_rsp) {
            esp_ble_gatts_send_response(gatts_if,
                                        param->write.conn_id,
                                        param->write.trans_id,
                                        ESP_GATT_OK, NULL);
        }
        break;
    }

    /* ---------- Read events ---------- */
    case ESP_GATTS_READ_EVT: {
        esp_gatt_rsp_t rsp = {0};
        rsp.attr_value.handle = param->read.handle;

        if (param->read.handle == char_a2_handle) {
            rsp.attr_value.len = strlen(device_name);
            memcpy(rsp.attr_value.value, device_name, rsp.attr_value.len);

        } else if (param->read.handle == char_c1_handle) {
            rsp.attr_value.len = strlen(pin_code);
            memcpy(rsp.attr_value.value, pin_code, rsp.attr_value.len);

        } else {
            break;
        }

        esp_ble_gatts_send_response(gatts_if,
                                    param->read.conn_id,
                                    param->read.trans_id,
                                    ESP_GATT_OK, &rsp);
        break;
    }

    /* ---------- Connection / Disconnection ---------- */
    case ESP_GATTS_CONNECT_EVT:
        ESP_LOGI(TAG, "Central connected");
        current_conn_id = param->connect.conn_id;
        break;

    case ESP_GATTS_DISCONNECT_EVT:
        ESP_LOGI(TAG, "Central disconnected, restarting advertising");
        temp_notify_enabled = false;
        esp_ble_gap_start_advertising(&adv_params);
        break;

    default:
        break;
    }
}

/* ====================== GATTS dispatcher ====================== */
static void gatts_event_handler(esp_gatts_cb_event_t event,
                                esp_gatt_if_t gatts_if,
                                esp_ble_gatts_cb_param_t *param)
{
    if (event == ESP_GATTS_REG_EVT) {
        if (param->reg.status == ESP_GATT_OK) {
            gatts_if_for_profile = gatts_if;
        } else {
            ESP_LOGE(TAG, "GATT app register failed: %d", param->reg.status);
            return;
        }
    }

    gatts_profile_event_handler(event, gatts_if, param);
}

/* ====================== Entry point ====================== */
void app_main(void)
{
    esp_err_t ret;

    ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES ||
        ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    ESP_ERROR_CHECK(esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT));

    esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_bt_controller_init(&bt_cfg));
    ESP_ERROR_CHECK(esp_bt_controller_enable(ESP_BT_MODE_BLE));

    ESP_ERROR_CHECK(esp_bluedroid_init());
    ESP_ERROR_CHECK(esp_bluedroid_enable());

    ESP_ERROR_CHECK(esp_ble_gap_register_callback(gap_event_handler));
    ESP_ERROR_CHECK(esp_ble_gatts_register_callback(gatts_event_handler));
    ESP_ERROR_CHECK(esp_ble_gatts_app_register(ESP_APP_ID));

    ESP_ERROR_CHECK(esp_ble_gatt_set_local_mtu(500));

    ESP_LOGI(TAG, "BLE initialization complete");
}
```

---

### 6) Files & Media

---

### Exercise 2.3 — Service UUID in Advertising Data

#### 1) Activity Objectives    
_For this exercise, a system will be implemented that allows including service UUIDs in the advertising data._  

1) Define service UUIDs in byte format  
2) Include UUIDs in advertising data  
3) Allow filtering by service  
4) Verify visibility before connection  

---

#### 2) Materials & Setup

**BOM (Bill of Materials)**

|#|Item|Qty|Notes|
|---------|--------|------|--------|
|1|ESP32-C6|1|Microcontroller with BLE|
|2|USB Cable|1|Power supply and programming|
|3|Smartphone|1|BLE client device|

---

**_Tools / Software_**   

* _OS/Environment: Windows_  
* _Editors: VS Code / ESP-IDF_  
* _BLE Scanner: nRF Connect or similar_  

**_Wiring / Safety_**  

* _Board power supply: USB connection_  
* _Verify the connection before loading the system_  

---

#### 3) Procedure   

* _**Step 1:** Create an array with the service UUIDs in little-endian format._  
* _**Step 2:** Assign the array to the advertising data structure._  
* _**Step 3:** Configure the length of the UUID data._  
* _**Step 4:** Compile, flash, and scan using a BLE client._  

---

#### 4) Analysis

_The system allows including service identifiers in the advertising data._  

_This enables BLE clients to identify available services before establishing a connection._  

_The use of UUIDs in advertising improves device discovery and filtering._  

_This approach enhances efficiency when working with multiple BLE devices._  

---

#### 5) Code  

```c
#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "esp_bt.h"
#include "esp_gap_ble_api.h"
#include "esp_gatts_api.h"
#include "esp_bt_main.h"
#include "esp_gatt_common_api.h"

static const char *TAG = "BLE_DEMO";

#define DEVICE_NAME              "ESPROY32"
#define PROFILE_NUM              1
#define PROFILE_APP_IDX          0
#define ESP_APP_ID               0x55

/* ── Service A: Device Info ── */
#define SVC_A_UUID               0xAA00
#define CHAR_A1_UUID             0xAA01
#define CHAR_A2_UUID             0xAA02
#define SVC_A_NUM_HANDLES        6

/* ── Service B: Sensor ── */
#define SVC_B_UUID               0xBB00
#define CHAR_B1_UUID             0xBB01
#define CHAR_B2_UUID             0xBB02
#define SVC_B_NUM_HANDLES        6

/* ── Default values ── */
static const char firmware_version[] = "1.0.0";
static char       device_name[32]    = "ESP32C6_BLE_DEMO";
static char       temperature[8]     = "25.0";

/* ── Handles ── */
static uint16_t svc_a_handle;
static uint16_t char_a1_handle;
static uint16_t char_a2_handle;

static uint16_t svc_b_handle;
static uint16_t char_b1_handle;
static uint16_t char_b2_handle;

static int setup_step = 0;
static esp_gatt_if_t gatts_if_for_profile = 0;

/* ── NUEVO: UUIDs de servicios en advertising (little-endian) ── */
/* 0xAA00 -> 0x00, 0xAA */
/* 0xBB00 -> 0x00, 0xBB */
static uint8_t service_uuid128[] = {
    0x00, 0xAA,
    0x00, 0xBB
};

/* ── Advertising ── */
static esp_ble_adv_data_t adv_data = {
    .set_scan_rsp        = false,
    .include_name        = true,
    .include_txpower     = false,
    .min_interval        = 0x20,
    .max_interval        = 0x40,
    .appearance          = 0x00,
    .manufacturer_len    = 0,
    .p_manufacturer_data = NULL,
    .service_data_len    = 0,
    .p_service_data      = NULL,

    /* ── EJERCICIO 2.3 ── */
    .service_uuid_len    = sizeof(service_uuid128),
    .p_service_uuid      = service_uuid128,

    .flag = ESP_BLE_ADV_FLAG_GEN_DISC | ESP_BLE_ADV_FLAG_BREDR_NOT_SPT,
};

static esp_ble_adv_params_t adv_params = {
    .adv_int_min        = 0x20,
    .adv_int_max        = 0x40,
    .adv_type           = ADV_TYPE_IND,
    .own_addr_type      = BLE_ADDR_TYPE_PUBLIC,
    .channel_map        = ADV_CHNL_ALL,
    .adv_filter_policy  = ADV_FILTER_ALLOW_SCAN_ANY_CON_ANY,
};

/* ====================== GAP callback ====================== */
static void gap_event_handler(esp_gap_ble_cb_event_t event,
                              esp_ble_gap_cb_param_t *param)
{
    switch (event) {
    case ESP_GAP_BLE_ADV_DATA_SET_COMPLETE_EVT:
        ESP_LOGI(TAG, "Advertising data configured");
        esp_ble_gap_start_advertising(&adv_params);
        break;

    case ESP_GAP_BLE_ADV_START_COMPLETE_EVT:
        if (param->adv_start_cmpl.status == ESP_BT_STATUS_SUCCESS) {
            ESP_LOGI(TAG, "Advertising started successfully");
        } else {
            ESP_LOGE(TAG, "Advertising start failed");
        }
        break;

    default:
        break;
    }
}

/* ====================== Helpers ====================== */
static void create_service(esp_gatt_if_t gif, uint16_t uuid, uint16_t num_handles)
{
    esp_gatt_srvc_id_t id = {
        .is_primary  = true,
        .id.inst_id  = 0x00,
        .id.uuid.len = ESP_UUID_LEN_16,
        .id.uuid.uuid.uuid16 = uuid,
    };
    esp_ble_gatts_create_service(gif, &id, num_handles);
}

static void add_char(uint16_t svc_handle, uint16_t uuid,
                     esp_gatt_perm_t perm, esp_gatt_char_prop_t prop,
                     uint8_t *value, uint16_t val_len, uint16_t max_len,
                     bool auto_rsp)
{
    esp_bt_uuid_t cuuid = {
        .len         = ESP_UUID_LEN_16,
        .uuid.uuid16 = uuid,
    };

    esp_attr_value_t val = {
        .attr_max_len = max_len,
        .attr_len     = val_len,
        .attr_value   = value,
    };

    esp_attr_control_t ctrl = {
        .auto_rsp = auto_rsp ? ESP_GATT_AUTO_RSP : ESP_GATT_RSP_BY_APP,
    };

    esp_ble_gatts_add_char(svc_handle, &cuuid, perm, prop, &val, &ctrl);
}

/* ====================== GATTS profile callback ====================== */
static void gatts_profile_event_handler(esp_gatts_cb_event_t event,
                                        esp_gatt_if_t gatts_if,
                                        esp_ble_gatts_cb_param_t *param)
{
    switch (event) {

    /* ---------- Registration ---------- */
    case ESP_GATTS_REG_EVT:
        ESP_LOGI(TAG, "GATT server registered");
        gatts_if_for_profile = gatts_if;

        esp_ble_gap_set_device_name(DEVICE_NAME);
        esp_ble_gap_config_adv_data(&adv_data);

        create_service(gatts_if, SVC_A_UUID, SVC_A_NUM_HANDLES);
        break;

    /* ---------- Service created ---------- */
    case ESP_GATTS_CREATE_EVT: {
        uint16_t handle = param->create.service_handle;
        esp_ble_gatts_start_service(handle);

        if (param->create.service_id.id.uuid.uuid.uuid16 == SVC_A_UUID) {
            ESP_LOGI(TAG, "Service A (Device Info) created");
            svc_a_handle = handle;
            add_char(svc_a_handle, CHAR_A1_UUID,
                     ESP_GATT_PERM_READ,
                     ESP_GATT_CHAR_PROP_BIT_READ,
                     (uint8_t *)firmware_version, strlen(firmware_version),
                     sizeof(firmware_version), true);

        } else if (param->create.service_id.id.uuid.uuid.uuid16 == SVC_B_UUID) {
            ESP_LOGI(TAG, "Service B (Sensor) created");
            svc_b_handle = handle;
            add_char(svc_b_handle, CHAR_B1_UUID,
                     ESP_GATT_PERM_READ,
                     ESP_GATT_CHAR_PROP_BIT_READ,
                     (uint8_t *)temperature, strlen(temperature),
                     sizeof(temperature), true);
        }
        break;
    }

    /* ---------- Characteristic added ---------- */
    case ESP_GATTS_ADD_CHAR_EVT: {
        setup_step++;
        ESP_LOGI(TAG, "Characteristic added (step %d), handle = %d",
                 setup_step, param->add_char.attr_handle);

        switch (setup_step) {
        case 1:
            char_a1_handle = param->add_char.attr_handle;
            add_char(svc_a_handle, CHAR_A2_UUID,
                     ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE,
                     ESP_GATT_CHAR_PROP_BIT_READ | ESP_GATT_CHAR_PROP_BIT_WRITE,
                     (uint8_t *)device_name, strlen(device_name),
                     sizeof(device_name), false);
            break;

        case 2:
            char_a2_handle = param->add_char.attr_handle;
            create_service(gatts_if_for_profile, SVC_B_UUID, SVC_B_NUM_HANDLES);
            break;

        case 3:
            char_b1_handle = param->add_char.attr_handle;
            add_char(svc_b_handle, CHAR_B2_UUID,
                     ESP_GATT_PERM_WRITE,
                     ESP_GATT_CHAR_PROP_BIT_WRITE,
                     NULL, 0, 64, false);
            break;

        case 4:
            char_b2_handle = param->add_char.attr_handle;
            ESP_LOGI(TAG, "All services and characteristics ready");
            break;
        }
        break;
    }

    /* ---------- Write events ---------- */
    case ESP_GATTS_WRITE_EVT: {
        uint16_t handle = param->write.handle;

        char buf[param->write.len + 1];
        memcpy(buf, param->write.value, param->write.len);
        buf[param->write.len] = '\0';

        if (handle == char_a2_handle) {
            strncpy(device_name, buf, sizeof(device_name) - 1);
            device_name[sizeof(device_name) - 1] = '\0';
            ESP_LOGI(TAG, "[Device Info] Name updated to: %s", device_name);

        } else if (handle == char_b2_handle) {
            ESP_LOGI(TAG, "[Sensor] Command received: %s", buf);

        } else {
            ESP_LOGW(TAG, "Write to unknown handle %d", handle);
        }

        if (param->write.need_rsp) {
            esp_ble_gatts_send_response(gatts_if,
                                        param->write.conn_id,
                                        param->write.trans_id,
                                        ESP_GATT_OK, NULL);
        }
        break;
    }

    /* ---------- Read events ---------- */
    case ESP_GATTS_READ_EVT: {
        if (param->read.handle == char_a2_handle) {
            esp_gatt_rsp_t rsp = {0};
            rsp.attr_value.handle = param->read.handle;
            rsp.attr_value.len    = strlen(device_name);
            memcpy(rsp.attr_value.value, device_name, rsp.attr_value.len);

            esp_ble_gatts_send_response(gatts_if,
                                        param->read.conn_id,
                                        param->read.trans_id,
                                        ESP_GATT_OK, &rsp);
        }
        break;
    }

    /* ---------- Connection / Disconnection ---------- */
    case ESP_GATTS_CONNECT_EVT:
        ESP_LOGI(TAG, "Central connected");
        break;

    case ESP_GATTS_DISCONNECT_EVT:
        ESP_LOGI(TAG, "Central disconnected, restarting advertising");
        esp_ble_gap_start_advertising(&adv_params);
        break;

    default:
        break;
    }
}

/* ====================== GATTS dispatcher ====================== */
static void gatts_event_handler(esp_gatts_cb_event_t event,
                                esp_gatt_if_t gatts_if,
                                esp_ble_gatts_cb_param_t *param)
{
    if (event == ESP_GATTS_REG_EVT) {
        if (param->reg.status == ESP_GATT_OK) {
            gatts_if_for_profile = gatts_if;
        } else {
            ESP_LOGE(TAG, "GATT app register failed: %d", param->reg.status);
            return;
        }
    }

    gatts_profile_event_handler(event, gatts_if, param);
}

/* ====================== Entry point ====================== */
void app_main(void)
{
    esp_err_t ret;

    ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES ||
        ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    ESP_ERROR_CHECK(esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT));

    esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_bt_controller_init(&bt_cfg));
    ESP_ERROR_CHECK(esp_bt_controller_enable(ESP_BT_MODE_BLE));

    ESP_ERROR_CHECK(esp_bluedroid_init());
    ESP_ERROR_CHECK(esp_bluedroid_enable());

    ESP_ERROR_CHECK(esp_ble_gap_register_callback(gap_event_handler));
    ESP_ERROR_CHECK(esp_ble_gatts_register_callback(gatts_event_handler));
    ESP_ERROR_CHECK(esp_ble_gatts_app_register(ESP_APP_ID));

    ESP_ERROR_CHECK(esp_ble_gatt_set_local_mtu(500));

    ESP_LOGI(TAG, "BLE initialization complete");
}
```

---

### 6) Files & Media