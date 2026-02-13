# Unidad 2

## Bitácora de proceso de aprendizaje

### Actividad 1: ¿Cómo funciona mi programa?
Los Estados (¿Qué está pasando?): Imagina que el programa tiene "ánimos" o momentos. El estado waitingInON es cuando el programa está en modo "encendido" y el waitingInOFF es cuando está en "pausa". El programa salta de uno a otro según lo que vaya pasando.

Los Eventos (El reloj): Son los avisos que le dicen al programa cuándo cambiar. El más importante aquí es el Timeout, que funciona como una alarma o cuenta regresiva. Es el que decide cuánto tiempo se quedan las luces prendidas o apagadas antes de que algo más suceda.

Las Acciones (Lo que vemos y oímos): Son las tareas concretas que el programa hace cuando llega el momento. Por ejemplo: encender los colores de los pixeles, dibujar formas en la pantalla o hacer que suene un pitido.

### Actividad 2:
Codigo del cemaforo modificado:

```
from microbit import *
import utime

class Timer:
def __init__(self, owner, event_to_post, duration):
    self.owner = owner
    self.event = event_to_post
    self.duration = duration
    self.start_time = 0
    self.active = False

def start(self, new_duration=None):
    if new_duration is not None:
        self.duration = new_duration
    self.start_time = utime.ticks_ms()
    self.active = True

def stop(self):
    self.active = False

def update(self):
    if self.active:
        if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
            self.active = False
            self.owner.post_event(self.event)

class Semaforo:
def __init__(self, _x, _y, _timeInRed, _timeInGreen, _timeInYellow):
    self.event_queue = []
    self.timers = []
    self.x = _x
    self.y = _y
    self.timeInRed = _timeInRed
    self.timeInGreen = _timeInGreen
    self.timeInYellow = _timeInYellow
    self.myTimer = self.createTimer("Timeout", self.timeInRed)
    self.estado_actual = None
    self.transicion_a(self.estado_waitInRed)

def createTimer(self, event, duration):
    t = Timer(self, event, duration)
    self.timers.append(t)
    return t

def post_event(self, ev):
    self.event_queue.append(ev)

def update(self):
    for t in self.timers:
        t.update()
    while len(self.event_queue) > 0:
        ev = self.event_queue.pop(0)
        if self.estado_actual:
            self.estado_actual(ev)

def transicion_a(self, nuevo_estado):
    if self.estado_actual:
        self.estado_actual("EXIT")
    self.estado_actual = nuevo_estado
    self.estado_actual("ENTRY")

def clear(self):
    display.set_pixel(self.x, self.y, 0)
    display.set_pixel(self.x, self.y+1, 0)
    display.set_pixel(self.x, self.y+2, 0)

def estado_waitInRed(self, ev):
    if ev == "ENTRY":
        self.clear()
        display.set_pixel(self.x, self.y, 9)
        self.myTimer.start(self.timeInRed)
    if ev == "Timeout":
        display.set_pixel(self.x, self.y, 0)
        self.transicion_a(self.estado_waitInGreen)

def estado_waitInGreen(self, ev):
    if ev == "ENTRY":
        self.clear()
        display.set_pixel(self.x, self.y+2, 9)
        self.myTimer.start(self.timeInGreen)
    if ev == "Timeout":
        display.set_pixel(self.x, self.y+2, 0)
        self.transicion_a(self.estado_waitInYellow)
    if ev == "A":
        self.myTimer.stop()
        display.set_pixel(self.x, self.y+2, 0)
        self.transicion_a(self.estado_waitInYellow)

def estado_waitInYellow(self, ev):
    if ev == "ENTRY":
        self.clear()
        display.set_pixel(self.x, self.y+1, 9)
        self.myTimer.start(self.timeInYellow)
    if ev == "Timeout":
        display.set_pixel(self.x, self.y+1, 0)
        self.transicion_a(self.estado_waitInRed)

semaforo1 = Semaforo(0, 0, 2000, 1000, 500)

while True:
if button_a.is_pressed():
    semaforo1.post_event("A")
semaforo1.update()
utime.sleep_ms(20)

```
Diagrama UML:

