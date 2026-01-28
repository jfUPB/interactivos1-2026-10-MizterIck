# Unidad 1
## Bitácora de proceso de aprendizaje
### Actividad 01
El zapato
<img width="300" height="144" alt="image" src="https://github.com/user-attachments/assets/da27d4fa-5bf0-4176-89ff-c8f9827e09cb" />

### Actividad 05
Programa p5.js

```
let port;
let connectBtn;

function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
    fill('white');
    PositionX = width / 2;
    ellipse(PositionX, height / 2, 100, 100);
}

function draw() {

    if(port.availableBytes() > 0)
{
        let dataRx = port.read(1);
        if(dataRx == 'A')
{
            fill('red');
            PositionX = PositionX + 15;
        }
        else if(dataRx == 'B')
{
            fill('yellow');
            PositionX = PositionX - 15;
        }
        background(220);
        ellipse(PositionX, height / 2, 100, 100);
        fill('black');
        text(dataRx, PositionX, height / 2);
    }


    if (!port.opened()) {
        connectBtn.html('Connect to micro:bit');
    }
    else {
        connectBtn.html('Disconnect');
    }
}

function connectBtnClick() {
    if (!port.opened()) {
        port.open('MicroPython', 115200);
    } else {
        port.close();
    }
}
```

Código Micro:

```
from microbit import *

uart.init(baudrate=115200)

while True:
    if button_a.is_pressed():
        uart.write('A')
        sleep(250)
    if button_b.is_pressed():
        uart.write('B')
        sleep(250)
```
## Bitácora de aplicación 



## Bitácora de reflexión

