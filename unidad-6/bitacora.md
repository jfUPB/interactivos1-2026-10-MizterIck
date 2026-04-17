# Unidad 6

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

Al igual que en las anteriores sesiones, lo primero que hay que hacer es crear un nuevo adaptador para Strudel

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

Para el bridgeServer al igual que en las anteriores sesiones se modifica para registrar dentro el nuevo adapter y para que a la hora de ejecutar todo se conecte al dispositivo correcto

``` js
const StrudelAdapter = require("./adapters/StrudelAdapter");

const STRUDEL_PORT = parseInt(getArg("strudelPort", "8080"), 10);
```

Dentro de adapter.onData también se agrega lo siguiente para que en caso de que se utilice el strudel este reenvie el dato ya normalizado

``` js
adapter.onData = (d) => {
    if (d.type === "strudel") {
      broadcast(wss, d); 
      return;
    }
```
bridgeServer

``` js
const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitV2Adapter = require("./adapters/MicrobitV2Adapter");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
const StrudelAdapter = require("./adapters/StrudelAdapter");

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

function hasFlag(name) { return process.argv.includes(`--${name}`); }
function nowMs() { return Date.now(); }

function safeJsonParse(s) {
  try { return JSON.parse(s); } catch (e) {
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
const STRUDEL_PORT = parseInt(getArg("strudelPort", "8080"), 10);

async function findMicrobitPort() {
  const ports = await SerialPort.list();
  const microbit = ports.find(p => p.vendorId && parseInt(p.vendorId, 16) === 0x0D28);
  return microbit?.path ?? null;
}

async function createAdapter() {
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) { log.error("micro:bit not found."); process.exit(1); }
    log.info(`micro:bit found at ${path}`);
    return new MicrobitV2Adapter({ path, baud: BAUD, verbose: VERBOSE });
  }
  if (DEVICE === "microbit-bin") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) { log.error("micro:bit not found."); process.exit(1); }
    return new MicrobitBinaryAdapter({ path, baud: BAUD });
  }
  if (DEVICE === "strudel") {
    return new StrudelAdapter({ port: STRUDEL_PORT, verbose: VERBOSE });
  }
  return new SimAdapter({ hz: SIM_HZ });
}

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapter = await createAdapter();

  adapter.onConnected = (detail) => {
    log.info(`[ADAPTER] Device Connected: ${detail}`);
    status(wss, "connected", detail);
  };

  adapter.onDisconnected = (detail) => {
    log.warn(`[ADAPTER] Device Disconnected: ${detail}`);
    status(wss, "disconnected", detail);
  };

  adapter.onError = (detail) => {
    log.error(`[ADAPTER] Device Error: ${detail}`);
    status(wss, "error", detail);
  };


  adapter.onData = (d) => {
    if (d.type === "strudel") {
      broadcast(wss, d); 
      return;
    }

    broadcast(wss, {
      type: "microbit",
      x: d.x,
      y: d.y,
      btnA: !!d.btnA,
      btnB: !!d.btnB,
      t: nowMs(),
    });
  }

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);
    const state = adapter.connected ? "connected" : "ready";
    const detail = adapter.connected ? adapter.getConnectionDetail() : `bridge (${DEVICE})`;
    ws.send(JSON.stringify({ type: "status", state, detail, t: nowMs() }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info(`[NETWORK] Client requested adapter connect`);
        if (adapter.connected) {
          log.info(`[HW-POLICY] Adapter already open. Sending current status to incoming client.`);
          ws.send(JSON.stringify({ type: "status", state: "connected", detail: adapter.getConnectionDetail(), t: nowMs() }));
          return;
        }
        try { await adapter.connect(); } catch (e) {
          const detail = `connect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "disconnect") {
        log.info(`[NETWORK] Client requested adapter disconnect`);
        if (wss.clients.size > 1) {
          log.info(`[HW-POLICY] Adapater kept open. Shared with ${wss.clients.size - 1} other active client(s).`);
          ws.send(JSON.stringify({ type: "status", state: "disconnected", detail: "logical disconnect only", t: nowMs() }));
          return;
        }
        try { await adapter.disconnect(); } catch (e) {
          const detail = `disconnect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "setSimHz" && adapter instanceof SimAdapter) {
        log.info(`Setting Sim Hz to ${msg.hz}`);
        await adapter.handleCommand(msg);
        status(wss, "connected", `sim hz=${adapter.hz}`);
        return;
      }

      if (msg.cmd === "setLed") {
        try { await adapter.handleCommand?.(msg); } catch (e) {
          const detail = `command failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }
    });

    ws.on("close", () => {
      log.info(`[NETWORK] Remote Client disconnected. Total clients left: ${wss.clients.size}`);
      if (wss.clients.size === 0) {
        log.info("[HW-POLICY] No more remote clients. Auto-disconnecting adapter device to free resources...");
        adapter.disconnect();
      }
    });
  });

  if (DEVICE === "sim" || DEVICE === "strudel") {
    await adapter.connect();
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});
```

Dentro de bridgeClient se agrega únicamente una línea para que los datos del type "strudel" pasen sin problema

``` js
  if (msg.type === "strudel") {
        this._onData?.(msg);
        return;
      }
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

