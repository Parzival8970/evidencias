# Task exercise

**_The purpose of these labs 1-3 is to analyze changes in our code or new behaviors_**

## Lab 1 

### 1) Activity Goals  

**_Activity Goals_**

+ _Implement a multitasking system in FreeRTOS using two concurrent tasks for LED control and serial message output._

+ _Validate the effect of task priorities and task blocking on the behavior of the FreeRTOS scheduler, as well as the prevention of starvation through the use of vTaskDelay._

+ _Document the source code, the results of the conducted experiments, and the observations obtained regarding concurrent task execution._

## 2)Exercises  
+ _Task 1: Heartbeat_
+ _Task 2: Alive task_
+ _Task 3: Queue Struct Send_
+ _Task 4: Queue Struct Receive_
+ _Task 5: and 6 Mutex reading a button_
+ _Task 7: Error loggin for task 1-6_

## 3) Materials & Setup  
BOM (bill of materials)

|#|Item|Qty|Link/Source|Cost (MXN)|Notes|
|---------|--------|------|--------|--------|--------|
|1|ESP32|1|amazon|$365|Nothing|
|2|Led|1|Electronic store|$3|Nothing|
|3|Push button|2|Electronic store|$2|Nothing|

**_Tools / Software_**  
OS/Env: ESP-IDF with FreeRTOS on ESP32 (Windows)  
Editors: VS Code with ESP-IDF extension, C/C++ 
Debug/Flash: ESP-IDF 

**_Wiring / Safety_**  
Board power: USB 5 V from host PC  
LED: Onboard LED (GPIO 2)  
Safety notes: Verify correct GPIO pin for the board; avoid short circuits during wiring  

## 4) Procedure (what you did)  
Step 1: Create a new ESP-IDF project and configure the target ESP32 board  

Step 2: Implement two FreeRTOS tasks: one for LED blinking and one for serial logging  

Step 3: Build, flash, and monitor the application using ESP-IDF tools  

Step 4: Modify task priorities and observe scheduler behavior  

Step 5: Remove and restore vTaskDelay in one task to demonstrate starvation and recovery  

Step 6: Verify correct LED blinking and periodic serial output   

## 5) Data, Tests & Evidence  
Test plan  
Inputs: Task priorities, presence or absence of vTaskDelay  
Expected: Stable LED blinking and periodic log output when tasks are properly blocked  

Results:  

Tables/observations  
Case | Configuration | LED Behavior | Serial Output | Pass?  
A | Both tasks priority 5, delays enabled | Normal blinking | Periodic |   
B | hello_task priority 2, delays enabled | Normal blinking | Periodic |   
C | hello_task without vTaskDelay | Yes / unstable blinking | Continuous output | 
D | vTaskDelay restored | Normal blinking | Periodic | 

## 6) Analysis  
Observed behavior matches the expected FreeRTOS scheduling model.  
When tasks use vTaskDelay, they enter the Blocked state, allowing the scheduler to execute other ready tasks regardless of priority differences.  
Removing vTaskDelay causes one task to continuously occupy the CPU, leading to starvation of the LED task.  

This behavior illustrates the importance of cooperative blocking in real-time multitasking systems.  

Proposed fixes: Ensure all tasks include appropriate blocking calls (vTaskDelay, queues, or synchronization primitives).  

## 7) Code  
```
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO GPIO_NUM_2   // CHANGE for your board

static const char *TAG = "LAB1";

static void blink_task(void *pvParameters)
{
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    while (1) {
        gpio_set_level(LED_GPIO, 1);
        vTaskDelay(pdMS_TO_TICKS(300));
        gpio_set_level(LED_GPIO, 0);
        vTaskDelay(pdMS_TO_TICKS(300));
    }
}

static void hello_task(void *pvParameters)
{
    int n = 0;
    while (1) {
        ESP_LOGI(TAG, "hello_task says hi, n=%d", n++);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Starting Lab 1 (two tasks)");

    // Stack size in ESP-IDF FreeRTOS is in BYTES
    xTaskCreate(blink_task, "blink_task", 2048, NULL, 5, NULL);
    xTaskCreate(hello_task, "hello_task", 2048, NULL, 5, NULL);
}
```


## 8) Files & Media   

**connection diagram:**   
  ![connection diagram](.png)
  
**Video:**
  
  <div style="position: relative; width: 100%; height: 0; padding-top: 56.25%; margin-bottom: 1em;">
    <iframe src="https://youtube.comQ9D2mdsE"
            style="position: absolute; width: 100%; height: 100%; top: 0; left: 0; border: none;"
            allowfullscreen>
    </iframe>
  </div>
