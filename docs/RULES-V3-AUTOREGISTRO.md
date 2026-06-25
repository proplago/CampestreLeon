# Reglas V3 — Auto-registro seguro (grupo/liga con Google)

> Estado: **propuesta** (pendiente de aplicar). No cambia el motor de cálculo.
> Disparador: bug confirmado el 2026-06-24 — el auto-registro de un jugador normal
> **falla en silencio** porque las reglas no le permiten crear su jugador ni ligar su cuenta,
> y `save()` se traga el error. Síntoma: liga nueva "0 jugadores" + cuenta "pendiente".

Nota de términos: **grupo = liga** (en el código se usa `liga`/`league`; es lo mismo que "grupo").

---

## 1. Objetivo

Que el modelo **"todos se registran con Google como jugador nuevo"** funcione con reglas
**seguras** (no abiertas), cubriendo:

1. **Auto-registro:** un recién llegado, autenticado y con el **código de la liga**, puede
   crear **su propio** jugador y ligar **su propia** cuenta — sin ser admin ni súper.
2. **Correos ligados:** el **admin** ve los correos de los miembros **de su liga** (no global).
3. **Súper operador:** el súper administra ligas donde no juega (ya hecho en código; las reglas
   no lo estorban porque escribe su propio `users/{uid}`).

---

## 2. Por qué hoy NO funciona (causa raíz)

Con `firebase-rules.json` (estricto), el auto-registro choca en **3 escrituras**:

| Paso del auto-registro | Ruta | Regla actual | Resultado |
|---|---|---|---|
| Validar código | leer `leagues/{lid}/meta/joinCode` | solo miembro/súper | el no-miembro no puede leer |
| Crear su jugador | escribir `leagues/{lid}/players` | admin/súper | **denegado** |
| Ligar su cuenta | escribir `users/{uid}/leagues/{lid}` | súper | **denegado** |

Y `save()` ([index.html](../index.html) ~L909) hace `.catch(()=>{ dirty=false })` → el fallo es
**silencioso**: el jugador queda solo en el respaldo local del que se registra y nunca llega a
Firebase. Por eso el súper ve **0 jugadores** y la cuenta **"pendiente"**.

Además, problema de granularidad: `buildShards` escribe **`players` como un solo bloque**
(`out['players'] = {todos}`), no `players/{pid}`. Aunque la regla permitiera "tu propio jugador",
la escritura toca **todo** el nodo `players` → requeriría permiso sobre los jugadores de los demás.

---

## 3. Cambios necesarios (3 capas)

### 3.1 Persistencia — shardear `players` por id  *(habilitante, va primero)*

En `buildShards(st)`:

```js
// ANTES
const players={}; (st.players||[]).forEach(p=>{ if(p&&p.id){ const o=Object.assign({},p); delete o.id; players[p.id]=o; } }); out['players']=players;
// DESPUÉS (granular, como days/{id}/scores/{pid})
(st.players||[]).forEach(p=>{ if(p&&p.id){ const o=Object.assign({},p); delete o.id; out['players/'+p.id]=o; } });
```

En `flattenTree(root)` y el rebuild: leer `root.players` y emitir `players/{pid}` por separado
(ya se hace ese patrón para `ventajas/$g/$r` y `days/$id/scores/$pid`; replicar para players).
Mantener compat de lectura: si llega `players` como bloque viejo, expandirlo.

