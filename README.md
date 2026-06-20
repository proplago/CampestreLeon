# Handoff: Golf Viernes — App de scoring y liquidación

## Overview
"Golf Viernes" es una app móvil para un grupo que juega golf los viernes. Resuelve: convocar
jugadores y armar grupos (Salidas), capturar el score hoyo por hoyo (Captura), liquidar las
apuestas/"rayas" del día con podio + leaderboard (El 19), y celebrar/"carrillar" los resultados
(Destacados). El diseño es un set de 4 pantallas en estética "estilo Masters": blanco limpio,
verde profundo, serif para títulos y nombres, cifras en rojo/verde y acento dorado.

Son **6 pantallas** en total: Resultados (El 19), Captura, Salidas, Destacados, Ventajas y Ajustes.

## About the Design Files
El archivo de este bundle (`Golf Viernes Rediseño.dc.html`) es una **referencia de diseño hecha
en HTML** — un prototipo que muestra el look & feel y el comportamiento buscado, **no** código de
producción para copiar tal cual. La tarea es **recrear estos diseños en el entorno del proyecto
destino** (React Native, Flutter, SwiftUI, etc.) usando sus patrones y librerías establecidos. Si
aún no existe un entorno, elige el framework más apropiado (para una app móvil de este tipo,
React Native o Flutter son buenas opciones) e impleméntalo ahí.

El HTML presenta las 6 pantallas lado a lado dentro de "maquetas" de teléfono. Esas maquetas
(bisel, barra de estado 7:59/📶/batería) son **solo andamiaje de presentación** — en la app real
el sistema operativo dibuja la barra de estado; no la reimplementes.

## Fidelity
**Alta fidelidad (hifi).** Colores, tipografía, espaciado e íconos son finales. Recrea la UI de
forma fiel usando las librerías del codebase. Los íconos están dibujados como SVG de línea inline;
puedes sustituirlos por su equivalente del set de íconos del proyecto (p. ej. Lucide/SF Symbols)
manteniendo el mismo significado.

## Pantallas / Views

### 1. El 19 · Resultados (pantalla principal / tab activa por defecto)
- **Propósito:** ver y cerrar la liquidación del viernes (quién ganó/perdió dinero y "rayas").
- **Nota de copy:** el título de pantalla y el label de la tab dicen **"Resultados"** (antes "Liquidación").
- **Layout (de arriba a abajo):** app bar → título "Resultados" (serif, 34px) → fila de filtros
  (pill de fecha, botón circular de búsqueda, pill "Todos") → tarjeta de **Podio** → tabla
  **Leaderboard** → línea "Cuadra en cero" → botón primario "Cerrar viernes y ajustar ventajas" →
  link dorado "Ver desglose por concepto".
