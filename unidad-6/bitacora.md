# Unidad 6

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

Al igual que en las anteriores sesiones, lo primero que hay que hacer es crear un nuevo adaptador para Strudel

StrudelAdapter

``` js
// adapters/StrudelAdapter.js
//
// Responsabilidad ÚNICA: recibir mensajes crudos de Strudel
// vía WebSocket, validar su estructura mínima y entregarlos
// al bridge con tipos garantizados.
//
// Este adapter NO decide:
//   - qué familia sonora es un sonido
//   - qué duración visual corresponde a un delta
//   - cómo se interpreta musicalmente ningún campo
//
// Contrato de salida hacia bridgeServer:
//   {
//     s:         string,   — nombre del sonido tal como viene de Strudel
//     note:      number | null,
//     freq:      number | null,
//     delta:     number,   — duración en la unidad que Strudel usa (ciclos)
//     gain:      number,   — 0 a 1, clampeado
//     timestamp: number,   — ms epoch
//   }

const { WebSocketServer } = require("ws");
const BaseAdapter = require("./BaseAdapter");

class StrudelAdapter extends BaseAdapter {
  constructor({ port = 8080, verbose = false } = {}) {
    super();
    this.port = port;
    this.verbose = verbose;
    this._wss = null;
  }

  getConnectionDetail() {
    return `strudel ws-server :${this.port}`;
  }

  async connect() {
    if (this.connected) return;

    this._wss = new WebSocketServer({ port: this.port });

    this._wss.on("listening", () => {
      this.connected = true;
      this.onConnected?.(`strudel WS server listening on :${this.port}`);
    });

    this._wss.on("connection", (ws) => {
      if (this.verbose) console.log("[StrudelAdapter] Strudel client connected");
      ws.on("message", (raw) => this._handleRaw(raw));
      ws.on("close",   ()    => {
        if (this.verbose) console.log("[StrudelAdapter] Strudel client disconnected");
      });
    });

    this._wss.on("error", (err) => {
      this.onError?.(String(err?.message ?? err));
    });
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;
    await new Promise((resolve) => this._wss.close(resolve));
    this._wss = null;
    this.onDisconnected?.("strudel WS server closed");
  }

  _handleRaw(raw) {
    let msg;
    try {
      msg = JSON.parse(raw.toString("utf8"));
    } catch {
      if (this.verbose) console.warn("[StrudelAdapter] JSON inválido:", raw.toString("utf8").slice(0, 80));
      return;
    }

    const validated = this._validate(msg);
    if (!validated) return;

    this.onData?.(validated);
  }

_validate(msg) {
    const ev = msg?.value ?? msg;
    if (!ev || typeof ev !== "object") return null;

    if (!ev.s) return null;

    return {
      s: ev.s,
      delta: typeof ev.delta === "number" ? ev.delta : 0.25,
      gain: typeof ev.gain === "number"
        ? Math.max(0, Math.min(1, ev.gain))
        : 1,
      timestamp: typeof ev.timestamp === "number"
        ? ev.timestamp
        : Date.now(),
    };
  }
}

module.exports = StrudelAdapter;
``` 

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
const STRUDEL_PORT = parseInt(getArg("strudelPort", "8080"), 10);


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
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`);
    return new MicrobitV2Adapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  if (DEVICE === "microbit-bin") {
     const path = SERIAL_PATH ?? await findMicrobitPort();
     if (!path) {
       log.error("micro:bit not found. Use --serialPort to specify manually.");
       process.exit(1);
     }
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
    if (DEVICE === "strudel") {
      broadcast(wss, {
      type: "strudel",
      timestamp: d.timestamp,
      payload: {
        s: d.s,
        delta: d.delta,
        gain: d.gain,
      },
    });
  } else {
      broadcast(wss, {
        type: "microbit",
        x:    d.x,
        y:    d.y,
        btnA: !!d.btnA,
        btnB: !!d.btnB,
        t:    nowMs(),
      });
    }
  };

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);

    const state = adapter.connected ? "connected" : "ready";

    const detail = adapter.connected
      ? adapter.getConnectionDetail()
      : `bridge (${DEVICE})`;

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
        
        try {
          await adapter.connect();
        } catch (e) {
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
        
        try {
          await adapter.disconnect();
        } catch (e) {
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
        try {
          await adapter.handleCommand?.(msg);
        } catch (e) {
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
  onStrudel(callback) { this._onStrudel = callback; }


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
      // Esperamos JSON normalizado desde el bridge
      let msg;
      try {
        msg = JSON.parse(event.data);
      } catch (e) {
        console.warn("WS message is not JSON:", event.data);
        return;
      }

      // Convención mínima:
      // - {type:"status", state:"...", detail:"..."}
      // - {type:"microbit", x:..., y:..., btnA:..., btnB:...}
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
        // payload ya normalizado
        this._onData?.(msg);
        return;
      }

            if (msg.type === "strudel") {
        this._onStrudel?.(msg);
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
// ─── Constantes de eventos ────────────────────────────────────────────────────
const EVENTS = {
  CONNECT:      "CONNECT",
  DISCONNECT:   "DISCONNECT",
  DATA:         "DATA",
  STRUDEL:      "STRUDEL",
  KEY_PRESSED:  "KEY_PRESSED",
  KEY_RELEASED: "KEY_RELEASED",
};

// ─── Dominio musical: lógica que antes estaba en el adapter ──────────────────

// Familias sonoras: s crudo de Strudel → clave normalizada
const SOUND_FAMILIES = {
  bd: "bd",
  sd: "sd",
  cp: "sd",
  hh: "hh",
};


function resolveFamily(s = "") {
  const key = s.toLowerCase();
  return SOUND_FAMILIES[key] ?? "other";
}


// Conversión de delta (ciclos) a duración visual (ms)
// Decisión visual: 1 ciclo = 4 beats a 120 BPM = 2000 ms
// Factor 0.4: la forma visual dura menos que el evento musical
const BPM          = 120;
const MS_PER_CYCLE = (60 / BPM) * 4 * 1000;

function deltaToMs(delta) {
  return Math.max(80, delta * MS_PER_CYCLE * 0.4);
}

// Mapa de familia sonora → color visual [r, g, b]
const SOUND_COLORS = {
  bd:    [220,  60,  30],  // kick   → rojo cálido
  sd:    [ 30, 160, 220],  // snare  → azul frío
  cp:    [ 30, 160, 220],  // clap   → igual que snare
  hh:    [230, 200,  20],  // hihat  → amarillo
  bass:  [140,  50, 200],  // bass   → violeta
  synth: [ 20, 200, 140],  // synth  → verde-cyan
  other: [180, 180, 180],  // resto  → gris
};

function colorForFamily(family) {
  return SOUND_COLORS[family] ?? SOUND_COLORS.other;
}

// ─── FSMTask ──────────────────────────────────────────────────────────────────
class PainterTask extends FSMTask {
  constructor() {
    super();

    this.c = color(181, 157, 0);
    this.lineSize = 100;
    this.clickPosX = 0;
    this.clickPosY = 0;

    this.rxData = {
      x: 0, y: 0,
      btnA: false, btnB: false,
      prevA: false, prevB: false,
      ready: false,
    };

    // Cola de eventos musicales pendientes de activar
    // Cada entrada: { triggerAt, s, delta, gain, note, freq }
    this.strudelQueue = [];

    // Hits activos que drawRunning lee para dibujar
    this.activeHits = [];

    this.transitionTo(this.estado_esperando);
  }

  // ── Estados ────────────────────────────────────────────────────────────────
  estado_esperando = (ev) => {
    if (ev.type === "ENTRY") {
      cursor();
      console.log("Esperando conexión...");
    } else if (ev.type === EVENTS.CONNECT) {
      this.transitionTo(this.estado_corriendo);
    }
  };

  estado_corriendo = (ev) => {
    if (ev.type === "ENTRY") {
      noCursor();
      strokeWeight(0.75);
      background(255);
      this.rxData = {
        x: 0, y: 0,
        btnA: false, btnB: false,
        prevA: false, prevB: false,
        ready: false,
      };
      this.strudelQueue = [];
      this.activeHits   = [];
    }
    else if (ev.type === EVENTS.DISCONNECT) {
      this.transitionTo(this.estado_esperando);
    }
    else if (ev.type === EVENTS.DATA) {
      this.updateLogic(ev.payload);
    }
    else if (ev.type === EVENTS.STRUDEL) {
      this.updateStrudel(ev.payload);
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

  // ── Lógica del acelerómetro ────────────────────────────────────────────────
  updateLogic(data) {
    this.rxData.ready = true;
    this.rxData.x     = map(data.x, -2048, 2047, 0, width);
    this.rxData.y     = map(data.y, -2048, 2047, 0, height);
    this.rxData.btnA  = data.btnA;
    this.rxData.btnB  = data.btnB;

    if (this.rxData.btnA && !this.prevA) {
      this.lineSize  = random(50, 160);
      this.clickPosX = this.rxData.x;
      this.clickPosY = this.rxData.y;
      console.log("A pressed");
    }

    if (!this.rxData.btnB && this.prevB) {
      this.c = color(random(255), random(255), random(255), random(80, 100));
      console.log("B released");
    }

    this.prevA = this.rxData.btnA;
    this.prevB = this.rxData.btnB;
  }

  // ── Lógica musical: encolar evento y traducir al dominio visual ────────────
  // payload llega crudo desde el adapter: { s, note, freq, delta, gain, timestamp }
  // Aquí vive toda la interpretación musical — no en el adapter ni en drawRunning
updateStrudel(payload) {
    const family = resolveFamily(payload.s);

    this.strudelQueue.push({
        triggerAt: payload.timestamp,
        family,
        delta: payload.delta,
        gain: payload.gain,
    });
    }


  // ── Tick: drenar cola temporal, activar hits ───────────────────────────────
  // Llamado desde draw() antes de drawRunning — separado del renderizado
  tickStrudel() {
    const now = Date.now();

    // Expirar hits viejos
    this.activeHits = this.activeHits.filter(h => h.expiresAt > now);

    // Activar eventos cuyo timestamp ya llegó
    let i = 0;
    while (i < this.strudelQueue.length) {
      const ev = this.strudelQueue[i];

      if (true) {
        const [r, g, b]  = colorForFamily(ev.family);
        const lifetimeMs = deltaToMs(ev.delta);

        this.activeHits.push({
          family:     ev.family,
          x:          random(width  * 0.1, width  * 0.9),
          y:          random(height * 0.1, height * 0.9),
          size:       map(ev.gain, 0, 1, 20, 120),
          r, g, b,
          alpha:      200,
          expiresAt:  now + lifetimeMs,
          lifetimeMs,
          startedAt:  now,
        });

        this.strudelQueue.splice(i, 1);
      } else {
        i++;
      }
    }
  }
}

// ─── Globals ──────────────────────────────────────────────────────────────────
let painter;
let bridge;
let connectBtn;
const renderer = new Map();

// ─── setup ────────────────────────────────────────────────────────────────────
function setup() {
  createCanvas(windowWidth, windowHeight);
  background(255);

  painter = new PainterTask();
  bridge  = new BridgeClient();

  bridge.onConnect(() => {
    connectBtn.html("Desconectar");
    painter.postEvent({ type: EVENTS.CONNECT });
  });

  bridge.onDisconnect(() => {
    connectBtn.html("Conectar");
    painter.postEvent({ type: EVENTS.DISCONNECT });
  });

  bridge.onStatus((s) => {
    console.log("BRIDGE STATUS:", s.state, s.detail ?? "");
  });

  bridge.onData((data) => {
    painter.postEvent({
      type:    EVENTS.DATA,
      payload: { x: data.x, y: data.y, btnA: data.btnA, btnB: data.btnB },
    });
  });

  bridge.onStrudel((msg) => {
    painter.postEvent({
        type: EVENTS.STRUDEL,
        payload: {
        ...msg.payload,
        timestamp: msg.timestamp
        },
    });
  });


  connectBtn = createButton("Conectar");
  connectBtn.position(10, 10);
  connectBtn.style("z-index", "10");
  connectBtn.style("position", "absolute");
  connectBtn.mousePressed(() => {
    if (bridge.isOpen) bridge.close();
    else bridge.open();
  });

  renderer.set(painter.estado_corriendo, drawRunning);
}

// ─── draw ─────────────────────────────────────────────────────────────────────
function draw() {
  background(255);
  painter.update();

  // Tick de la cola temporal: separado del renderizado
  if (painter.state === painter.estado_corriendo) {
    painter.tickStrudel();
  }

  renderer.get(painter.state)?.();
}

// ─── drawRunning: SOLO lee estado ya calculado y dibuja ───────────────────────
// Aquí no se parsea ningún mensaje, no se interpreta ningún tipo de evento
function drawRunning() {
  const mb  = painter.rxData;
  const now = Date.now();

  // ── Visualización del acelerómetro ────────────────────────────────────────
  if (mb.ready && mb.btnA) {
    push();
    translate(width / 2, height / 2);

    let circleResolution = int(map(mb.y, 0, height, 2, 10));
    let radius = mb.x - width / 2;
    let angle  = TAU / circleResolution;

    if (mb.btnB) fill(34, 45, 122, 50);
    else         noFill();

    beginShape();
    for (let i = 0; i <= circleResolution; i++) {
      vertex(cos(angle * i) * radius, sin(angle * i) * radius);
    }
    endShape();
    pop();
  }

  // ── Visualización musical: leer activeHits y dibujar ─────────────────────
  noStroke();
  for (const hit of painter.activeHits) {
    const age      = now - hit.startedAt;
    const progress = age / hit.lifetimeMs;
    const alpha    = lerp(hit.alpha, 0, progress);
    const size     = hit.size * (1 + progress * 0.6);

    fill(hit.r, hit.g, hit.b, alpha);

    if (hit.family === "hh") {
      // Hihat → línea horizontal delgada
      stroke(hit.r, hit.g, hit.b, alpha);
      strokeWeight(2);
      noFill();
      line(hit.x - size * 0.6, hit.y, hit.x + size * 0.6, hit.y);
      noStroke();

    } else if (hit.family === "bd") {
      // Kick → anillo expansivo
      stroke(hit.r, hit.g, hit.b, alpha);
      strokeWeight(3 * (1 - progress));
      noFill();
      circle(hit.x, hit.y, size);
      noStroke();

    } else if (hit.family === "sd" || hit.family === "cp") {
      // Snare / Clap → rectángulo rotado
      push();
      translate(hit.x, hit.y);
      rotate(QUARTER_PI);
      fill(hit.r, hit.g, hit.b, alpha);
      rect(-size * 0.3, -size * 0.3, size * 0.6, size * 0.6);
      pop();

    } else {
      // Bass / synth / other → elipse sólida
      circle(hit.x, hit.y, size);
    }
  }
}

// ─── windowResized ────────────────────────────────────────────────────────────
function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}
``` 
## Bitácora de reflexión