Borrado de un jugador (`delPlayer`) ya quedaría granular: `out` sin esa clave → `update` pone `null`
en `players/{pid}`. (Ojo: aprovechar para limpiar pids huérfanos — ver hallazgo #2 de la revisión.)

> Riesgo: medio. Tocar la serialización de jugadores. Mitiga: respaldo + probar headless que el día
> cuadra en cero y que el roster se reconstruye igual.

### 3.2 Código de la liga — índice público de solo-lectura

En vez de exponer todo `leagues/{lid}` o dejar el código adentro de `meta` (que pide membresía),
crear un nodo aparte **solo para el código**, legible por cualquier autenticado:

```
joinCodes/{lid} = "ABC123"
```

- Lo escribe el **súper/admin** al generar el código (espejo de `meta.joinCode`).
- `joinRegister` valida leyendo `joinCodes/{ACTIVE_LEAGUE}` (no `leagues/{lid}/meta`).
- Así el recién llegado valida el código **sin** poder leer el resto de la liga.

### 3.3 Membresía propia — `leagues/{lid}/members/{uid}`

Mover la lista que hoy se arma leyendo **todo** `/users` a un nodo **por liga**:

```
leagues/{lid}/members/{uid} = { email, name, playerId, role }
```

- Lo escribe **el propio miembro** al registrarse (y el súper al cambiar rol/quitar).
- Lo leen **los miembros de esa liga** (admin incluido) → "Correos ligados" **scoped a la liga**.
- Beneficio extra: **elimina la confusión** de hoy (cuentas de otras ligas saliendo "pendiente"),
  porque solo se listan miembros de la liga actual.

`users/{uid}/leagues/{lid}` se mantiene (lo necesita `resolveAuthIdentity` para el lookup inverso
uid→ligas). Es duplicación deliberada: uno para "quién soy yo", otro para "quiénes están en la liga".

---

## 4. Reglas nuevas (fragmentos clave)

```jsonc
{
  "rules": {
    ".read": false, ".write": false,

    // Código público (solo lectura) para validar antes de unirse
    "joinCodes": {
      "$lid": {
        ".read": "auth != null",
        ".write": "auth != null && ( root.child('users').child(auth.uid).child('super').val() === true || root.child('users').child(auth.uid).child('leagues').child($lid).child('role').val() === 'admin' )"
      }
    },

    "users": {
      "$uid": {
        ".read":  "auth != null && (auth.uid === $uid || root.child('users').child(auth.uid).child('super').val() === true)",
        ".write": "auth != null && (auth.uid === $uid || root.child('users').child(auth.uid).child('super').val() === true)",
        "super": { ".write": "root.child('users').child(auth.uid).child('super').val() === true" }, // nadie se auto-nombra súper
        "leagues": {
          "$lid": {
            // súper todo; o el dueño SE UNE como regular con código válido (1ª vez, no existía)
            ".write": "auth != null && ( root.child('users').child(auth.uid).child('super').val() === true || ( auth.uid === $uid && newData.child('role').val() === 'regular' && newData.child('code').val() === root.child('joinCodes').child($lid).val() ) )"
          }
        }
      }
    },

    "leagues": {
      "$lid": {
        ".read": "auth != null && ( root.child('users').child(auth.uid).child('super').val() === true || root.child('users').child(auth.uid).child('leagues').child($lid).exists() )",
        ".write": false,

        "meta":  { ".write": "auth != null && root.child('users').child(auth.uid).child('super').val() === true" },

        // jugador: admin/súper cualquiera; un miembro SOLO el suyo (su membership ya fijó playerId)
        "players": {
          "$pid": {
            ".write": "auth != null && ( root.child('users').child(auth.uid).child('super').val() === true || root.child('users').child(auth.uid).child('leagues').child($lid).child('role').val() === 'admin' || root.child('users').child(auth.uid).child('leagues').child($lid).child('playerId').val() === $pid )"
          }
        },

        // miembros por liga: el dueño escribe el suyo; admin/súper cualquiera; lo leen los miembros
        "members": {
          "$uid": {
            ".write": "auth != null && ( auth.uid === $uid || root.child('users').child(auth.uid).child('super').val() === true || root.child('users').child(auth.uid).child('leagues').child($lid).child('role').val() === 'admin' )"
          }
        }

        // ... resto (courses, units, castigos, ventajas, ventReq, days) igual que firebase-rules.json
      }
    }
  }
}
```

Notas:
- `.read` de `leagues/{lid}/members` lo hereda del `.read` de `leagues/{lid}` → **cualquier miembro**
  (incl. admin) puede leer la lista de su liga. ✔ cumple "admin ve correos".
- La validación del código vive **en la regla de escritura** de la membresía (no se confía en el cliente).
- No se exige que el `playerId` sea nuevo en la regla (se puede endurecer luego con
  `!root.child('leagues').child($lid).child('players').child(newData.child('playerId').val()).exists()`).

---

## 5. Cambios en el cliente

1. **`joinRegister`** — reordenar: escribir **primero** la membresía (`users/{uid}/leagues/{lid}`
   con `code`), **esperar** el `.then`, y **luego** crear el jugador (`save()`), para que la regla de
   `players/{pid}` ya vea `playerId===$pid`. Escribir también `leagues/{lid}/members/{uid}` y
   `joinCodes` (este último lo escribe el admin/súper al generar, no el que se une).
2. **Validación del código** — leer de `joinCodes/{ACTIVE_LEAGUE}` en vez de `leagues/{lid}/meta`.
3. **`ensureJoinCode`/generar** — al crear el código, escribir también `joinCodes/{lid}` (espejo).
4. **`save()`** — dejar de tragarse el error: en `.catch`, si es `permission_denied`, marcar estado
   y reflejarlo en el indicador de sync (⚠️) y/o toast **una sola vez** (no spamear). Esto evita
   el "registro fantasma" silencioso.
5. **`cfgAuthUsersHtml` / `fetchAuthUsers`** — leer `leagues/{lid}/members` en vez de `/users`.
   Esto scopea a la liga y mata la confusión de "pendientes" de otras ligas.

---

## 6. Rollout seguro (orden)

1. **Respaldo** de la BD (`backups/`) — exportar Realtime Database.
2. Deploy de los **cambios de cliente** (sharding de players + lecturas nuevas) con reglas **abiertas**
   temporales (`{".read":"auth!=null",".write":"auth!=null"}`) — probar que todo el flujo funciona.
3. Probar auto-registro real con 2 cuentas, verificar día cuadra en cero.
4. Pegar las **reglas V3**. Reprobar auto-registro, captura, cierre, correos ligados (admin), operador.
5. Si algo truena, revertir a reglas abiertas (siguen sirviendo) y depurar.

---

## 7. Decisiones abiertas (confirmar antes de implementar)

- ¿Endurecer "el playerId del auto-registro debe ser NUEVO" en la regla? (recomendado, casa con
  "todos nuevos").
- ¿`members` guarda `email` (correo visible a toda la liga) o solo a admins? (hoy se asume visible a
  miembros; si se quiere solo-admin, hay que separar el correo a otro subnodo).
- ¿El admin puede **cambiar rol** o solo el súper? (hoy en código: solo súper).
