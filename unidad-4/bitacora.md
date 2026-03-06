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



## Bitácora de reflexión


