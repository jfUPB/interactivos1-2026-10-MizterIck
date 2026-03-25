# Unidad 5
## Bitácora de proceso de aprendizaje
### Actividad 1
**¿Qué ventajas y desventajas ves en usar un formato binario en lugar de texto ASCII?**
- La ventaja principal es la optimización del uso de la memoria en códigos complejos a comparación del ASCII y la mayor desventaja es que a cambio de la eficiencia de este método, así mismo aumenta la dificultad y complejidad a la hora de programar.

**Si xValue=500, yValue=524, aState=True, bState=False, ¿cómo se vería el paquete en hexadecimal? (Pista: convierte cada valor según su tipo y anota los bytes en orden.)**
- 01 F4 02 0C 01 00

**¿Por qué el protocolo ASCII de la unidad anterior no tenía este problema de sincronización? (Pista: piensa en qué rol cumplía el carácter \n.)**
- Debido a que el \n cumplía como un delimitador, lo cúal, permitia que el proframa pudiera saber en que punto comenzaba la información con la que tenía que trabajar.

**¿Por qué en binario no podemos usar \n como delimitador?**
- Porque el Enter deja de ser un carácter válido en binario para ser usado como delimitador, este pasa a ser un valor más.


## Bitácora de aplicación 
### Actividad 2

<img width="1910" height="933" alt="image" src="https://github.com/user-attachments/assets/8b8581e9-eb2d-45f1-841b-a4533192de4a" />

Microbit

``` python

from microbit import *

uart.init(115200)
display.set_pixel(0,0,9)

HEADER = 0xAA

def int16_to_bytes(value):
    # convertir enteros con signos a 2 bytes big endian

    if value < 0:
        value = 65536 + value
        
    high = (value >> 8) & 0xFF
    low = value & 0xFF

    return high, low

while True:
    
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()

    if xValue < -2048:
        xValue = -2048
    if xValue > 2047:
        xValue = 2047

    if yValue < -2048:
        yValue = -2048
    if yValue > 2047:
        yValue = 2047
    
    aState = 1 if button_a.is_pressed() else 0
    bState = 1 if button_b.is_pressed() else 0

    #convertir X y Y a 2 bytes cada uno
    xHigh, xLow = int16_to_bytes(xValue)
    yHigh, yLow = int16_to_bytes(yValue)
    
    checksum = (xHigh + xLow + yHigh + yLow + aState + bState) % 256
    
    packet = bytes([HEADER,xHigh,xLow,yHigh,yLow,aState,bState,checksum])
    
    uart.write(packet)
    
    sleep(100) # Envia datos a 10 Hz
```
``` js

```
## Bitácora de reflexión
