# Unidad 8

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 
CAMBIOS CÓDIGO
ANTES: Trabajaba principalmente con un solo adapter llamado adapter.
AHORA: Ahora trabaja con una lista adapters.

- createAdapter() ahora devuelve arrays, por ejemplo [microbit, strudel, osc].
  - Se agregaron combinaciones como strudel-osc y micro-strudel-osc.

- En La versión anterior se creaba un oscAdapter aparte dentro de main(), ahora OSC entra como parte de createAdapter().
- El servidor hacía adapter.connect() sobre una sola fuente, ahora recorre todos los adapters con for (const adapter of adapters).
- El servidor reenviaba principalmente micro:bit y algunos casos de Strudel/OSC, ahora cualquier adapter puede enviar datos normalizados al frontend.
- Recibía sólamente eventos de Strudel y OSC, ahora también recibe de Microbit
- Anteriormente las animaciones aparecían en posiciones aleatorias, ahora dependen de la posición del microbit
- El botón A activa oscColorOverride y botón B lo desactiva.
- Se agregó node-osc.

bridgeServer
``` js

//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200

//   WS contract:
//    * bridge To client:
//        {type:"status", state:"ready|connected|disconnected|error", detail:"..."}
//        {type:"microbit", x:int, y:int, btnA:bool, btnB:bool, t:ms}
//    * client To bridge:
//        {cmd:"connect"} | {cmd:"disconnect"}
//        {cmd:"setSimHz", hz:30}
//        {cmd:"setLed", x:2, y:3, value:9}


const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitV2Adapter = require("./adapters/MicrobitV2Adapter");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
const StrudelAdapter = require("./adapters/StrudelAdapter");
const STRUDEL_PORT = parseInt(getArg("strudelPort", "8080"), 10);
const OSCAdapter = require("./adapters/OSCAdapter");
const OSC_PORT = parseInt(getArg("oscPort", "8082"), 10);

const log = {
  info: (...args) => console.log(`[${new Date().toISOString()}] [INFO]`, ...args),
  warn: (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`, ...args),
  error: (...args) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...args)
};


function getArg(name, def = null) {
  const i = process.argv.indexOf(`--${name}`);
  if (i >= 0 && i + 1 < process.argv.length) return process.argv[i + 1];
  return def;
}

function hasFlag(name) {
  return process.argv.includes(`--${name}`);
}

function nowMs() { return Date.now(); }

function safeJsonParse(s) {
  try {
    return JSON.parse(s);

  } catch (e) {
    log.warn("Failed to parse JSON: ", s, e);
    return null;
  }
}

function broadcast(wss, obj) {
  const text = JSON.stringify(obj);
  for (const client of wss.clients) {
    if (client.readyState === 1) client.send(text);
  }
}