- **Podio:** tarjeta con borde 1px, radius 18px, fondo #fcfcfb. Título "EL PODIO DEL VIERNES"
  (dorado #b98a2e, 11px, uppercase, letter-spacing .14em). 3 columnas alineadas abajo: 2º y 3º con
  avatar circular verde (54px, iniciales en serif), 1º más grande (66px) con anillo dorado
  (#d8b24e, borde 3px + halo box-shadow), corona dorada SVG encima. Base con número de posición
  (1º dorado, 2º/3º gris #eef1ee).
- **Leaderboard (tabla):** header verde #0a6a3a, columnas `POS | JUGADOR | PESOS | RAYAS`
  (grid `38px 1fr 64px 56px`). Subhead gris "HOY · CAMPESTRE LEÓN · RAYA $20". Filas: posición,
  nombre en serif verde (líder con ★ dorado), pesos en bold (verde si +, rojo #c1272d si −, gris
  #9aa39a si 0), rayas. Separador 1px #eef0ed.

### 2. Captura
- **Propósito:** capturar el score por hoyo de cada jugador.
- **Layout:** app bar → título "Captura" → toggle segmentado `Capturar | Tarjeta` → navegador de
  hoyo (flecha ‹ outline, centro "Hoyo 7" + "Par 4 · HCP 11 · 7 de 18", flecha › sólida) → tarjeta
  con filas de jugador → banner "Faltan 3 hoyos" → botón primario "Ir al siguiente pendiente" →
  botón secundario "Ir a cerrar la jugada".
- **Fila de jugador:** nombre serif + estado (✓ verde si capturado / pill roja "⏳ N" si pendiente),
  stepper `− [score] +` (caja con borde, número 24px bold coloreado por resultado), contador de
  tiros con ícono banderín. La fila activa (Beto) expande: botones de resultado
  `Birdie | Par | Bogey | Zopi` (Birdie activo = verde sólido), y chips de extras
  `auto | Sandy | Chevez` (chip "auto" con borde punteado dorado).

### 3. Salidas
- **Propósito:** convocar jugadores y armar los grupos del día.
- **Layout:** app bar → título "Salidas" → bloque "¿Quién juega hoy?" con chips de convocados
  (verde sólido = confirmado; outline gris = no confirmado; ★ = organizador) → tarjeta **Grupos**
  (botón dorado "Sortear", selector de Campo, toggle Hoyos 18/9, listas Grupo 1 verde / Grupo 2
  morado #7a5bb0 con chips ↔ para mover) → bloque **Invitados** con botón outline "+ Agregar invitado".

### 4. Destacados
- **Propósito:** "salón de la fama" del día + la "carrilla" (relajo) del grupo.
- **Layout:** app bar → título "Destacados" → toggle `Día | Jugadores | Temporada` → **hero** verde
  (líder del día, serif blanco 27px, cifras doradas #e3c878, corona de fondo a baja opacidad) →
  **grid de medallas 2×2** (cada tarjeta: borde superior 3px de color, emoji, label uppercase,
  nombre serif, valor) → sección "La mesa del 19" con 3 quotes (fondo #f4f5f2, borde-izq de color).
- **Nota:** en Destacados los **emojis se conservan a propósito** (medallas 🐦🔥🎯💸, corona 👑,
  micrófono 🎙️ y los de las quotes) — son el toque informal de la "carrilla". En el resto de la app
  los íconos son SVG de línea.

### 5. Ventajas
- **Propósito:** administrar las rayas de hándicap que el scratch (menor índice) le da al resto para
  emparejar la apuesta.
- **Layout:** app bar → título "Ventajas" + intro → toggle `Hándicaps | Cara a cara` → tarjeta
  **Hándicap del grupo** (fila por jugador: avatar, nombre serif + índice, stepper `− rayas +`; el
  scratch marcado con ★ dorado y 0 rayas) → tarjeta **Cara a cara · hoy** (filas `Beto → Tavo` con
  pill verde "da N" indicando cuántas rayas reparte en ese match).

### 6. Ajustes
- **Propósito:** configuración del grupo, reglas de la casa y cuenta.
- **Layout:** app bar → título "Ajustes" → tarjeta de **perfil** (avatar + nombre + rol) → sección
  **Grupo** (Miembros, Invitar al grupo, Campo por defecto — filas con ícono + chevron/valor) →
  sección **Reglas de la casa** (Valor de la raya \$20, Hoyos por defecto 18, y switches para Birdie
  automático / Sandy / Chevez) → sección **App** (Notificaciones switch, Apariencia, Ayuda y reglas) →
  botón outline rojo **Cerrar sesión** + versión.
- **Switch (toggle):** pista 42×24 radius 999px; ON = verde #0a6a3a con perilla a la derecha,
  OFF = gris #d6dad4 con perilla a la izquierda; perilla blanca 20px.
- **Fila de lista (patrón reutilizable):** ícono línea gris #6e776f (19px) → label 14.5px/600 #1b211d
  (flex:1) → a la derecha valor #6e776f + chevron, o un switch. Separador 1px #eef0ed entre filas,
  tarjeta contenedora borde 1px #e4e6e1 radius 16px. Encabezado de sección uppercase 11px/800
  letter-spacing .1em color #9aa39a.

## Navegación / Bottom nav
Barra inferior sticky, 6 tabs (grid 6 columnas): **El 19, Destac., Salidas, Captura, Ventajas,
Ajustes**. Ícono SVG de línea + label 9px. Tab activa en verde #0a6a3a (label weight 700); inactivas
en gris #9aa39a (weight 600). Los íconos heredan color vía `currentColor`.
- El 19 = trofeo, Destac. = llama, Salidas = banderín, Captura = lápiz, Ventajas = balanza,
  Ajustes = engranaje.
- Las 6 tabs ya tienen pantalla diseñada.

## App bar (compartida en las 4 pantallas)
Izquierda: ícono de perfil (línea, verde). Centro: emblema + "Golf Viernes" (serif itálica 19px,
verde, `white-space:nowrap`). Derecha: ícono de campana/notificaciones (línea, verde). Borde inferior
1px #e4e6e1.

### Emblema "Golf Viernes"
Marca **original** estilo Masters (sin el mapa de USA, que es marca registrada): un banderín rojo
sobre un *green* dorado con contorno verde. SVG 24×24:
- Green (mound): fill `#e6c34a`, stroke `#0a6a3a` 1.3px.
- Hoyo: elipse `#0a6a3a`.
- Asta: línea `#0a6a3a`. Banderín: triángulo `#c1272d`.

## Interactions & Behavior (a implementar)
- **Captura:** stepper +/− ajusta el score; tocar un resultado (Birdie/Par/…) fija el score
  correspondiente; "Ir al siguiente pendiente" salta al próximo hoyo/jugador sin capturar; el estado
  ✓/⏳ se deriva de si el jugador tiene score en el hoyo.
- **Salidas:** "Sortear" reparte aleatoriamente a los convocados en grupos respetando el tamaño;
  chips ↔ permiten mover un jugador entre grupos; toggle 18/9 y selector de campo afectan el cálculo.
- **El 19:** la liquidación calcula pesos/rayas por jugador; "Cuadra en cero" valida que la suma de
  todas las cuentas sea 0 antes de habilitar "Cerrar viernes".
- **Destacados:** los toggles Día/Jugadores/Temporada cambian el periodo agregado.
- Navegación entre tabs vía bottom nav. Transiciones suaves estándar de la plataforma.

## State Management (orientativo)
- `players`, `attendees` (convocados/confirmados), `groups`, `course`, `holesCount` (18/9).
- `scores[holeId][playerId]` → strokes + flags (auto/sandy/chevez).
- Liquidación derivada: `settlement[playerId]` = pesos + rayas; `balances` deben sumar 0.
- `awards` (medallas) y `quotes` para Destacados, derivados del score del día/temporada.

## Design Tokens
**Colores**
- Verde primario: `#0a6a3a`
- Tinta/negro texto: `#1b211d` · `#0e1410` (bisel)
- Gris texto secundario: `#6e776f` · `#54635a` · `#7c7567`
- Gris inactivo/deshabilitado: `#9aa39a`
- Dorado (acento/1er lugar): `#d8b24e` · `#b98a2e` · `#e3c878` (sobre verde)
- Verde-green emblema: `#e6c34a`
- Rojo (pérdidas / banderín): `#c1272d`
- Morado (Grupo 2): `#7a5bb0`
- Fondos: blanco `#fff`, panel `#fcfcfb`, gris claro `#f4f5f2` / `#eef1ee`, banner ámbar `#fbf4e6`
- Bordes: `#e4e6e1` · `#eef0ed` · `#d6dad4`
- Fondo escritorio (presentación): `#e7e5df`

**Tipografía**
- Títulos y nombres: **Newsreader** (serif), weights 400–700; itálica para el logo y algunos labels.
  Títulos de pantalla 34px/600; hero 27px; nombres 17–19px.
- Texto/UI y cifras: **Hanken Grotesk** (sans), weights 400–800. Cifras importantes 16–24px/800.
- Labels uppercase: 10–11px, weight 700–800, letter-spacing .05–.18em.

**Forma**
- Radios: pills 999px; tarjetas 13–18px; tabla 14px; stepper 11px; bisel teléfono 42px.
- Sombra teléfono: `0 24px 60px rgba(15,40,28,.22)`.
- Borde estándar de tarjeta: 1px `#e4e6e1`.

## Assets
- **Fuentes:** Newsreader + Hanken Grotesk vía Google Fonts.
- **Íconos:** SVG de línea inline (sin dependencia externa). Sustituibles por el set del codebase.
- **Emojis:** solo en la pantalla Destacados (intencional).
- No hay imágenes raster ni logos de terceros. El emblema es SVG original incluido en el HTML.

## Files
- `Golf Viernes Rediseño.dc.html` — las 6 pantallas (Resultados/El 19, Captura, Salidas, Destacados,
  Ventajas, Ajustes) con su app bar y bottom nav. Es un "Design Component" en HTML; ábrelo en el
  navegador para verlo.
