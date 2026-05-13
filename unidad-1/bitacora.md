# Unidad 1
## Bitácora de proceso de aprendizaje
### Actividad 01
¿Qué es un sistema físico interactivo?
Un sistema físico interactivo es una combinación de componentes físicos (como sensores, pantallas, luces, etc.) con un componente digital (software) y lógica que permiten a una persona o un entorno influir en el comportamiento de un espacio en tiempo real.

¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
Teniendo en cuenta que yo pienso ir por la rama de videojuegos, todo el contenido visto puede tener una aplicación directa en videojuegos que utilicen realidad virtual o realidad aumentada.

### Actividad 02
¿Qué es el diseño/arte generativo?
El diseño generativo es un proceso creativo en el que se usan reglas, algoritmos y datos para que un sistema (generalmente una computadora) genere resultados visuales, estructurales o narrativos de forma autónoma o semi-autónoma.

¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
Se pueden utilizar dentro de un videojuego para:
- Crear mundos, niveles o paisajes que cambian automáticamente,
- Generar patrones visuales o efectos que reaccionan a la interacción del jugador,
- Programar música o sonido adaptativo que evoluciona durante el juego.

### Actividad 04
¿Por qué no funcionaba el programa con was_pressed() y por qué funciona con is_pressed()?
Si usas was_pressed(), el programa no podría enviar múltiples mensajes si el botón se mantiene presionado, eso quiere decir que en caso de mantener presionado el botón el color no se mantendría.

## Bitácora de aplicación 

### Actividad 05
Programa p5.js

``` js
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

``` python
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


Yo creo una variable PositionX en la cual inicializo el valor de la posición en X y la coloco dentro de los valores del elipse donde corresponde, luego dentro de los IF del botón A y B modifico el valor
de la variable PositionX en +15/-15 en cada botón para así modificar la posición del círculo en el canvas.

## Bitácora de reflexión

### Actividad 06

El código de MicroBit crea como tal los botones con los que se puede interactuar dentro del sistema, en este caso solo el botón "A" y le aplica un cooldown para presionar el botón

``` python
from microbit import *

uart.init(baudrate=115200)

while True:

    if button_a.was_pressed():
        uart.write('A')
    else:
        uart.write('N')

    sleep(100)
```

El código de p5.js por otro lado le asigna al botón habilitado una función específica, pero antes de eso crea un canvas en el que se coloca una figura, en esta ocación un cuadrado y a la vez se crea un botón para conectar al micro:bit y se le asigna una posición en el canvas. La función específica en este caso de los botones es el hacer cambiar de color el cuadrado a ser rojo en caso de presionar el botón A y en caso de no tenerlo presionado el cuadrado se mantiene en el color verde.

``` python
  let port;
  let connectBtn;
  let connectionInitialized = false;

  function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton("Connect to micro:bit");
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
  }

  function draw() {
    background(220);

    if (port.opened() && !connectionInitialized) {
      port.clear();
      connectionInitialized = true;
    }

    if (port.availableBytes() > 0) {
      let dataRx = port.read(1);
      if (dataRx == "A") {
        fill("red");
      } else if (dataRx == "N") {
        fill("green");
      }
    }

    rectMode(CENTER);
    rect(width / 2, height / 2, 50, 50);

    if (!port.opened()) {
      connectBtn.html("Connect to micro:bit");
    } else {
      connectBtn.html("Disconnect");
    }
  }

  function connectBtnClick() {
    if (!port.opened()) {
      port.open("MicroPython", 115200);
      connectionInitialized = false;
    } else {
      port.close();
    }
  }
```

