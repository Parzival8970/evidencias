# 📚 **Tarea 2**

## **Introducción**

- **Nombre del proyecto:** _Outputs Basicos_  
- **Equipo / Autor(es):** _Rodrigo Miranda FLores_  
- **Curso / Asignatura:** _Sistemas embebidos 1_  
- **Fecha:** _27/08/2025_  
- **Descripción breve:** _En este apartado se muestran 3 ejercicios un Contador binario 4 bits, un barrido de 5 leds y una secuencia en codigo Gray._

### **Contador binario de 4 bits**


- **Que debe hacer:**
_En cuatro leds debe mostrarse cad segundo la representacion binaria del 0 al 15_

- **Codigo:**
``` 
#include "pico/stdlib.h"
#include "hardware/gpio.h"
 
#define A 0
#define B 1
#define C 2
#define D 3
 
int main() {
    const uint32_t MASK = (1u<<A) | (1u<<B) | (1u<<C) | (1u<<D);
    gpio_init_mask(MASK);
    gpio_set_dir_masked(MASK, MASK);
    while (true) {
        for (uint8_t i = 0; i < 16; i++) {
            gpio_put_masked(MASK, i << A);
            sleep_ms(500);
        }
    }
}
```
- **Esquematico de conexion:**

- **Video:**



### **Barrido de 5 Leds**

- **Que debe hacer:**
_Correr un “1” por cinco LEDs P0..P3 y regresar (0→1→2→3→2→1…)_

- **Codigo:**
``` 
#include "pico/stdlib.h"
#include "hardware/gpio.h"
 
#define LED0 0
#define LED1 1
#define LED2 2
#define LED3 3
 
int main() {
 
    const uint32_t MASK = (1u<<LED0) | (1u<<LED1) | (1u<<LED2) | (1u<<LED3);
 
    gpio_init_mask(MASK);
    gpio_set_dir_masked(MASK, MASK);
 
    int pos = 0;
    int dir = 1;
 
    while (true) {
 
        gpio_put_masked(MASK, (1u << pos));
 
        sleep_ms(200);
 
        pos += dir;
 
        if (pos == 3) dir = -1;
        else if (pos == 0) dir = 1;
    }
}
```

- **Esquematico de conexion:**

- **Video:**



### **Secuencia en codigo Gray**

- **Que debe hacer:**
_En cuatro leds debe mostrarse cad segundo la representacion en codigo Gray del 0 al 15_

- **Codigo:**
```
#include "pico/stdlib.h"
#include "hardware/gpio.h"
 
#define LED0 0
#define LED1 1
#define LED2 2
#define LED3 3
 
uint8_t binario_a_gray(uint8_t num) {
    return num ^ (num >> 1);
}
 
int main() {
    const uint32_t MASK = (1u<<LED0) | (1u<<LED1) | (1u<<LED2) | (1u<<LED3);
 
    gpio_init_mask(MASK);
    gpio_set_dir_masked(MASK, MASK);
 
    while (true) {
        for (uint8_t i = 0; i < 16; i++) {  
            uint8_t gray = binario_a_gray(i);
            gpio_put_masked(MASK, gray);
            sleep_ms(500);
        }
    }
}
```

- **Esquematico de conexion:**

- **Video:**