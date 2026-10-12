# SENDAWHEREP2P — Custodia del Proyecto

> Última actualización: 2026-10-12 · Versión del documento: v1

## Mapa del documento

Lectura por defecto: `7. MEMORIA` y este mapa. Las secciones `1`–`5` se abren **solo** cuando la tarea las mencione (`4. CAPACIDADES` y `5. LECCIONES` son de consulta).

- p2p, peerjs, peer id, codigo de 6 digitos, conectar → `4. CAPACIDADES`
- varios archivos, multi-archivo, en orden, carpeta → `4. CAPACIDADES`
- gzip, streaming, compresion, envio sin conexion previa → `4. CAPACIDADES`
- fetch, sin backend, build, repo, deploy → `2. ARQUITECTURA`
- que es el proyecto, stack, rutas, repo, URL de Pages → `1. IDENTIDAD`
- por que algo se hizo asi, gotchas, errores del pasado → `5. LECCIONES`
- como validar, desplegar, ver logs → `6. SKILLS`
- estado actual, decisiones, tarea activa → `7. MEMORIA`
- historial de cambios → `8. VERSIONES` (solo visibles; `8b` nunca)
- entradas antiguas → `5b` / `8b`

## 1. IDENTIDAD

- Qué es el proyecto: clon funcional de Send Anywhere. Transferencia de archivos entre dispositivos mediante código de 6 dígitos, con conexión directa P2P (WebRTC) y sin servidor propio de archivos.
- Stack: HTML/CSS/JavaScript en un único archivo (`index.html`, 933 líneas). Sin build, sin framework, sin gestor de dependencias. Bibliotecas externas por CDN: PeerJS 1.5.4 (señalización WebRTC) y qrcodejs 1.0.0 (código QR).
- Rutas principales: `index.html` (todo el proyecto). `SENDAWHEREP2P-Custodia.md` (este documento).
- Repo git y estado: `https://github.com/SandovalJon/send-anywhere-p2p.git`, rama `master`. `git status` limpio en el momento del arranque. Últimos commits: `6bf6dd1`, `fa90259`, `4c9ec41`.
- URLs: producción `https://sandovaljon.github.io/send-anywhere-p2p/` (HTTP 200, 33.019 bytes servidos, idéntico al tamaño del archivo local).
- Versión del código: `6bf6dd1` ("Comprimir solo formatos de texto (gzip en flujo) con fallback seguro").
- Fecha de la última versión del código: 2026-10-12.

## 2. ARQUITECTURA

- Componentes: un solo documento HTML autocontenido. Incluye CSS, lógica de juego, definición de los datos del wallpaper/archivo y las 6 imágenes SVG embebidas. Dos scripts externos por CDN (PeerJS, qrcodejs) para señalización y QR.
- Cómo se conectan: no hay backend. PeerJS usa su broker público (0.peerjs.com) **solo** para el handshake inicial; después los datos viajan directo entre dispositivos por WebRTC DataChannel. La configuración ICE se fija en `iceConfig()` (línea 379 en el commit `6bf6dd1`): STUN de Google más TURN público de openrelay.metered.ca, y opcionalmente un TURN propio del usuario.
- Entry points: `index.html` servido estáticamente por GitHub Pages.
- Flujos clave:
  - Enviar: selección de archivos → `startSending()` genera un peer ID de 6 dígitos → espera conexión → `peer.on('connection')` → `waitReady()` confirma apertura → `streamTo()` envía.
  - Recepción: `connectReceiver()` → `recvPeer.connect(codigo)` → `handleData()` clasifica mensajes (`meta`, `end`, `done`) yaccumula bloques binarios → `finishCurrent()` reconstruye el Blob.
  - Handshake: el receptor envía `ready` al abrir el canal para desbloquear al emisor sin esperar la cabecera.
- Despliegue (comandos exactos): `git add index.html` · `git commit -m "<mensaje>"` · `git push` · verificar con `gh api repos/SandovalJon/send-anywhere-p2p/pages --jq '.status'` hasta que devuelva `built`.
- Operación (comandos exactos): estado del sitio `gh api repos/SandovalJon/send-anywhere-p2p/pages --jq '{url: .html_url, status: .status}'`. Verificar que el deploy coincide con el código local comparando el tamaño servido con el de `index.html`.

## 3. RECURSOS