<img width="194" height="397" alt="image" src="https://github.com/user-attachments/assets/e27b31b4-7428-4865-ba66-c399a0e9bdcf" />

### Actividad 3:

No siempre está claro el flujo que sigue el programa. Hay momentos en los que ver tanto estado junto marea.


## Bitácora de aplicación 

## Actividad 4:
#### Maquina de Estados:

<img width="1294" height="602" alt="image" src="https://github.com/user-attachments/assets/d1e8391e-2623-44d0-889b-9864ab76565b" />

# Codigo:

```
from microbit import *
import utime
import music

def make_fill_images(on='9', off='0'):
imgs = []
for n in range(26):
    rows = []
    k = 0
    for y in range(5):
        row = []
        for x in range(5):
            row.append(on if k < n else off)
            k += 1
        rows.append(''.join(row))
    imgs.append(Image(':'.join(rows)))
return imgs

FILL = make_fill_images()

class Timer:
def __init__(self, owner, event_to_post, duration):
    self.owner = owner
    self.event = event_to_post
    self.duration = duration
    self.start_time = 0
    self.active = False

def start(self, new_duration=None):
    if new_duration is not None:
        self.duration = new_duration
    self.start_time = utime.ticks_ms()
    self.active = True

def stop(self):
    self.active = False

def update(self):
    if self.active:
        if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
            self.active = False
            self.owner.post_event(self.event)

class EscapeTimer:
def __init__(self):
    self.event_queue = []
    self.timers = []
    self.time_pixels = 20
    self.max_pixels = 25
    self.min_pixels = 15
    self.current_pixel = 0
    self.myTimer = self.createTimer("Timeout", 1000)
    self.estado_actual = None
    self.transicion_a(self.estado_config)

def createTimer(self,event,duration):
    t = Timer(self, event, duration)
    self.timers.append(t)
    return t

def post_event(self, ev):
    self.event_queue.append(ev)

def update(self):
    for t in self.timers:
        t.update()
    while len(self.event_queue) > 0:
        ev = self.event_queue.pop(0)
        if self.estado_actual:
            self.estado_actual(ev)

def transicion_a(self, nuevo_estado):
    if self.estado_actual:
        self.estado_actual("EXIT")
    self.estado_actual = nuevo_estado
    self.estado_actual("ENTRY")

def estado_config(self, ev):
    if ev == "ENTRY":
        self.current_pixel = self.time_pixels
        display.show(FILL[self.current_pixel])
    if ev == "A":
        if self.current_pixel < self.max_pixels:
            self.current_pixel += 1
            display.show(FILL[self.current_pixel])
    if ev == "B":
        if self.current_pixel > self.min_pixels:
            self.current_pixel -= 1
            display.show(FILL[self.current_pixel])
    if ev == "S":
        self.time_pixels = self.current_pixel
        self.transicion_a(self.estado_armed)

def estado_armed(self, ev):
    if ev == "ENTRY":
        self.current_pixel = self.time_pixels
        display.show(FILL[self.current_pixel])
        self.myTimer.start(1000)
    if ev == "Timeout":
        if self.current_pixel > 0:
            self.current_pixel -= 1
            display.show(FILL[self.current_pixel])
            self.myTimer.start(1000)
        else:
            self.transicion_a(self.estado_alarm)
    if ev == "B":
        self.transicion_a(self.estado_config)

def estado_alarm(self, ev):
    if ev == "ENTRY":
        display.show(Image.SKULL)
        for i in range(3):
            music.play(music.POWER_DOWN, wait=False)
    if ev == "A":
        self.time_pixels = 20
        self.transicion_a(self.estado_config)

 task = EscapeTimer()

while True:
if button_a.was_pressed():
    task.post_event("A")
if button_b.was_pressed():
    task.post_event("B")
if accelerometer.was_gesture("shake"):
    task.post_event("S")
task.update()
utime.sleep_ms(20)
```
## Bitácora de reflexión