Sketch

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

        this.c = color(181, 157, 0);
        this.lineSize = 100;
        this.angle = 0;
        this.clickPosX = 0;
        this.clickPosY = 0;

        this.rxData = {
            x: 0,
            y: 0,
            btnA: false,
            btnB: false,
            prevA: false,
            prevB: false,
            ready: false
        };

        this.eventQueue = [];
        this.activeAnimations = [];

        this.mode = null; // "microbit" | "strudel"

        this.transitionTo(this.estado_esperando);
    }

    update() {
        super.update();
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

                this.activeAnimations.push({
                    startTime: ev.timestamp,
                    duration: d.delta * 1000,
                    type: d.s,
                    x: random(width * 0.2, width * 0.8),
                    y: random(height * 0.2, height * 0.8),
                    color: getColorForSound(d.s)
                });

                this.eventQueue.shift();

            } else break;
        }
    }

    estado_esperando = (ev) => {
        if (ev.type === "ENTRY") {
            cursor();
            console.log("Waiting for connection...");
        } 
        else if (ev.type === EVENTS.CONNECT) {
            this.transitionTo(this.estado_corriendo);
        }
    };

    estado_corriendo = (ev) => {
        if (ev.type === "ENTRY") {
            noCursor();
            background(0);

            console.log("Connected. Waiting for data...");
        }

        else if (ev.type === EVENTS.DISCONNECT) {
            this.mode = null;
            this.transitionTo(this.estado_esperando);
        }

        else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        }

        else if (ev.type === EVENTS.KEY_PRESSED) {
            this.handleKeys(ev.keyCode, ev.key);
        }

        else if (ev.type === EVENTS.KEY_RELEASED) {
            this.handleKeyRelease(ev.keyCode, ev.key);
        }

        else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(data) {


        if (data.type === "strudel") {
            this.mode = "strudel";
            this.eventQueue.push(data);
            return;
        }

        this.mode = "microbit";

        this.rxData.ready = true;
        this.rxData.x = map(data.x, -2048, 2047, 0, width);
        this.rxData.y = map(data.y, -2048, 2047, 0, height);
        this.rxData.btnA = data.btnA;
        this.rxData.btnB = data.btnB;

        if (this.rxData.btnA && !this.prevA) {
            this.lineSize = random(50, 160);
            this.clickPosX = this.rxData.x;
            this.clickPosY = this.rxData.y;
        }

        if (!this.rxData.btnB && this.prevB) {
            this.c = color(random(255), random(255), random(255), random(80, 100));
        }

        this.prevA = this.rxData.btnA;
        this.prevB = this.rxData.btnB;
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
    bridge = new BridgeClient();

    bridge.onConnect(() => {
        connectBtn.html("Disconnect");
        painter.postEvent({ type: EVENTS.CONNECT });
    });

    bridge.onDisconnect(() => {
        connectBtn.html("Connect");
        painter.postEvent({ type: EVENTS.DISCONNECT });
    });

    bridge.onStatus((s) => {
        console.log("BRIDGE STATUS:", s.state, s.detail ?? "");
    });

    bridge.onData((data) => {

        if (data.type === "strudel") {
            painter.postEvent({ type: EVENTS.DATA, payload: data });
            return;
        }

        painter.postEvent({
            type: EVENTS.DATA,
            payload: {
                x: data.x,
                y: data.y,
                btnA: data.btnA,
                btnB: data.btnB
            }
        });
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

        background(0, 30);

        let now = Date.now();

        for (let i = painter.activeAnimations.length - 1; i >= 0; i--) {
            let anim = painter.activeAnimations[i];

            let elapsed = now - anim.startTime;
            let p = elapsed / anim.duration;

            if (p <= 1) {
                dibujarElemento(anim, p);
            } else {
                painter.activeAnimations.splice(i, 1);
            }
        }

        return;
    }


    else {

        let mb = painter.rxData;

        if (!mb.ready) return;

        if (mb.btnA == 1) {

            push();
            translate(width / 2, height / 2);

            let circleResolution = int(map(mb.y + 100, 0, height, 2, 10));
            let radius = mb.x - width / 2;
            let angle = TAU / circleResolution;

            if (mb.btnB == 1) {
                fill(34, 45, 122, 50);
            } else {
                noFill();
            }

            beginShape();
            for (let i = 0; i <= circleResolution; i++) {
                let x = cos(angle * i) * radius;
                let y = sin(angle * i) * radius;
                vertex(x, y);
            }
            endShape();

            pop();
        }

        return;
    }
}



function dibujarElemento(anim, p) {
    push();

    switch (anim.type) {
        case 'tr909bd':
            dibujarBombo(p, anim.color);
            break;

        case 'tr909sd':
            dibujarCaja(p, anim.color);
            break;

        case 'tr909hh':
        case 'tr909oh':
            dibujarHat(anim, p, anim.color);
            break;

        default:
            dibujarDefault(anim, p, anim.color);
            break;
    }

    pop();
}

function dibujarBombo(p, c) {
    let d = lerp(100, 600, p);
    let alpha = lerp(255, 0, p);
    fill(c[0], c[1], c[2], alpha);
    circle(width / 2, height / 2, d);
}

function dibujarCaja(p, c) {
    let w = lerp(width, 0, p);
    let alpha = lerp(255, 0, p);
    fill(c[0], c[1], c[2], alpha);
    rect(width / 2, height / 2, w, 50);
}

function dibujarHat(anim, p, c) {
    let sz = lerp(40, 0, p);
    fill(c[0], c[1], c[2]);
    rect(anim.x, anim.y, sz, sz);
}

function dibujarDefault(anim, p, c) {
    let size = lerp(100, 0, p);
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

Node BridgeServer en este caso: node bridgeServer.js --device strudel --wsPort 8081 --strudelPort 8080
## Bitácora de reflexión
