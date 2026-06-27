# Análisis de permisos (RTDB) — por qué "cada cambio rompe" y cómo prevenirlo

> Reglas vigentes: `firebase-rules-v3.json`. Hay que **pegarlas en Firebase → Realtime Database → Reglas**
> cada vez que cambien (no se despliegan con `git push`).

## 1. Cómo funcionan REALMENTE las reglas de RTDB (las 4 trampas)

1. **`.write`/`.read` CASCADEAN hacia abajo.** Si concedes write en un nivel, TODO lo de abajo queda escribible
   y las reglas más profundas se **ignoran**. No se puede "revocar" más abajo.
2. **Una operación en el PADRE necesita permiso EN EL PADRE.** Si el `.write` solo está en el hijo
   (ej. `players/$pid`), entonces escribir/borrar la colección entera (`players`) o el nodo raíz
   (`leagues/{lid}`) se **DENIEGA** — no existe regla que lo conceda ahí. (Este fue el bug de borrar "test".)
3. **`update()` multi-ruta es ATÓMICO: si UNA ruta se deniega, se cae TODO el update.** Por eso un regular
   capturando un score puede perder el save completo si el lote también tocó `meta` o `courses` (solo-admin).
4. **No puedes LEER una colección si el `.read` solo está en sus hijos.** `/leagues` y `/leagueMembers`
   raíz no son legibles (solo `{lid}`), por eso la lista de grupos se arma desde `users/{uid}/leagues`.

## 2. Causa raíz de la rotura recurrente

**Granularidad INCONSISTENTE** entre nodos + **writes atómicos que mezclan rutas de distinto permiso.**
Cada vez que se agrega un write nuevo, hay que recordar a mano que (a) el usuario tenga permiso en ESA ruta
y (b) no se mezcle con rutas restringidas en el mismo `update()`. Si se olvida, se cae entero.

## 3. Mapa de acceso real de la app (código → ruta → quién)

| Operación en código | Ruta | Lectura | Escritura |
|---|---|---|---|
| `save()` → `_fbRef.update()` (diff de `buildShards`) | `leagues/{lid}/*` (granular) | — | cada shard por su regla |
| listener del grupo | `leagues/{lid}` | auth | — |
| listener identidades | `users` | auth | — |
| listener miembros | `leagueMembers/{lid}` | auth | — |
| login (rol) | `users/{uid}/super`, `leagueMembers/{lid}/{uid}/role` | auth | — |
| alta propia | `users/{uid}`, `users/{uid}/leagues/{lid}`, `leagues/{lid}/players/{pid}`, `leagueMembers/{lid}/{uid}` | — | propio / 1er miembro |
| crear grupo | `leagues/{id}/meta/name`, `users/{uid}/leagues/{id}` (+ `leagueMembers`+`players` si admin) | — | súper/admin |
| borrar grupo | cada hijo de `leagues/{id}` (players **por pid**) + `leagueMembers/{id}/*` + `users/*/leagues/{id}` + `photos/{id}` | — | súper |
| hidratar grupos | `users/{uid}/leagues`, `leagues/{lid}/meta/name` | auth/propio | — |
| editar rol | `leagueMembers/{lid}/{uid}` | — | súper/admin |
| editar nombre jugador | `users/{uid}` | — | **propio/súper** ⚠️ |
| fotos | `photos/{lid}/{dayId}/{photoId}` | auth | auth |

## 4. Reglas endurecidas (ya en `firebase-rules-v3.json`) — SUPERSET, no rompe nada

Los cambios solo **AGREGAN** capacidades que faltaban (todo lo que ya funcionaba sigue igual):

1. **`leagues/{lid}/.write = súper`** → el súper puede gestionar/borrar el grupo entero de un golpe
   (antes el nodo raíz no tenía write y por eso no se podía borrar).
2. **`leagues/{lid}/players/.write = súper|admin`** (a nivel COLECCIÓN) manteniendo `players/{pid}` = tu propio uid.
   Así el admin/súper puede borrar o reescribir el roster en lote, y el regular sigue dándose de alta solo.
   (Esto evita el bug del `update()` atómico que se caía por `players`.)
3. **`leagueMembers/.read = súper`** → el súper puede enumerar TODOS los grupos (operador de grupos donde no juega).

## 5. Checklist para CUALQUIER cambio futuro que toque Firebase

Antes de agregar un `set/update/remove`:

- [ ] ¿En qué NIVEL está el `.write` de esa ruta? ¿Coincide con lo que estoy escribiendo (hijo vs colección vs raíz)?
- [ ] Si es `update()` multi-ruta: ¿el usuario actual tiene permiso en **TODAS** las rutas del lote?
      Si no, **pre-filtra por rol** (como hace `save()`: quita `meta/courses/units/castigos/houseRules` y jugadores ajenos para regulares).
- [ ] ¿Voy a borrar/escribir una COLECCIÓN cuyo `.write` está solo en el hijo? → hazlo **hijo por hijo**
      (caso `players/{pid}`), o agrega `.write` a nivel colección en las reglas.
- [ ] ¿Necesito LEER una colección/índice? Verifica que haya `.read` en ese nivel (no solo en `{id}`).
- [ ] Tras login, recuerda el **timing del token**: la 1ª lectura puede denegarse; reintenta con backoff
      y NO escribas hasta bajar datos (`_fbSynced`).

## 6. Bugs latentes detectados (pendientes, no críticos)

- **Admin no-súper editando el nombre de OTRO jugador** falla: `users/{otroUid}` solo lo escribe el dueño o el súper.
  El cambio de rol (`leagueMembers`) sí pasa; el de nombre/correo (`users/{uid}`) se deniega en silencio. (savePlayer)
- **`save()` actualiza `_shardCache=cur` ANTES de confirmar el update**: si Firebase deniega, esos shards quedan
  marcados como "ya escritos" y no se reintentan aunque después haya permiso. Para regulares se pre-filtran, por eso no muerde hoy.
- **`rules` es escribible por cualquier autenticado** y contiene `rayaValue` (dinero): un regular podría cambiarlo
  por API (la UI lo bloquea). Está acoplado con `rayaReq`/`ventSeedConfirm` que SÍ escriben los regulares,
  por eso no se puede simplemente volver solo-admin sin separar el nodo.