function status(wss, state, detail = "") {
  broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const DEVICE = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT = parseInt(getArg("wsPort", "8081"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD = parseInt(getArg("baud", "115200"), 10);
const SIM_HZ = parseInt(getArg("hz", "30"), 10);
const VERBOSE = hasFlag("verbose");

async function findMicrobitPort() {
  const ports = await SerialPort.list();
  const microbit = ports.find(p =>
    p.vendorId && parseInt(p.vendorId, 16) === 0x0D28
  );
  return microbit?.path ?? null;
}

async function createAdapter() {
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) { log.error("micro:bit not found."); process.exit(1); }
    log.info(`micro:bit found at ${path}`);

    return [
      new MicrobitV2Adapter({ path, baud: BAUD, verbose: VERBOSE })
    ];
  }

  if (DEVICE === "microbit-bin") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) { log.error("micro:bit not found."); process.exit(1); }

    return [
      new MicrobitBinaryAdapter({ path, baud: BAUD, verbose: VERBOSE })
    ];
  }

  if (DEVICE === "strudel") {
    return [
      new StrudelAdapter({ port: STRUDEL_PORT, verbose: VERBOSE })
    ];
  }

  if (DEVICE === "strudel-osc") {
    const oscPort = parseInt(getArg("oscPort", "9000"), 10);

    return [
      new StrudelAdapter({ port: STRUDEL_PORT, verbose: VERBOSE }),
      new OSCAdapter({ port: oscPort, verbose: VERBOSE })
    ];
  }

  if (DEVICE === "micro-strudel-osc") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) { log.error("micro:bit not found."); process.exit(1); }
    log.info(`micro:bit found at ${path}`);

    const oscPort = parseInt(getArg("oscPort", "9000"), 10);

    return [
      new MicrobitV2Adapter({ path, baud: BAUD, verbose: VERBOSE }),
      new StrudelAdapter({ port: STRUDEL_PORT, verbose: VERBOSE }),
      new OSCAdapter({ port: oscPort, verbose: VERBOSE })
    ];
  }

  return [
    new SimAdapter({ hz: SIM_HZ })
  ];
}

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapters = await createAdapter();

  for (const adapter of adapters) {
    adapter.onConnected = (detail) => {
      log.info(`[ADAPTER] Connected: ${detail}`);
      status(wss, "connected", detail);
    };

    adapter.onDisconnected = (detail) => {
      log.warn(`[ADAPTER] Disconnected: ${detail}`);
      status(wss, "disconnected", detail);
    };

    adapter.onError = (detail) => {
      log.error(`[ADAPTER] Error: ${detail}`);
      status(wss, "error", detail);
    };

    adapter.onData = (d) => {
      if (d.type === "strudel" || d.type === "osc") {
        broadcast(wss, d);
        return;
      }

      broadcast(wss, {
        type: "microbit",
        x: d.x,
        y: d.y,
        btnA: !!d.btnA,
        btnB: !!d.btnB,
        t: nowMs()
      });
    };
  }

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);

    const anyConnected = adapters.some(a => a.connected);
    const state = anyConnected ? "connected" : "ready";

    const detail = anyConnected
      ? adapters.map(a => a.getConnectionDetail?.() ?? "adapter").join(" + ")
      : `bridge (${DEVICE})`;

    ws.send(JSON.stringify({
      type: "status",
      state,
      detail,
      t: nowMs()
    }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info(`[NETWORK] Client requested adapters connect`);

        try {
          for (const adapter of adapters) {
            if (!adapter.connected) {
              await adapter.connect();
            }
          }
        } catch (e) {
          const detail = `connect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ${detail}`);
          status(wss, "error", detail);
        }

        return;
      }

      if (msg.cmd === "disconnect") {
        log.info(`[NETWORK] Client requested adapters disconnect`);

        try {
          for (const adapter of adapters) {
            if (adapter.connected) {
              await adapter.disconnect();
            }
          }

          status(wss, "disconnected", "all adapters disconnected");
        } catch (e) {
          const detail = `disconnect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ${detail}`);
          status(wss, "error", detail);
        }

        return;
      }

      if (msg.cmd === "setSimHz") {
        const simAdapter = adapters.find(a => a instanceof SimAdapter);

        if (simAdapter) {
          await simAdapter.handleCommand(msg);
          status(wss, "connected", `sim hz=${simAdapter.hz}`);
        }

        return;
      }

      if (msg.cmd === "setLed") {
        try {
          for (const adapter of adapters) {
            await adapter.handleCommand?.(msg);
          }
        } catch (e) {
          const detail = `command failed: ${e.message || e}`;
          log.error(`[ADAPTER] ${detail}`);
          status(wss, "error", detail);
        }

        return;
      }
    });

    ws.on("close", () => {
      log.info(`[NETWORK] Remote Client disconnected. Total clients left: ${wss.clients.size}`);

      if (wss.clients.size === 0) {
        for (const adapter of adapters) {
          adapter.disconnect();
        }
      }
    });
  });

  if (DEVICE === "sim") {
    for (const adapter of adapters) {
      await adapter.connect();
    }
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});
```

bridgeClient
``` js
class BridgeClient {
  constructor(url = "ws://127.0.0.1:8081") {
    this._url = url;
    this._ws = null;
    this._isOpen = false;

    this._onData = null;
    this._onConnect = null;
    this._onDisconnect = null;
    this._onStatus = null;
  }

  get isOpen() {
    return this._isOpen;
  }

