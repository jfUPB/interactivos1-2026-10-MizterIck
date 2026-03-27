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

Procedimiento:

Primeramente las bases conceptuales sobre las que se hicieron los cambios al código fueron las siguientes:

| Concepto    | ASCII V2  | Binario          |
| ----------- | --------- | ---------------- |
| Entrada     | Texto     | Bytes            |
| Buffer      | String    | Buffer           |
| Delimitador | `\n`      | Header `0xAA`    |
| Parsing     | `split()` | posiciones fijas (8 bytes) |
| Lectura     | por texto | por bytes        |
| Complejidad | Baja      | Media            |

Ahora sí, lo primero que se hace es descomentar la linea de importación del adapter binario dentro del BridgeServer.

En este caso, para hacer la menor cantidad de cambios posibles, solo hubo dos cambios importantes dentro de las funciones de la respectiva clase de cada adaptador:
- _onChunk:

Aquí se hacen cambios desde la lógica de que el binario no maneja strings, por lo tanto el tipo de buffer cambia a bytes y el uso de delimitador se transforma en buscar la cabeza del paquete "0xAA" y agregando un límite fijo de 8 bytes para leer (en caso de estar completos), acto seguido se extraen exactamente esos 8 bits, se procesan y despues se borran. Despues de esto se llama al _parsePacket().

- _parsePacket():

Este decodifica los datos de x, y, btnA, btnB y checksum en Big Endian y verifica si el checksum es correcto, de ser así entrega el objeto parseado.

``` js
 _onChunk(chunk) {
    this.buf = Buffer.concat([this.buf, chunk]);

    while (this.buf.length >= 8) {

      // Buscar header 0xAA
      const start = this.buf.indexOf(0xAA);

      if (start === -1) {
        // No hay header, limpiar buffer
        this.buf = Buffer.alloc(0);
        return;
      }

      // Si no hay suficientes bytes aún
      if (this.buf.length < start + 8) {
        return;
      }

      const packet = this.buf.slice(start, start + 8);

      // Remover paquete del buffer
      this.buf = this.buf.slice(start + 8);

      try {
        const parsed = this._parsePacket(packet);
        if (parsed) {
          this.onData?.(parsed);
        }
      } catch (e) {
        if (this.verbose) {
          console.log("Bad binary packet:", e.message, packet);
        }
      }
    }
  }
  
  _parsePacket(packet) {
    // packet[0] = 0xAA

    const x = packet.readInt16BE(1);
    const y = packet.readInt16BE(3);
    const btnA = packet[5];
    const btnB = packet[6];
    const chk = packet[7];

    // Calcular checksum
    let calc = 0;
    for (let i = 1; i <= 6; i++) {
      calc += packet[i];
    }
    calc = calc % 256;

    if (calc !== chk) {
      throw new Error("Checksum mismatch");
    }

    return {
      x: x,
      y: y,
      btnA: btnA === 1,
      btnB: btnB === 1
    };
  }
```

Adicionalmente de forma global para la estructura del Adaptador binario, se debe cambiar la cualquier linea:
``` js
    this.buf = "";
```

por una

``` js
    this.buf = Buffer.alloc(0);
```

Esto ya que la primera está adaptada para que el buffer sean strings y no bytes como la segunda.

Resultado:
<img width="1910" height="933" alt="image" src="https://github.com/user-attachments/assets/8b8581e9-eb2d-45f1-841b-a4533192de4a" />

Problematicas:

Borré descuidadamente la totalidad de la función Parse original para acoplarme al formato de binario y dentro de esta función yo tenía el checksum, así que al no tener un checksum como tal en visual studio, me generaba de manera continua trama corrupta, al darme cuenta recuperé la linea de código del checksum, para que funcionara adecuadamente la ubiqué dentro de la nueva funcion "_parsepacket" y la adapté al método binario.
<img width="975" height="387" alt="image" src="https://github.com/user-attachments/assets/51a24aa6-6c75-4448-8e48-98430eb441b2" />

AdaptadorBinario

``` js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

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
    this.buf = Buffer.concat([this.buf, chunk]);

    while (this.buf.length >= 8) {

      // Buscar header 0xAA
      const start = this.buf.indexOf(0xAA);

      if (start === -1) {
        // No hay header, limpiar buffer
        this.buf = Buffer.alloc(0);
        return;
      }

      // Si no hay suficientes bytes aún
      if (this.buf.length < start + 8) {
        return;
      }

      const packet = this.buf.slice(start, start + 8);

      // Remover paquete del buffer
      this.buf = this.buf.slice(start + 8);

      try {
        const parsed = this._parsePacket(packet);
        if (parsed) {
          this.onData?.(parsed);
        }
      } catch (e) {
        if (this.verbose) {
          console.log("Bad binary packet:", e.message, packet);
        }
      }
    }
  }
  
  _parsePacket(packet) {
    // packet[0] = 0xAA

    const x = packet.readInt16BE(1);
    const y = packet.readInt16BE(3);
    const btnA = packet[5];
    const btnB = packet[6];
    const chk = packet[7];

    // Calcular checksum
    let calc = 0;
    for (let i = 1; i <= 6; i++) {
      calc += packet[i];
    }
    calc = calc % 256;

    if (calc !== chk) {
      throw new Error("Checksum mismatch");
    }

    return {
      x: x,
      y: y,
      btnA: btnA === 1,
      btnB: btnB === 1
    };
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
## Bitácora de reflexión
