# Unidad 4

## Bitácora de proceso de aprendizaje
### Actividad 01
<img width="994" height="460" alt="image" src="https://github.com/user-attachments/assets/1e48705a-7a53-4665-9cba-98ad21600b8a" />````

``` bash
ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~
$ mkdir sfi1JP //crea carpeta

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~
$ cd sfi1JP //ingresa la carpeta al sistema

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP
$ git clone ^[[200~https://github.com/juanferfranco/sfi1-2026-20-u4-CaseStudy.git~
Cloning into 'sfi1-2026-20-u4-CaseStudy.git~'...
fatal: protocol '?[200~https' is not supported

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP
$ git clone https://github.com/juanferfranco/sfi1-2026-20-u4-CaseStudy.git //clona un repositorio de github
Cloning into 'sfi1-2026-20-u4-CaseStudy'...
remote: Enumerating objects: 17, done.
remote: Counting objects: 100% (17/17), done.
remote: Compressing objects: 100% (15/15), done.
remote: Total 17 (delta 0), reused 17 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (17/17), 10.65 KiB | 5.33 MiB/s, done.

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP
$ ls
sfi1-2026-20-u4-CaseStudy/

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP
$ cd ^C

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP
$ cd sfi1-2026-20-u4-CaseStudy/ //ingresa a carpeta del repositorio clonado

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP/sfi1-2026-20-u4-CaseStudy (main)
$ ls //muestra el interior de las carpetas
adapters/  bridgeClient.js  bridgeServer.js  fsm.js  index.html  jsconfig.json  package-lock.json  package.json  sketch.js  style.css

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP/sfi1-2026-20-u4-CaseStudy (main)
$ ls
adapters/  bridgeClient.js  bridgeServer.js  fsm.js  index.html  jsconfig.json  package-lock.json  package.json  sketch.js  style.css

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP/sfi1-2026-20-u4-CaseStudy (main)
$ node bridgeserver.js
node:internal/modules/cjs/loader:1423
  throw err;
  ^

Error: Cannot find module 'ws'
Require stack:
- C:\Users\ESTUDIANTES\sfi1JP\sfi1-2026-20-u4-CaseStudy\bridgeserver.js
    at Module._resolveFilename (node:internal/modules/cjs/loader:1420:15)
    at defaultResolveImpl (node:internal/modules/cjs/loader:1058:19)
    at resolveForCJSWithHooks (node:internal/modules/cjs/loader:1063:22)
    at Module._load (node:internal/modules/cjs/loader:1226:37)
    at TracingChannel.traceSync (node:diagnostics_channel:328:14)
    at wrapModuleLoad (node:internal/modules/cjs/loader:245:24)
    at Module.require (node:internal/modules/cjs/loader:1503:12)
    at require (node:internal/modules/helpers:152:16)
    at Object.<anonymous> (C:\Users\ESTUDIANTES\sfi1JP\sfi1-2026-20-u4-CaseStudy\bridgeserver.js:16:29)
    at Module._compile (node:internal/modules/cjs/loader:1760:14) {
  code: 'MODULE_NOT_FOUND',
  requireStack: [
    'C:\\Users\\ESTUDIANTES\\sfi1JP\\sfi1-2026-20-u4-CaseStudy\\bridgeserver.js'
  ]
}

Node.js v25.3.0

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP/sfi1-2026-20-u4-CaseStudy (main)
$ ls
adapters/  bridgeClient.js  bridgeServer.js  fsm.js  index.html  jsconfig.json  package-lock.json  package.json  sketch.js  style.css

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP/sfi1-2026-20-u4-CaseStudy (main)
$ npm install //instala lo necesario para crear el server

added 22 packages, and audited 23 packages in 939ms

14 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities
npm notice
npm notice New minor version of npm available! 11.6.2 -> 11.11.0
npm notice Changelog: https://github.com/npm/cli/releases/tag/v11.11.0
npm notice To update run: npm install -g npm@11.11.0
npm notice

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP/sfi1-2026-20-u4-CaseStudy (main)
$ ls
adapters/  bridgeClient.js  bridgeServer.js  fsm.js  index.html  jsconfig.json  node_modules/  package-lock.json  package.json  sketch.js  style.css

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP/sfi1-2026-20-u4-CaseStudy (main)
$ node bridge //crea el bridge
bridgeClient.js  bridgeServer.js

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP/sfi1-2026-20-u4-CaseStudy (main)
$ node bridgeServer.js //crea el servidor
[2026-03-04T20:16:39.720Z] [INFO] WS listening on ws://127.0.0.1:8081 device=sim
[2026-03-04T20:16:39.721Z] [INFO] [ADAPTER] Device Connected: sim connected

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP/sfi1-2026-20-u4-CaseStudy (main)
$ code . //abro el directorio de trabajo

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP/sfi1-2026-20-u4-CaseStudy (main)
$ node bridgeServer.js
[2026-03-04T20:33:40.836Z] [INFO] WS listening on ws://127.0.0.1:8081 device=sim
[2026-03-04T20:33:40.837Z] [INFO] [ADAPTER] Device Connected: sim connected
[2026-03-04T20:33:45.499Z] [INFO] [NETWORK] Remote Client connected from ::ffff:127.0.0.1. Total clients: 1
[2026-03-04T20:33:45.504Z] [INFO] [NETWORK] Client requested adapter connect
[2026-03-04T20:33:45.505Z] [INFO] [HW-POLICY] Adapter already open. Sending current status to incoming client.

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1JP/sfi1-2026-20-u4-CaseStudy (main)
$ node bridgeServer.js --device microbit
[2026-03-04T20:39:09.948Z] [INFO] WS listening on ws://127.0.0.1:8081 device=microbit
[2026-03-04T20:39:09.960Z] [INFO] micro:bit found at COM9
[2026-03-04T20:39:16.568Z] [INFO] [NETWORK] Remote Client connected from ::ffff:127.0.0.1. Total clients: 1
[2026-03-04T20:39:16.575Z] [INFO] [NETWORK] Client requested adapter connect
[2026-03-04T20:39:16.579Z] [INFO] [ADAPTER] Device Connected: serial open COM9 @115200


```