  onData(callback) { this._onData = callback; }
  onConnect(callback) { this._onConnect = callback; }
  onDisconnect(callback) { this._onDisconnect = callback; }
  onStatus(callback) { this._onStatus = callback; }


  open() {
    if (this._ws && this._ws.readyState === WebSocket.OPEN) {
      if (!this._isOpen) this.send({ cmd: "connect" });
      return;
    }

    if (this._ws) {
      this.close();
    }

    this._ws = new WebSocket(this._url);

    this._ws.onopen = () => {
      this.send({ cmd: "connect" });
    };

    this._ws.onmessage = (event) => {
      let msg;
      try {
        msg = JSON.parse(event.data);
      } catch (e) {
        console.warn("WS message is not JSON:", event.data);
        return;
      }


      if (msg.type === "status") {
        this._onStatus?.(msg);

        if (msg.state === "connected") {
          this._isOpen = true;
          this._onConnect?.();
        }

        if (msg.state === "disconnected" || msg.state === "error" || msg.state === "ready") {
          this._isOpen = false; 
          this._onDisconnect?.();
          if (msg.state === "error") {
            this._ws?.close();
            this._ws = null;
          }          
        }
        return;
      }

      if (msg.type === "microbit") {
        this._onData?.(msg);
        return;
      }

      if (msg.type === "strudel") {
        this._onData?.(msg);
        return;
      }
      if (msg.type === "osc") {
        this._onData?.(msg);
        return;
      }
    };

    this._ws.onerror = (err) => {
      console.warn("WS error:", err);
    };

    this._ws.onclose = () => {
      this._handleDisconnect();
    };
  }

  close() {
    if (!this._ws || this._ws.readyState !== WebSocket.OPEN) return;

    try {
      this.send({ cmd: "disconnect" });
      this._isOpen = false;
    } catch (e) {
      console.warn("Failed to send disconnect command:", e);
    }
  }

  send(obj) {
    if (!this._ws || this._ws.readyState !== WebSocket.OPEN) return;
    this._ws.send(JSON.stringify(obj));
  }

  _handleDisconnect() {
    this._isOpen = false;
    this._ws = null;
    this._onDisconnect?.();
  }
}

```

StrudelAdapter
``` js
const { WebSocketServer } = require("ws");
const BaseAdapter = require("./BaseAdapter");

class StrudelAdapter extends BaseAdapter {
  constructor({ port = 8080, verbose = false } = {}) {
    super();
    this.port    = port;
    this.verbose = verbose;
    this._wss    = null;
  }

  async connect() {
    if (this.connected) return;

    this._wss = new WebSocketServer({ port: this.port });

    this._wss.on("listening", () => {
      this.connected = true;
      this.onConnected?.(`strudel ws escuchando en :${this.port}`);
    });

    this._wss.on("connection", (ws) => {
      if (this.verbose) console.log("[StrudelAdapter] Strudel conectado");


      ws.on("message", (raw) => this._onMessage(raw.toString("utf8")));
      ws.on("error",   (err) => this._fail(err));
    });

    this._wss.on("error", (err) => this._fail(err));
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;
    await new Promise((resolve) => this._wss.close(resolve));
    this._wss = null;
    this.onDisconnected?.("strudel ws cerrado");
  }

  getConnectionDetail() {
    return `strudel ws :${this.port}`;
  }

  _onMessage(raw) {
    let parsed;
    try {
      parsed = JSON.parse(raw);
    } catch (e) {
      if (this.verbose) console.log("[StrudelAdapter] JSON inválido:", raw);
      return;
    }

    const normalized = this._normalize(parsed);

    if (!normalized) return;


    this.onData?.(normalized);
  }


