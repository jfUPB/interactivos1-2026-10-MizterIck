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
Borré descuidadamente la totalidad de la función Parse original para acoplarme al formato de binario y dentro de esta función yo tenía el checksum, así que al no tener un checksum como tal en visual studio, me generaba de manera continua trama corrupta, al darme cuenta recuperé la linea de código del checksum y arreglé el problema
<img width="975" height="387" alt="image" src="https://github.com/user-attachments/assets/51a24aa6-6c75-4448-8e48-98430eb441b2" />

AdaptadorBinario

``` js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

const x = Number(values[1].split(":")[1]);
const y = Number(values[2].split(":")[1]);
const btnA = Number(values[3].split(":")[1]);
const btnB = Number(values[4].split(":")[1]);
const chk = Number(values[5].split(":")[1]);

const calcChk = Math.abs(x) + Math.abs(y) + btnA + btnB;

if (calcChk !== chk) {
    throw new ParseError("Checksum no coincide");
  }

class MicrobitBinaryAdapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;
    this.buf = Buffer.alloc(0);
    this.verbose = verbose;
  }

  async connect() {
    if (this.connected) return;
    if (!this.path) throw new Error("serialPort is required for microbit device mode");

    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    await new Promise((resolve, reject) => {
      this.port.open((err) => (err ? reject(err) : resolve()));
    });

    this.connected = true;
    this.onConnected?.(`serial open ${this.path} @${this.baud}`);

    this.port.on("data", (chunk) => this._onChunk(chunk));
    this.port.on("error", (err) => this._fail(err));
    this.port.on("close", () => this._closed());
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;

    if (this.port && this.port.isOpen) {
      await new Promise((resolve, reject) => {
        this.port.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
    }

    this.port = null;
    this.buf = Buffer.alloc(0);
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

 _onChunk(chunk) {
  // Asegura que buf sea un Buffer
  if (!this.buf) this.buf = Buffer.alloc(0);

  // Concatenar buffers
  this.buf = Buffer.concat([this.buf, chunk]);

  let idx;
  while ((idx = this.buf.indexOf(0x0A)) >= 0) { // '\n'
    // Extraer línea (sin incluir \n)
    const lineBuf = this.buf.subarray(0, idx);
    this.buf = this.buf.subarray(idx + 1);

    if (lineBuf.length === 0) continue;

    try {
      // Convertir SOLO la línea actual a string
      const line = lineBuf.toString("utf8").trim();
      if (!line) continue;

      const parsed = parseCsvLine(line);
      this.onData?.(parsed);
    } catch (e) {
      if (e instanceof ParseError) {
        if (this.verbose) {
          console.log("Bad data:", e.message, "raw:", lineBuf);
        }
      } else {
        this._fail(e);
      }
    }
  }

  // Protección contra crecimiento infinito
  if (this.buf.length > 4096) {
    this.buf = Buffer.alloc(0);
  }
}

  

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = Buffer.alloc(0);
    this.onDisconnected?.("serial closed (event)");
  }

  async writeLine(line) {
    if (!this.port || !this.port.isOpen) return;
    await new Promise((resolve, reject) => {
      this.port.write(line, (err) => (err ? reject(err) : resolve()));
    });
  }

  async handleCommand(cmd) {
    if (cmd?.cmd === "setLed") {
      const x = Math.max(0, Math.min(4, Math.trunc(cmd.x)));
      const y = Math.max(0, Math.min(4, Math.trunc(cmd.y)));
      const v = Math.max(0, Math.min(9, Math.trunc(cmd.value)));
      await this.writeLine(`LED,${x},${y},${v}\n`);
    }
  }
}

module.exports = MicrobitBinaryAdapter;

```
## Bitácora de reflexión