- APIs que usa:
  - PeerJS 1.5.4 (`https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js`): señalización y DataChannel.
  - qrcodejs 1.0.0 (`https://cdn.jsdelivr.net/npm/qrcodejs@1.0.0/qrcode.min.js`): QR del enlace de recepción.
  - STUN: `stun.l.google.com:19302`, `stun1.l.google.com:19302`.
  - TURN público de demostración: `openrelay.metered.ca` en puertos 80, 443 y 443/TCP (credenciales públicas de ejemplo incluidas en el código, no son secretas).
  - APIs nativas del navegador: `CompressionStream('gzip')` y `DecompressionStream('gzip')`, con degradación si no existen.
- Dónde vive la configuración: el servidor TURN propio del usuario se guarda **solo en el navegador**, en `localStorage` bajo la clave `sendAnywhereTurn` (`TURN_KEY`). No se sube a ningún sitio. No hay archivos de configuración en el repo; no hay secretos versionados.
- Esquemas de datos: sin base de datos. El protocolo de transferencia usa mensajes de texto JSON por el DataChannel: `{type:'meta', name, size, mime, compressed}`, `{type:'end'}`, `{type:'done'}`, `{type:'ready'}`. Los bloques binarios viajan como `ArrayBuffer`.
- Memoria / almacenamiento: únicamente `localStorage` para la clave `sendAnywhereTurn`. El progreso no se persiste porque no hay juego ni cuentas. Los archivos recibidos viven en memoria como `Blob` con URL de objeto temporal.

## 4. CAPACIDADES

- Transferencia P2P de archivos con código de 6 dígitos — el código es el peer ID de PeerJS. Verificado por: ejecución real entre móvil y PC (conexión ICE establecida).
- Transferencia de múltiples archivos en una sesión — se recorren en orden, cada uno con su `meta`/`end`. Verificado por: `4. CAPACIDADES` completada con 1 archivo en test jsdom; la cadena de varios archivos está implementada pero no verificada en producción.
- Envío sin conexión previa entre dispositivos — compresión gzip en streaming solo para formatos de texto. Verificado por: test con archivo de texto de 48.000 bytes que llegó como 290 bytes y se descomprimió idéntico al original.
- Bloques adaptativos con reintento — `chunkSize` empieza en 64 KB y se reduce a la mitad en cada fallo hasta un mínimo de 16 KB (`MIN_CHUNK`). Verificado por: ejecución en navegador real.
- Control de flujo con ventana deslizante — `PIPE` = 1 MB; se espera solo cuando el búfer se llena, nunca a vacío. Verificado por: test jsdom con red limitada.
- QR de recepción — genera un QR con el enlace `?c=CODIGO`; el receptor abre la pestaña de recepción con el código autocompletado. Verificado por: `urlCode` autocompletó `123456` en test jsdom.
- Diagnóstico ICE en pantalla — líneas `ICE: <estado>` en emisor y receptor mediante `watchIce()`. Verificado por: ejecución real (mostró `ICE: conectado`).
- Configuración de TURN propio — campo opcional en "Configuración avanzada", persistido en `localStorage`. Verificado por: lectura/escritura del campo; el flujo con un TURN real no está verificado.
- Descarga de archivos recibidos — botón por archivo y "Descargar todo". Verificado por: `no verificado` (no probado en producción).

## 5. LECCIONES

