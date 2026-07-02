# Rayas Campestre — contexto del proyecto

App web de **un solo archivo** (`rayas-campestre.html`) para llevar las apuestas ("rayas")
del grupo de golf de los viernes del **Club Campestre de León**. Móvil-first, vanilla JS,
toda la lógica vive en un IIFE: `const App=(()=>{ ... return {...}; })(); App.init();`.

## ⚠️ Regla de oro

La **interfaz se puede rediseñar libremente**, pero el **motor de cálculo de rayas NO se toca**
sin avisar. Si vas a recablear la UI, conserva intactas estas funciones y su comportamiento:
`computeDay`, `_computeDayRaw`, `medalSeg`, `h2h`, `allocate`, `received`, `playOrder`,
`orderIdx`, `isSecond`, `autoUnits`, `validManual`, `unitsOnHole`, `inferOyes`.
Después de CUALQUIER cambio, corre la validación de sintaxis y, si puedes, las pruebas headless
(ver más abajo). El resultado de cada día debe **cuadrar en cero** (lo que unos ganan, otros lo pierden).

---

## Las 6 apuestas (reglas confirmadas)

Cada **raya = $20** (editable en Ajustes). El total de cada jugador = suma de todo lo que gana
menos lo que le ganan. Todo es **pairwise** (todos contra todos) y **neto** (con ventaja por HCP),
salvo donde se indique.

1. **Hoyos** — match play neto. En cada hoyo, el de menor score neto le gana **1 raya** a cada quien
   que pierda ese hoyo. Empate = nadie cobra.
2. **Medal** — stroke play estilo **Nassau (9/9/18)**. Son **3 rayas** por total de golpes netos:
   1 al menor de los **primeros 9**, 1 al menor de los **segundos 9**, 1 al menor de los **18**.
   La del back-nine va **×2 si esa pareja dobló**.
3. **Unidades** ("junk") — 1 raya c/u, neto pairwise.
   - Automáticas: **birdie** 🐦, **eagle** 🦅, **hole-out** 🎯 (meterla sin puttear).
   - Manuales (se prenden a mano, requieren par): **sandy** 🏖️, **aqua** 💧.
4. **Oyes** — closest-to-pin, **solo par-3**, **solo dentro del mismo grupo**. Se guarda un **lugar**
   por jugador en cada par-3 (1° = más cerca). Comparación por lugar: el de **menor número le gana a
   todos los de número mayor** y pierde con los menores (escalera). Solo cuenta si ambos tienen lugar.
5. **Doblar** — el que pierde la 1ª vuelta (al *turn*) le **dobla la 2ª vuelta ×2** a quien le ganó.
6. **Castigos** — **Chevez** 🍺, **Calaca** 💀 (automática con **10+** golpes en un hoyo), **Pomo** 🍾.
   Cada uno puede ser tipo **'trago'** (va a la cuenta del 19) o **'raya'** ($), togglable.

---

## Detalles finos que NO se pueden romper

- **Orden de juego es la única fuente de verdad.** Si salen del **hoyo 10**, el "primer 9" del medal
  es 10–18 y el "segundo 9" es 1–9. Esto lo resuelve `orderIdx(d,h)` y `playOrder(d)`. Nunca uses
  índices crudos `0..8 / 9..17` para los segmentos del medal: usa el orden de juego.
- **9 hoyos**: `d.holes === 9`. El medal se reduce a **1 segmento** (el total), **doblar se oculta**,
  y la tarjeta/comparativa muestran solo **TOTAL** (sin OUT/IN).
- **OUT / IN / TOTAL**: en 18 hoyos, comparativa y tarjeta muestran subtotal de los primeros 9 (OUT),
  segundos 9 (IN) y los 18 (TOTAL), siempre en orden de juego.
- **Oyes entre grupos distintos NO se comparan** (no juegan el mismo par-3 a la vez).

---

## Modelo de datos

`state = { rayaValue, courses[], players[], days[], ventajas{}, ventHistory[], dblHistory[],
celebrated{}, sound, currentDayId, units[], castigos[], activeBets{}, customBets[] }`

- **Registro de apuestas** (`CORE_BETS` + helpers `betOn/dayCustom/snapshotBets/cbEvent`): las 6 core se
  prenden/apagan por grupo (`state.activeBets`) y las **apuestas de la casa** (`state.customBets[]`) son
  plantillas `evento-por-hoyo`: `{id,name,emoji,evento,accion:'paga'|'cobra',alcance:'todos'|'salida',rayas,dobla,on}`.
  **Cada día CONGELA su snapshot al crearse** (`d.bets{}` + `d.customBets[]`): reconfigurar apuestas NUNCA
  reescribe días ya creados/cerrados. Días sin snapshot (viejos) = las 6 activas. El gateo de doblar vive
  DENTRO de `isDoubled()` (un solo punto). UI: Ajustes → "Apuestas del grupo" (solo admin/súper).