  _normalize(msg) {
    const args   = msg.args ?? [];
    const params = {};
    for (let i = 0; i + 1 < args.length; i += 2) {
      params[args[i]] = args[i + 1];
    }

    // Sin campo 's' no hay sonido; descartamos el evento.
    if (!params.s) return null;


    return {
      type:      "strudel",
      timestamp: msg.timestamp ?? Date.now(),
      payload: {
        eventType: "noteEvent",
        s:         params.s,
        bank:      params.bank  ?? null,
        delta:     params.delta ?? 0.5,
        cycle:     params.cycle ?? 0,
        cps:       params.cps   ?? 0.5,
      },
    };
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  async handleCommand(_cmd) {}
}

module.exports = StrudelAdapter;
```

OSCAdapter
``` js
// OSCAdapter.js
// Recibe mensajes OSC por UDP desde Open Stage Control,
// los normaliza al contrato del sistema y los entrega
// a bridgeServer mediante onData.
// No decide nada visual ni musical.

const { Server: OscServer } = require("node-osc");
const BaseAdapter = require("./BaseAdapter");

class OSCAdapter extends BaseAdapter {
  constructor({ port = 8082, verbose = false } = {}) {
    super();
    this.port    = port;
    this.verbose = verbose;
    this._server = null;
  }

  // Abre el servidor UDP/OSC donde Open Stage Control enviará mensajes.
  // Equivalente a abrir el WebSocketServer en StrudelAdapter.
  connect() {

    return new Promise((resolve) => {
      this._server = new OscServer(this.port, "0.0.0.0", () => {
        this.connected = true;
        this.onConnected?.(`OSC escuchando en udp://0.0.0.0:${this.port}`);
        resolve();
      });

      // Cada mensaje OSC que llega pasa por _onMessage.
      /*this._server.on("message", (msg) => {
        // node-osc entrega [address, arg1, arg2, ...]
        this._onMessage(msg);
      });*/

      this._server.on("message", (msg, rinfo) => {
    console.log("[OSC RAW]", msg); // temporal para debug
    this._onMessage(msg);
});

      this._server.on("error", (err) => {
        this.onError?.(err.message);
        resolve(); // no bloquear el arranque del bridge
      });
    });
  }

  disconnect() {
    return new Promise((resolve) => {
      if (!this._server) { resolve(); return; }
      this._server.close(() => {
        this.connected = false;
        this._server = null;
        this.onDisconnected?.("OSC server cerrado");
        resolve();
      });
    });
  }

  getConnectionDetail() {
    return `osc udp://0.0.0.0:${this.port}`;
  }

  handleCommand(_cmd) {}

  // Recibe el array crudo de node-osc y lo normaliza.
  _onMessage(msg) {
    const normalized = this._normalize(msg);
    if (!normalized) return;

    if (this.verbose) console.log("[OSCAdapter]", normalized);
    this.onData?.(normalized);
  }

  // node-osc entrega: ["/address", arg1, arg2, ...]
  // Contrato de salida estable que bridgeServer reenvía sin transformar.
  _normalize(msg) {
    const address = msg[0];
    const args    = msg.slice(1);

    // Descarta mensajes sin address válida.
    if (typeof address !== "string" || !address.startsWith("/")) return null;

    return {
      type: "osc",
      payload: {
        address,
        args,
      },
    };
  }
}

module.exports = OSCAdapter;
```

MicrobitV2Adapter
``` js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class ParseError extends Error { }

function parseCsvLine(line) {
  const trimmed = line.trim();

  // Verificar que empiece con $
  if (!trimmed.startsWith("$")) {
    throw new ParseError("Frame does not start with $");
  }

  // Quitar el $ y separar por |
  const values = trimmed.slice(1).split("|");

  if (values.length !== 6) {
    throw new ParseError(`Expected 6 values, got ${values.length}`);
  }

  const x = Number(values[1].split(":")[1]);
  const y = Number(values[2].split(":")[1]);
  const btnA = Number(values[3].split(":")[1]);
  const btnB = Number(values[4].split(":")[1]);
  const chk = Number(values[5].split(":")[1]);

  const calcChk = Math.abs(x) + Math.abs(y) + btnA + btnB;

  if (calcChk !== chk) {
    throw new ParseError("Checksum no coincide");
  }

  if (!Number.isFinite(x) || !Number.isFinite(y)) {
    throw new ParseError("Invalid numeric data");
  }

  if (x < -2048 || x > 2047 || y < -2048 || y > 2047) {
    throw new ParseError("Out of expected range");
  }

  return {
    x: x | 0,
    y: y | 0,
    btnA: btnA === 1,
    btnB: btnB === 1
  };
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

```

sketch
``` js 
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    KEY_PRESSED: "KEY_PRESSED",
    KEY_RELEASED: "KEY_RELEASED",
};

class PainterTask extends FSMTask {
    constructor() {
        super();

        this.eventQueue = [];
        this.activeAnimations = [];
        this.mode = null;

        // ── estado OSC — NUEVO ─────────────────────────────
        // Variables persistentes que los controles de OSC modifican.
        // No se resetean entre frames: son estado continuo, no eventos.

        // Control 1: color base RGB — /rgb_1 [r, g, b] rango 0-255
        // Sobreescribe el color por defecto de getColorForSound()
        // cuando oscColorOverride está activo.
        this.oscColor = [220, 60, 50];
        this.oscColorOverride = false; // empieza desactivado: cada sonido mantiene su color

        // Control 2: escala de tamaño global — /size [0-1]
        // Multiplica el tamaño de todas las formas.
        this.oscSize = 0.5;

        // Control 3: capa de fondo — /bg_layer [0 o 1]
        // Si está activo, el fondo deja un trail del color OSC
        // en vez del negro neutro.
        this.oscBgLayer = false;

        this.oscSpeed = 1.0; // velocidad normal por defecto

        // ── MICROBIT ───────────────────────

        this.microX = 0;
        this.microY = 0;

        this.microBtnA = false;
        this.microBtnB = false;

        this.transitionTo(this.estado_esperando);
    }