1. El callback de `dc.send(data, callback)` de WebRTC puede no dispararse nunca — Causa raíz: en PeerJS la confirmación del buffer no es fiable; la cadena de `await` se quedaba bloqueada en el primer bloque y la transferencia se congelaba para siempre con la cabecera recibida pero 0 bytes. · Fix: no depender del callback; enviar sin callback y controlar el flujo con `dc.bufferedAmount`. · Verificación: test jsdom con par de conexiones simuladas, archivo completado.
2. Vaciar el búfer a cero tras cada bloque serializa la transferencia y la vuelve lentísima — Causa raíz: `while (bufferedAmount > 0)` esperaba el ida y vuelta de red por cada bloque, dejando la red ociosa el 99% del tiempo. · Fix: ventana deslizante de 1 MB (`PIPE`), solo se espera al llenarse. · Verificación: test jsdom con red limitada, 2 MB entregados en 2.57 s.
3. Usar un helper declarado con `const` antes de su declaración rompe todo el script — Causa raíz: `const $ = ...` está en zona muerta temporal; llamarlo antes أحدث `ReferenceError` que abortó el script completo, dejando la página sin selector de archivos. · Fix: declarar el helper antes de su primer uso, o mover el código que lo usa al final. · Verificación: `node --check` no detecta esto; hubo que ejecutar la página en jsdom y comprobar que no hubiera errores.
4. Sin servidores TURN la conexión P2P se queda colgada en silencio — Causa raíz: detrás de NAT estricto (datos móviles, aislamiento AP del router) no hay ruta directa y sin TURN no hay alternativa; PeerJS no reporta error, simplemente nunca se abre el canal. · Fix: añadir TURN a `iceConfig()` más un timeout con diagnóstico visible. · Verificación: ejecución real, `ICE: conectado` tras añadir TURN.
5. El handshake malDiseñado produce un deadlock — Causa raíz: el emisor esperaba `ready` y el receptor solo enviaba `ready` al recibir `meta`, que el emisor aún no había enviado; se desbloqueaba por un timeout de 8 s. · Fix: el receptor envía `ready` al abrir el canal, y el emisor tiene un respaldo de 2.5 s por si el receptor es una versión antigua. · Verificación: código revisado; sin prueba automatizada del caso de versiones cruzadas.
6. Comprimir formatos ya comprimidos no ahorra nada y cuesta CPU — Causa raíz: JPG/PNG ya venir comprimidos por diseño; volver a comprimir es comprimir un ZIP. · Fix: `shouldCompress()` decide por tipo MIME y extensión, excluyendo imagen, vídeo, audio, PDF y ZIP. · Verificación: test con `shouldCompress()`: txt sí, jpg no, mp4 no, csv sin MIME sí.
7. El límite de 16 KB por mensaje de WebRTC se diagnósticó mal — Causa raíz: se atribuyó el fallo a bloques de 64 KB cuando la causa real era el callback de `send()`; el tamaño era una hipótesis no verificada. · Fix: subir a 64 KB con reducción automática a 16 KB ante error. · Verificación: transferencia de foto JPG de 2.2 MB completada en producción.
8. jsdom no es un navegador: `URL.createObjectURL`, `Blob.stream()` y `Storage` como Proxy no se comportan como en un navegador real — Causa raíz: `jsdom` no implementa esas APIs y su `Storage` es un Proxy donde asignar `.setItem` guarda una clave llamada `setItem` en vez de sobrescribir el método. · Fix: stubear `URL.createObjectURL` en `beforeParse` y exponer las funciones del script inyectando código dentro del propio `<script>` original (otro tag de script comparte otro ámbito léxico). · Verificación: los tests pasaron con esos stubs.
9. `node --check` no detecta errores de ejecución, solo de sintaxis — Causa raíz: un `ReferenceError` en tiempo de ejecución pasa la comprobación de sintaxis. · Fix: comprobar sintaxis con `node --check` **y** ejecutar la página en jsdom buscando errores de runtime. · Verificación: el bug del punto 3 llegó a producción pese a pasar `--check`.

## 5b. LECCIONES (archivo)

Sin entradas archivadas todavía.

## 6. SKILLS

- Skill: publicar en GitHub Pages con la CLI de GitHub — Estado: verificada
  - Propósito: subir el cambio y confirmar que el sitio sirve exactamente el código local.
  - Precondiciones: `gh` autenticado con scope `repo`; repo con Pages ya habilitado.
  - Pasos: `git add index.html` · `git commit -m "<mensaje>"` · `git push` · `gh api repos/SandovalJon/send-anywhere-p2p/pages --jq '.status'` en bucle hasta `built` · `Invoke-WebRequest -Uri "https://sandovaljon.github.io/send-anywhere-p2p/" -UseBasicParsing` y comparar `$r.RawContentLength` con el tamaño de `index.html`.
  - Verificación: HTTP 200 y tamaño servido idéntico al local.
  - Agente: [opencode-1]

- Skill: comprobar sintaxis de un HTML autocontenido — Estado: verificada
  - Propósito: validar el JavaScript inline sin abrir un navegador.
  - Pasos: extraer el bloque `<script>` sin `src` con `html.match(/<script>([\s\S]*?)<\/script>/)`, escribirlo a un archivo temporal y ejecutar `node --check`.
  - Verificación: salida limpia, sin salida de error.
  - Agente: [opencode-1]

- Skill: test de transferencia con jsdom — Estado: verificada
  - Propósito: comprobar la lógica de envío y recepción sin dos dispositivos reales.
  - Precondiciones: `npm install --no-save jsdom`.
  - Pasos: cargar el HTML con `runScripts:'dangerously'` · stubear `URL.createObjectURL`, `Peer` y `QRCode` en `beforeParse` · crear un par de conexiones simuladas con `send` que emita `data` en la otra dirección · inyectar código dentro del `<script>` original para exponer `streamTo`, `handleData` y `shouldCompress` (comparten ámbito léxico) · llamar al flujo y esperar a que aparezca `.recv-item.done`.
  - Verificación: `archivos completados: 1` y el nombre del archivo recibido coincide con el enviado.
  - Agente: [opencode-1]