- **courses[]**: `{ id, name, holes:[{n,par,si}] }`. Hay dos: `'campestre'` y `'molino'` (ambos par 72).
- **players[]**: `{ id, name, apodo, fav, guest }`.
- **day**: `{ id, date, label, courseId, playing[], salidas:[[],[]], start(1|10), holes(9|18),
  doubles{}, closed, scores:{ pid:{ gross[18], putts[18], oyes{}, units{}, cast{} } } }`.
- `migrate()` asegura defaults: `holes=18`, `courseId='campestre'`, y que existan ambos campos.

---

## Mapa de funciones núcleo

**Cálculo del día**
- `_computeDayRaw(d)` / `computeDay(d)` — motor principal (memoizado por `d.id`). Reparte las 6 apuestas.
- `medalSeg(d,a,b,allocA,allocB)` — devuelve `{first, second, total}` con `{winner, mult}` por segmento
  (first = orderIdx<9, second = orderIdx≥9 con ×2 si dobló, total = 18). 1 raya por segmento.
- `h2h(d,a,b)` — neto cabeza a cabeza entre dos (incluye doblar); `firstNet` para elegibilidad de doblar.
- `allocate(total,d)` — reparte los golpes de ventaja por stroke index, restringido al orden de juego.
- `received(a,b)` — golpes que recibe `a` de `b` (de `state.ventajas`).
- `dayStats(d)` / `_dayStatsRaw(d)` — stats por día (memoizado).
- `seasonStats()` / `careerStats()` — acumulados.

**Orden de juego / campo**
- `courseOf(d)` / `holesOf(d)` — campo del día y sus 18 hoyos.
- `playOrder(d)` / `holesOrder(d)` — índices en orden de juego (9 o 18 según `d.holes`/`start`).
- `orderIdx(d,h)` — posición 0..17 de un hoyo en el orden de juego. `isSecond(d,h)` = back nine.
- `isNine(d)`, `holesLabel(d)`, `sameGroup(d,a,b)`, `isDoubled(d,a,b)`.

**Unidades / oyes / castigos**
- `autoUnits(gross,parIdx,putts,holes)` — birdie/eagle/holeout automáticos.
- `validManual(sc,h,holes)` — qué unidades manuales aplican (reqPar).
- `unitsOnHole(d,pid,h)` — neto de unidades en un hoyo.
- `inferOyes(d,h,grp)` — infiere/asigna lugares de oyes en un par-3.
- `autoCast(gross)` — castigo automático (Calaca con 10+).

**Persistencia**
- `detectBackend()`, `buildShards(st)`, `rebuildFromShards(sh)`, `applyRemoteTree(root)`, `save()`,
  `clearStatCache()`.

---

## Persistencia (importante)

Cadena de respaldo: **Firebase (sync granular) → `window.storage` → `localStorage` → memoria**.

- Bloque `FIREBASE_CONFIG` arriba del IIFE: si está **vacío**, usa localStorage (modo solo-este-dispositivo).
  Para sync multi-teléfono, llénalo con la config de tu proyecto Firebase (Realtime Database).
- Modelo de **shards**: cada jugador-tarjeta es su propia ranura (`days/{id}/scores/{pid}`), para que
  dos teléfonos capturando jugadores distintos **no se pisen**. `save()` hace diff y `update()` granular.
- Reglas RTDB recomendadas: `{"rules":{"rayas":{".read":true,".write":true}}}`.
- Indicador de estado en Ajustes: ☁️ Sincronizado / ⚠️ sin conexión / 📱 Solo este dispositivo.

---

## Cómo probar (hazlo siempre antes de entregar)

**Sintaxis** (rápido, obligatorio):
```bash
awk '/<script>/{f=1;next}/<\/script>/{f=0}f' rayas-campestre.html > /tmp/app.js && node --check /tmp/app.js && echo OK
```

**Fixtures del motor** (OBLIGATORIO si tocas cualquier función de cálculo): abre **`tests.html`**
servido junto a `index.html` (servidor local o https://proplago.github.io/CampestreLeon/tests.html).
Carga el motor REAL vía `window.__TEST__=true` + `App.__test` (sin arrancar la UI) y corre ~74 checks
con días sintéticos: las 6 apuestas, ventajas por SI, salida del 10, 9 hoyos, doblar, oyes, castigos
raya/trago, rayaPairs, seasonStats/careerStats y fuzz con invariantes (cuadre en cero, PM antisimétrica,
PM≡h2h.net). **Todos deben pasar antes de desplegar.** Resultados legibles por máquina en `window.__RESULTS__`.

**Headless (Playwright/Chromium)** — patrón: `addInitScript` con un stub de `window.storage` (objeto
en memoria) y **precargar el estado dentro del initScript**. Por el debounce de `save()` (~350–450 ms),
espera >450 ms antes de leer storage. Verifica que cada día **cuadre en cero**.

---

## Convenciones

- **Un solo archivo .html.** CSS, JS y markup juntos. No dependencias de build.
- Lógica en IIFE; el objeto `App` expone los handlers (`App.go`, `App.setCmp`, `App.exportScorecard`, etc.).
- Tipografías: Anton (títulos) + Inter (cuerpo). Tema verde "pizarrón".
- Para deploy estático: GitHub Pages (renombrar a `index.html`) o Cloudflare Pages.