    update() {
        super.update();

        // botones del micro:bit

        if (this.microBtnA) {
            this.oscColorOverride = true;
        }

        if (this.microBtnB) {
            this.oscColorOverride = false;
        }

        this._flushQueue();
    }

    _flushQueue() {
        if (this.mode !== "strudel") return;

        const now = Date.now();
        this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);

        while (this.eventQueue.length > 0) {
            const ev = this.eventQueue[0];

            if (ev.timestamp <= now) {
                const d = ev.payload;

                // NUEVO: si oscColorOverride está activo, usa el color OSC.
                // Si no, mantiene el color original por tipo de sonido.
                const col = this.oscColorOverride
                    ? this.oscColor
                    : getColorForSound(d.s);

                // NUEVO: oscSize escala la duración visual de la animación.
                // Un size mayor hace las formas más grandes y duraderas.
                const scaledDelta = (d.delta * map(this.oscSize, 0, 1, 0.3, 2.0)) / this.oscSpeed;
                this.activeAnimations.push({
                    startTime: ev.timestamp,
                    duration: scaledDelta * 1000,
                    type: d.s,

                    x: map(this.microX, -1024, 1024, width * 0.1, width * 0.9),
                    y: map(this.microY, -1024, 1024, height * 0.1, height * 0.9),

                    color: col,
                    size: this.oscSize, // guardamos el size en la animación
                });

                this.eventQueue.shift();
            } else break;
        }
    }

    estado_esperando = (ev) => {
        if (ev.type === "ENTRY") {
            cursor();
            console.log("Waiting for connection...");
        } else if (ev.type === EVENTS.CONNECT) {
            this.transitionTo(this.estado_corriendo);
        }
    };

    estado_corriendo = (ev) => {
        if (ev.type === "ENTRY") {
            noCursor();
            background(0);
            console.log("Connected. Waiting for data...");
        } else if (ev.type === EVENTS.DISCONNECT) {
            this.mode = null;
            this.transitionTo(this.estado_esperando);
        } else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        } else if (ev.type === EVENTS.KEY_PRESSED) {
            this.handleKeys(ev.keyCode, ev.key);
        } else if (ev.type === EVENTS.KEY_RELEASED) {
            this.handleKeyRelease(ev.keyCode, ev.key);
        } else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(data) {

        // ── STRUDEL ─────────────────────────

        if (data.type === "strudel") {
            this.mode = "strudel";
            this.eventQueue.push(data);
            return;
        }

        // ── OSC ─────────────────────────────

        if (data.type === "osc") {
            this._applyOscControl(data.payload);
            return;
        }

        // ── MICROBIT ────────────────────────

        if (data.type === "microbit") {

            this.microX = data.x;
            this.microY = data.y;

            this.microBtnA = data.btnA;
            this.microBtnB = data.btnB;

            return;
        }
    }

