# Rayas Campestre — Arquitectura V2 (multi-liga, auth y roles)

> Estado: **propuesta Fase 1** (pendiente de validar reglas antes de migrar datos).
> No cambia el motor de cálculo de rayas (`computeDay`, `medalSeg`, `h2h`, …).
> Solo cambia **dónde** viven los datos y **quién** puede escribirlos.

## Conceptos (no confundir)

- **Liga / grupo de juego** (`league`) = la comunidad (p.ej. "los ~20 amigos del viernes
  en Campestre León"). Es la **unidad multi-tenant**. Un usuario puede pertenecer a **varias**
  ligas, con un **rol por liga**.
- **Salida (flight)** = el grupo *por salida del día* (`d.salidas`). NO es multi-tenant.
  Caps: **sáb/dom máx 5**, **entre semana máx 6**; **1–3 salidas** por día (regla global).

## Roles

| Rol | Quién | Puede |
|-----|-------|-------|
| **super** | Tú | Todo. Crear ligas, definir reglas custom por liga, gestionar usuarios. |
| **admin** | Uno por liga | Ventajas y opciones de *su* liga (jugadores, campos, unidades, castigos), aprobar solicitudes de ventaja, cerrar días. |
| **regular** | Golfistas | Ver todo de su liga y **capturar tarjetas de cualquiera**; editar **sus dobladas**; fijar su **ventaja 1ª vez** (luego requiere aprobación del admin). |

## Modelo de datos (Realtime Database)

```
/users/{uid}
  email: "..."
  super: true            // solo tu cuenta
  leagues:
    {leagueId}: { role: "admin" | "regular", playerId: "<pid en esa liga>" }

/leagues/{leagueId}
  meta:   { name, createdBy, createdAt }
  rules:                       // reglas custom por liga (UI de edición en Fase 2)
    rayaValue: 20
    activeBets: { hoyos, medal, unidades, oyes, doblar, castigos }   // booleans
  players:  { {pid}: { name, apodo, fav, guest } }
  courses:  { {courseId}: { name, holes:[{n,par,si}] } }
  units:    [ { id, name, auto, rayas, emoji, reqPar } ]
  castigos: [ { id, name, emoji, auto, tipo } ]

  ventajas:                    // ANIDADO (antes era "giver>receiver" plano) para que las reglas
    {giverPid}:                //  puedan comparar pid contra la ruta sin partir strings
      {receiverPid}: { strokes: <num>, locked: <bool>, setBy: <uid> }
  ventReq:                     // solicitudes de cambio de ventaja (lockeada)
    {reqId}: { giver, receiver, from, to, requestedBy, status: "pending|approved|rejected", ts }

  ventHistory: [ ... ]         // log sliding
  dblHistory:  [ ... ]

  days:
    {dayId}:
      setup:  { date, label, courseId, salidas:[[],[]…], start(1|10), holes(9|18) }
      closed: <bool>           // cerrar = admin/super (ajusta ventajas sliding)
      scores: { {pid}: { gross[18], putts[18], oyes{}, units{}, cast{} } }
      doubles:                 // DIRECCIONAL (antes pairKey simétrico): doubler→target
        {doublerPid}: { {targetPid}: true }
```

### Por qué cambian dos estructuras

- **ventajas anidadas** `{{giver}}/{{receiver}}`: permite que las Security Rules comparen
  `myPid === $giver || myPid === $receiver` directamente sobre la ruta (RTDB no puede
  partir la llave `"a>b"`). En memoria se sigue usando `vkey()`; solo cambia el shard.
- **doubles direccional** `{{doubler}}/{{target}}`: el derecho a doblar es del **perdedor de
  la 1ª vuelta** (el que dobla). Guardar la dirección permite que la regla autorice "solo el
  doubler escribe lo suyo". El **cálculo** sigue tratando la pareja como doblada (simétrico).

### Workflow de ventaja (confirmado)

1. Regular fija su ventaja la **1ª vez** → se guarda con `locked: true`.
2. Cualquier cambio posterior por el regular lo **rechaza la regla**; en su lugar crea un
   `ventReq` (pendiente).
3. El **admin** aprueba (escribe la ventaja, que es admin) o rechaza. **Dobladas**: siempre
   libres para el regular en su propia dirección.

## Migración

- La BD nueva (`campestreleon`) está **vacía**. La data histórica está en la BD vieja
  (`golf-viernes`). Antes de migrar: **respaldo** (`backups/`) y decisión de
  *importar histórico* vs *empezar limpio* (pendiente de confirmar contigo).
- El grupo actual se migra a una **liga por defecto**; tu cuenta se marca `super`.

## Notas de seguridad

- Las reglas **rechazan en servidor**; esconder botones en la UI es solo cosmético.
- `scores` es escribible por **cualquier miembro** de la liga (decisión tuya: captura dinámica).
  Trade-off aceptado: un miembro podría sobrescribir tarjetas ajenas.
- `closed`, `players`, `courses`, `units`, `castigos`, `rules` → admin/super.