- Skill: comprobar presencia de prompts anteriores en el área de trabajo — Estado: verificada
  - Propósito: decidir si aplica la ruta de actualización del prompt.
  - Pasos: `Get-ChildItem -LiteralPath "C:\SandovalJon\NO BORRAR" -Filter "Prompt_Maestro*.md" -Recurse -Force`.
  - Verificación: solo existen `Prompt_Maestro.md` y `Prompt_Maestro-Operaciones.md`, ambos V1.3.0; no hay nada que borrar.
  - Agente: [opencode-1]

## 7. MEMORIA

- [opencode-1] Tarea activa: ninguna. Bootstrap Ruta A ejecutado el 2026-10-12 (creación del archivo de custodia). Siguiente paso disponible: esperar la siguiente tarea asignada.
- [opencode-1] Decisión: `NOMBRE` del archivo de custodia es `SENDAWHEREP2P`, tomado del repo y no de la carpeta `Paginas` — motivo: la carpeta es genérica y convive con 10 carpetas hermanas; el repo describe el proyecto real — estado: tomada (confirmada por el usuario).
- [opencode-1] Decisión: los wallpapers SVG van embebidos en el propio HTML como `Blob` en tiempo de ejecución — motivo: mantener el requisito de un único archivo y funcionar en GitHub Pages sin assets externos — estado: tomada.
- [opencode-1] Decisión: usar el broker público de PeerJS en lugar de un backend propio — motivo: el usuario pidió un solo archivo HTML para GitHub Pages, incompatible con un servidor de señalización propio — estado: tomada. Consecuencia asumida: si el broker público cae o satura, el servicio no tiene alternativa.
- [opencode-1] Decisión: credenciales TURN propias gestionadas en `localStorage` y no en el repo — motivo: son secretos del usuario y no deben documentarse ni versionarse — estado: tomada.
- [opencode-1] Decisión: tamaño de chunk de 64 KB con reducción adaptativa a 16 KB — motivo: 16 KB era una hipótesis no verificada sobre el límite de WebRTC; la medición real mostró que 64 KB funciona — estado: tomada.
- [opencode-1] Decisión: comprimir solo formatos de texto y excluir imágenes — motivo: comprimir JPG/PNG no ahorra nada — estado: tomada.
- Preferencias del usuario: español; respuestas concisas; quiere que el agente ejecute y publique en GitHub sin pedir Confirmación paso a paso;-reporta el enlace cuando algo queda desplegado. Pide explicación de conceptos ("qué es", "cómo que compresión") antes de aceptarlos.
- Comprometidos: repo `send-anywhere-p2p` con Pages activo en la rama `master`. El repo anterior `wallpaper-quest` existe pero su carpeta local fue vaciada por el usuario; su borrado remoto quedó pendiente por falta del scope `delete_repo`.
- Ideas (no comprometidos): DataChannels en paralelo para dividir el archivo; migrar a WebTransport; servidor TURN propio; borrar el repo `wallpaper-quest`.

## 8. VERSIONES

- v1 | 2026-10-12 | Arranque de la custodia (Ruta A) | código: 6bf6dd1
  - Agente: [opencode-1]
  - Qué cambió: se creó `SENDAWHEREP2P-Custodia.md` con las secciones `1` a `8`, más `5b` y `8b`, a partir de evidencia verificada del repo y del sitio publicado. No se modificó código.
  - Secciones actualizadas: todas (creación). `6. SKILLS` incluye la publicación en Pages, la comprobación de sintaxis, el test de transferencia con jsdom y la detección de prompts anteriores.
  - Verificación: `git status` limpio; `gh api .../pages` devuelve `built` y `https_enforced: true`; el sitio responde HTTP 200 con 33.019 bytes, idénticos al archivo local. Constantes de `index.html` leídas en las líneas 367-379, 516, 551, 637, 660, 847, 955-956.
  - Pendientes: borrar el repo `wallpaper-quest` (requiere `gh auth refresh -s delete_repo`); verificar en producción la descarga de archivos recibidos, la cadena de varios archivos y el flujo con TURN propio.

## 8b. VERSIONES (archivo)

Sin entradas archivadas todavía.