    // NUEVO: traduce cada address OSC a una variable de estado.
    // updateLogic delega aquí para mantener el routing limpio.
    _applyOscControl({ address, args }) {
        if (address === "/color") {
            this.oscColor = [
                Math.round(args[0]),
                Math.round(args[1]),
                Math.round(args[2])
            ];
            this.oscColorOverride = true;
            return;
        }

        if (address === "/size") {
            this.oscSize = constrain(args[0] ?? 0.5, 0, 1);
            return;
        }

        if (address === "/bg_layer") {
            this.oscBgLayer = args[0] === 1;
            return;
        }

        if (address === "/speed") {
            this.oscSpeed = constrain(args[0] ?? 1.0, 0.1, 2.0);
            return;
        }
    }
}

let painter;
let bridge;
let connectBtn;
const renderer = new Map();

function setup() {
    createCanvas(windowWidth, windowHeight);
    background(0);

    painter = new PainterTask();
    bridge = new BridgeClient("ws://127.0.0.1:8081");

    bridge.onConnect(() => {
        connectBtn.html("Disconnect");
        painter.postEvent({ type: EVENTS.CONNECT });
    });

    bridge.onDisconnect(() => {
        connectBtn.html("Connect");
        painter.postEvent({ type: EVENTS.DISCONNECT });
    });

    bridge.onStatus((s) => console.log("BRIDGE STATUS:", s.state, s.detail ?? ""));

    bridge.onData((data) => {

        if (
            data.type === "strudel" ||
            data.type === "osc" ||
            data.type === "microbit"
        ) {

            painter.postEvent({
                type: EVENTS.DATA,
                payload: data
            });
        }
    });

    connectBtn = createButton("Connect");
    connectBtn.position(10, 10);
    connectBtn.mousePressed(() => {
        if (bridge.isOpen) bridge.close();
        else bridge.open();
    });

    renderer.set(painter.estado_corriendo, drawRunning);
}

function draw() {
    painter.update();
    renderer.get(painter.state)?.();
}

function drawRunning() {
    if (painter.mode === "strudel") {

        // NUEVO: si oscBgLayer está activo, el trail usa el color OSC.
        // Si no, usa el negro neutro original.
        if (painter.oscBgLayer) {
            const [r, g, b] = painter.oscColor;
            background(r * 0.1, g * 0.1, b * 0.1, 30); // trail teñido
        } else {
            background(0, 30); // comportamiento original
        }

        const now = Date.now();

        for (let i = painter.activeAnimations.length - 1; i >= 0; i--) {
            const anim = painter.activeAnimations[i];
            const elapsed = now - anim.startTime;
            const p = elapsed / anim.duration;

            if (p <= 1) {
                dibujarElemento(anim, p);
            } else {
                painter.activeAnimations.splice(i, 1);
            }
        }

        return;
    }
}

// ── funciones de dibujo — sin cambios ─────────────────────
// drawRunning no interpreta mensajes: solo pasa anim a estas funciones.
// El color y el size ya fueron resueltos en _flushQueue.

function dibujarElemento(anim, p) {
    push();
    switch (anim.type) {
        case 'tr909bd': dibujarBombo(anim, p, anim.color); break;
        case 'tr909sd': dibujarCaja(anim, p, anim.color); break;
        case 'tr909hh':
        case 'tr909oh': dibujarHat(anim, p, anim.color); break;
        default: dibujarDefault(anim, p, anim.color); break;
    }
    pop();
}

function dibujarBombo(anim, p, c) {
    // NUEVO: el tamaño máximo escala con anim.size (resuelto en _flushQueue)
    const maxD = map(anim.size, 0, 1, 200, 800);
    let d = lerp(100, maxD, p);
    let alpha = lerp(255, 0, p);
    noStroke();
    fill(c[0], c[1], c[2], alpha);
    circle(width / 2, height / 2, d);
}

