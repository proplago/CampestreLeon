# Rollback — procedimiento oficial

> La ventaja estructural de esta app: **los resultados nunca se guardan** — se derivan al vuelo
> de las capturas crudas (gross, putts, oyes, unidades, castigos). Un bug de CÁLCULO muestra
> números mal pero NO corrompe datos: al revertir el código, los números vuelven a ser correctos solos.

## Revertir el código (≈2 minutos, todos los teléfonos incluidos)

```bash
cd "G:\Unidades compartidas\CLAUDE\CampestreLeon"

# 1. Recuperar el index.html de la última versión estable (tag o commit)
git checkout estable-33 -- index.html        # o: git checkout <commit-bueno> -- index.html

# 2. EDITAR APP_VERSION a un valor NUEVO (ej. '2026-07-01-40-rollback')
#    ⚠️ NUNCA reusar un string viejo: checkForUpdate() dispara cuando la versión es DISTINTA,
#    así que un string nuevo garantiza que TODOS los teléfonos se recargan al abrir la app.

# 3. Publicar
git add index.html
git commit -m "rollback a estable-33"
git push origin main
```

En 1–2 min GitHub Pages sirve la versión revertida y cada teléfono se **auto-recarga solo**
al abrir la app (el mismo mecanismo del auto-update funciona en reversa).

## Antes de cada cambio riesgoso (checklist)

- [ ] `git tag estable-NN` en el último commit bueno y `git push origin estable-NN`
- [ ] Descargar **Exportar respaldo (JSON)** en Ajustes → Datos (o snapshot del nodo
      `leagues/campestre-viernes` de Firebase a un archivo)
- [ ] Correr las pruebas del motor (fixtures): todo día sintético debe **cuadrar en cero**
- [ ] Validar sintaxis del `<script>` (node --check o new Function)

## Reglas de esquema que hacen el rollback SIEMPRE seguro

| Regla | Por qué |
|---|---|
| Campos nuevos solo **aditivos** con default en `migrate()` | El código viejo ignora lo que no conoce → tras revertir, la app funciona y los datos nuevos quedan intactos esperando |
| **Nunca** renombrar ni mover shards existentes | El código viejo sigue encontrando todo donde siempre |
| Snapshot de config que afecte resultados **por día** (`day.setup`) | Ni un rollback ni un toggle reescriben días ya cerrados (dinero ya pagado) |

## Lo que un rollback de código NO cubre

Datos ya **escritos** mal por un bug de escritura (no de cálculo). Para eso es el respaldo JSON
pre-deploy: restaurar importándolo, o corregir el nodo puntual en la consola de Firebase.

## Referencias

- Reglas de la base y sus trampas: `firebase-permisos-analisis.md`
- Las reglas RTDB viven en `firebase-rules-v3.json` y se pegan A MANO en la consola de Firebase
  (no se despliegan con git push)
