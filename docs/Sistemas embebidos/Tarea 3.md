# 📚 **Tarea 3**

## **Introducción**

- **Nombre del proyecto:** _Inputs_  
- **Equipo / Autor(es):** _Rodrigo Miranda Flores_  
- **Curso / Asignatura:** _Sistemas embebidos 1_  
- **Fecha:** _01/09/2025_  
- **Descripción breve:** _En este apartado se muestran  2 ejercicios 3 compuertas básicas AND / OR / XOR con 2 botones y un selector cíclico de 4 LEDs con avance/retroceso._
### **3 compuertas básicas AND / OR / XOR con 2 botones**

- **Que debe hacer:**
_Con dos botones A y B (pull-up; presionado=0) enciende tres LEDs que muestren en paralelo los resultados de AND, OR y XOR. En el video muestra las 4 combinaciones (00, 01, 10, 11)._

- **Codigo:**
Codigo de compuerta AND
```
#include "pico/stdlib.h"
#include "hardware/gpio.h"
 
#define BTN_A 2
#define BTN_B 3
#define LED_AND 6
 
int main() {
    stdio_init_all();
 
    gpio_init(BTN_A); gpio_set_dir(BTN_A, false); gpio_pull_up(BTN_A);
    gpio_init(BTN_B); gpio_set_dir(BTN_B, false); gpio_pull_up(BTN_B);
 
    gpio_init(LED_AND); gpio_set_dir(LED_AND, true);
 
    while (true) {
        bool a = !gpio_get(BTN_A);
        bool b = !gpio_get(BTN_B);
 
        bool result = a && b;
 
        gpio_put(LED_AND, result);
        sleep_ms(50);
    }
}
```

**Codigo:**
Codigo de compuerta OR
```
#include "pico/stdlib.h"
#include "hardware/gpio.h"
 
#define BTN_A 2
#define BTN_B 3
#define LED_OR 7
 
int main() {
    stdio_init_all();
 
    gpio_init(BTN_A); gpio_set_dir(BTN_A, false); gpio_pull_up(BTN_A);
    gpio_init(BTN_B); gpio_set_dir(BTN_B, false); gpio_pull_up(BTN_B);
 
  
    gpio_init(LED_OR); gpio_set_dir(LED_OR, true);
 
    while (true) {
        bool a = !gpio_get(BTN_A);
        bool b = !gpio_get(BTN_B);
 
        bool result = a || b;
 
        gpio_put(LED_OR, result);
        sleep_ms(50);
    }
}
```
**Codigo:**
Codigo de compuerta XOR
```
#include "pico/stdlib.h"
#include "hardware/gpio.h"
 
#define BTN_A 2
#define BTN_B 3
#define LED_XOR 8
 
int main() {
    stdio_init_all();
 
    gpio_init(BTN_A); gpio_set_dir(BTN_A, false); gpio_pull_up(BTN_A);
    gpio_init(BTN_B); gpio_set_dir(BTN_B, false); gpio_pull_up(BTN_B);
 
    gpio_init(LED_XOR); gpio_set_dir(LED_XOR, true);
 
    while (true) {
        bool a = !gpio_get(BTN_A);
        bool b = !gpio_get(BTN_B);
 
        bool result = a ^ b;
 
        gpio_put(LED_XOR, result);
        sleep_ms(50);
    }
}
```
- **Esquematico de conexion:** Se usó el mismo circuito para las 3 compuertas

- **Video:**


### **Selector cíclico de 4 LEDs con avance/retroceso**

- **Que debe hacer:**
_Mantén un único LED encendido entre LED0..LED3. Un botón AVANZA (0→1→2→3→0) y otro RETROCEDE (0→3→2→1→0). Un push = un paso (antirrebote por flanco: si dejas presionado no repite). En el video demuestra en ambos sentidos._

- **Codigo:**
```

```

- **Esquematico de conexion:**

- **Video:**