function dibujarCaja(anim, p, c) {
    // NUEVO: el ancho máximo escala con anim.size
    const maxW = map(anim.size, 0, 1, width * 0.3, width);
    let w = lerp(maxW, 0, p);
    let alpha = lerp(255, 0, p);
    noStroke();
    fill(c[0], c[1], c[2], alpha);
    rectMode(CENTER);
    rect(width / 2, height / 2, w, 50);
}

function dibujarHat(anim, p, c) {
    // NUEVO: el tamaño escala con anim.size
    const maxSz = map(anim.size, 0, 1, 20, 80);
    let sz = lerp(maxSz, 0, p);
    noStroke();
    fill(c[0], c[1], c[2]);
    rectMode(CENTER);
    rect(anim.x, anim.y, sz, sz);
}

function dibujarDefault(anim, p, c) {
    // NUEVO: el tamaño escala con anim.size
    const maxSz = map(anim.size, 0, 1, 50, 200);
    let size = lerp(maxSz, 0, p);
    let angle = p * TWO_PI;

    translate(anim.x, anim.y);
    rotate(angle);
    stroke(c[0], c[1], c[2]);
    noFill();
    rect(0, 0, size, size);
    line(-size, 0, size, 0);
    line(0, -size, 0, size);
}

function getColorForSound(s) {
    const colors = {
        'tr909bd': [255, 0, 80],
        'tr909sd': [0, 200, 255],
        'tr909hh': [255, 255, 0],
        'tr909oh': [255, 150, 0]
    };
    if (colors[s]) return colors[s];
    let charCode = s.charCodeAt(0) || 0;
    return [
        (charCode * 123) % 255,
        (charCode * 456) % 255,
        (charCode * 789) % 255
    ];
}

function windowResized() {
    resizeCanvas(windowWidth, windowHeight);
}
```

Diagrama de flujo
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/afeab4e2-a95a-4ae5-adca-33c1d15d39d7" />


Desgloce de roles

| Fuente                 | Qué controla                                                                                                                                                                         | Cómo entra al sistema                                                                                                                                                                                                                                                                                              | Por qué se usa                                                                                                                                                                                                                 |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **micro:bit**          | Controla la posición de las animaciones visuales mediante los valores de acelerómetro `x` y `y`, y modifica comportamientos persistentes usando los botones A y B.                   | Entra por `MicrobitV2Adapter`, el cual recibe datos seriales desde el micro:bit, los normaliza a un contrato `{type:"microbit", x, y, btnA, btnB}` y los envía a `bridgeServer.js`. Luego los mensajes llegan al frontend mediante `bridgeClient.js` y son procesados por `FSMTask` y `updateLogic`.               | Se utiliza para introducir interacción física y gestual al sistema. Permite que la performance responda al movimiento real del intérprete y agrega control expresivo en tiempo real sin alterar directamente el código visual. |
| **Strudel**            | Controla los eventos temporizados de la obra, activando explosiones visuales, ritmos y animaciones sincronizadas con patrones musicales.                                             | Entra por `StrudelAdapter`, el cual recibe los eventos generados desde Strudel y los normaliza como mensajes `{type:"strudel"}`. Estos eventos son reenviados por `bridgeServer.js` y posteriormente organizados por `FSMTask` para entrar a la cola temporal de `updateLogic`.                                    | Se utiliza porque permite sincronizar el sistema visual con patrones musicales generativos y live coding. Strudel funciona como el motor rítmico y temporal de la obra.                                                        |
| **Open Stage Control** | Controla parámetros persistentes del sistema visual, como color, tamaño, velocidad y capas de fondo. Sus cambios permanecen activos entre frames hasta que se modifiquen nuevamente. | Entra por `OSCAdapter`, el cual recibe mensajes OSC vía UDP desde Open Stage Control. Los datos son normalizados como `{type:"osc", address, args}` y enviados al frontend a través de `bridgeServer.js` y `bridgeClient.js`. Luego `updateLogic` traduce cada dirección OSC a variables persistentes del sistema. | Se utiliza para tener control performático continuo sobre el estado visual del sistema sin depender de eventos temporales. Permite modificar la estética de la obra en vivo mediante sliders, toggles y controles manuales.    |

## Bitácora de reflexión