Mísma secuencia pero más limpia
``` Bash
ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~
$ mkdir sfi1v2JP

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~
$ cd sfi1v2JP

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1v2JP
$ git clone https://github.com/juanferfranco/sfi1-2026-20-u4-CaseStudy.git
Cloning into 'sfi1-2026-20-u4-CaseStudy'...
remote: Enumerating objects: 17, done.
remote: Counting objects: 100% (17/17), done.
remote: Compressing objects: 100% (15/15), done.
remote: Total 17 (delta 0), reused 17 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (17/17), 10.65 KiB | 2.66 MiB/s, done.

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1v2JP
$ ls
sfi1-2026-20-u4-CaseStudy/

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1v2JP
$ cd sfi1-2026-20-u4-CaseStudy/

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1v2JP/sfi1-2026-20-u4-CaseStudy (main)
$ ls
adapters/  bridgeClient.js  bridgeServer.js  fsm.js  index.html  jsconfig.json  package-lock.json  package.json  sketch.js  style.css

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1v2JP/sfi1-2026-20-u4-CaseStudy (main)
$ code .

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1v2JP/sfi1-2026-20-u4-CaseStudy (main)
$ npm install

added 22 packages, and audited 23 packages in 531ms

14 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities

ESTUDIANTES@DESKTOP-N4RMBH7 MINGW64 ~/sfi1v2JP/sfi1-2026-20-u4-CaseStudy (main)
$ node bridgeServer.js
[2026-03-06T19:47:44.845Z] [INFO] WS listening on ws://127.0.0.1:8081 device=sim
[2026-03-06T19:47:44.846Z] [INFO] [ADAPTER] Device Connected: sim connected
```



## Bitácora de aplicación 

const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class ParseError extends Error { }

function parseCsvLine(line) {
  const values = line.trim().split("|");
  if (values.length !== 6) throw new ParseError(`Expected 6 values, got ${values.length}`);

  const x = Number(values[0]);
  const y = Number(values[1]);
  const btnA = String(values[2]).trim().toLowerCase();
  const btnB = String(values[3]).trim().toLowerCase();
  const chk = Number(values[4]);

  if (!Number.isFinite(x) || !Number.isFinite(y)) throw new ParseError("Invalid numeric data");
  if (x < -2048 || x > 2047 || y < -2048 || y > 2047) throw new ParseError("Out of expected range");
  if (!["true", "false"].includes(btnA) || !["true", "false"].includes(btnB)) throw new ParseError("Invalid button data");

  return { x: x | 0, y: y | 0, btnA: btnA === "true", btnB: btnB === "true" };
}

class MicrobitV2Adapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;
    this.buf = "";
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
    this.buf = "";
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  _onChunk(chunk) {
    this.buf += chunk.toString("utf8");

    let idx;
    while ((idx = this.buf.indexOf("\n")) >= 0) {
      const line = this.buf.slice(0, idx).trim();
      this.buf = this.buf.slice(idx + 1);

      if (!line) continue;

      try {
        const parsed = parseCsvLine(line);
        this.onData?.(parsed);
      } catch (e) {
        if (e instanceof ParseError) {
          if (this.verbose) console.log("Bad data:", e.message, "raw:", line);
        } else {
          this._fail(e);
        }
      }
    }

    if (this.buf.length > 4096) this.buf = "";
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = "";
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

module.exports = MicrobitV2Adapter;


## Bitácora de reflexión



