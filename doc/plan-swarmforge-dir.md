# Plan: Opción B — `SWARMFORGE_DIR` (maquinaria externa, config por proyecto)

> **Estado**: pendiente de implementación — documento de trabajo para retomar más tarde.
> **Contexto**: salió de la corrida de prueba con `saas-prototype` (ver README del fork).
> El objetivo es que los proyectos no lleven el código de SwarmForge, solo su configuración.

## 1. Objetivo

El proyecto deja de llevar el **código** de SwarmForge (scripts) y las **reglas compartidas**
(artículos, roles por defecto). Solo conserva su **configuración propia** (`swarmforge.conf` +
`project.prompt` + overrides). El fork (o cualquier ubicación compartida vía `SWARMFORGE_DIR`)
es la única fuente de la maquinaria. Con `SWARMFORGE_DIR` sin definir, **todo sigue funcionando
como ahora** (retrocompatibilidad total).

## 2. Diseño: dónde vive cada cosa

```
SWARMFORGE_DIR (base, ej. ~/.local/share/swarmforge = clon del fork)
├── swarm/ + swarmforge/scripts/          ← launcher, daemon, helpers, adaptadores
├── swarmforge/roles/*.prompt             ← roles por defecto
└── swarmforge/constitution/articles/     ← engineering, handoffs, workflow (compartidos)

PROYECTO
└── swarmforge/                            ← "delgado": SOLO lo por-proyecto
    ├── swarmforge.conf                    ← roles + modelos del proyecto
    └── constitution/articles/project.prompt (+ local-*.prompt, overrides de roles si hay)
```

**Regla de merge**: el proyecto gana por nombre — si el proyecto tiene
`roles/cleaner.prompt`, ese manda; si no, se usa el del base.

## 3. Cambios de código (archivo por archivo)

| Archivo | Cambio | Por qué |
|---|---|---|
| `swarmforge.bb` → `context` | Añadir `:base-dir` = `(or (System/getenv "SWARMFORGE_DIR") (fs/path working-dir "swarmforge"))`; mantener `:swarm-forge-dir` = `working-dir/swarmforge` (config del proyecto) | Base compartida vs. config por proyecto |
| `swarmforge.bb` → `parse-config` | `roles-dir` = lookup por rol: `proyecto/roles/<rol>.prompt` si existe, si no `base/roles/<rol>.prompt` | Overrides de roles |
| `swarmforge.bb` → nueva `sync-shared-config!` | Para cada worktree **y para master**: copiar de `base` los artículos compartidos y los roles por defecto al `swarmforge/` destino **solo si faltan** (los del proyecto ganan); scripts como hoy | El agente lee `swarmforge/constitution.prompt` relativo a su cwd — tras el sync, el view fusionado está ahí |
| `swarmforge.bb` → `prepare-workspace!` / setup | Al sincronizar en master (raíz del proyecto), añadir a `.gitignore` los archivos derivados (artículos compartidos) para que el git del proyecto quede limpio | Los shared articles se generan en el arranque, no se commitean |
| `write-agent-instruction-file!` | **Sin cambios** | Las rutas relativas siguen funcionando porque el sync fusiona el view en cada worktree |
| `check-helper-scripts!` | **Sin cambios** (valida `script-dir`, que apunta al base) | — |
| `handoffd.bb`, helpers, adaptadores | **Sin cambios** | Ya resuelven el proyecto por git (`roles.tsv`) |
| `swarm` wrapper | Pequeño ajuste de doc/instalación: al instalarlo globalmente, ejecuta `swarmforge.sh` del base; el bloque de descarga queda para el primer setup | Instalación global |

**Total estimado**: ~40-60 líneas nuevas/modificadas en `swarmforge.bb` + docs. Nada más.

## 4. Setup del usuario (una vez)

```bash
# 1. Instalar la maquinaria una sola vez
git clone https://github.com/pablo-io/swarm-forge ~/.local/share/swarmforge
ln -s ~/.local/share/swarmforge/swarm ~/.local/bin/swarm
export SWARMFORGE_DIR=~/.local/share/swarmforge   # (en tu .bashrc)

# 2. En cualquier proyecto: crear SOLO la config
mkdir -p swarmforge/constitution/articles
# swarmforge.conf + project.prompt (+ local-* / overrides si aplica)

# 3. Correr
cd /mi/proyecto && swarm
```

## 5. Plan de pruebas

1. **`bb test`** — la suite existente (24 tests) debe pasar sin cambios de semántica.
2. **Modo B**: proyecto mínimo con solo conf + `project.prompt` → lanzar con `SWARMFORGE_DIR`
   → verificar: worktrees con el view fusionado (artículos compartidos + roles + scripts),
   master funcional, arranque limpio.
3. **Handoff smoke**: un handoff real end-to-end (como el de la corrida de `saas-prototype`)
   con el modo B.
4. **Retrocompatibilidad**: `saas-prototype` (con `swarmforge/` completo) sin la env var →
   debe seguir funcionando igual.
5. **Sincronización**: cambiar un script en el fork (ej. un fix) → se refleja sin copiar nada
   al proyecto.

## 6. Migración opcional de `saas-prototype` (después de validar B)

- Adelgazar su `swarmforge/`: borrar `scripts/`, `roles/`, y los artículos compartidos; dejar
  `swarmforge.conf` + `project.prompt` (con las reglas de diseño).
- Re-copiar los prompts actualizados del fork (`architect.prompt` con la regla del reporte
  escrito) — ahora vía el base, no por copia manual.

## 7. Riesgos / decisiones abiertas

- **Los artículos compartidos sincronizados en master** quedarán gitignored (derivados) — si
  alguien quiere versionarlos explícitamente, que los commitee (el sync no pisa los existentes).
- **`roles.tsv` y el estado** siguen en `.swarmforge/` del proyecto (gitignored) — no cambia.
- La instrucción a los agentes sigue siendo relativa al worktree — clave del diseño
  (cero cambios en el protocolo).

## 8. Orden de implementación sugerido

1. `context` + `parse-config` (base dir + lookup de roles)
2. `sync-shared-config!` + gitignore de derivados
3. `bb test`
4. Prueba en modo B con un proyecto mínimo
5. Doc en el README del fork (sección "SWARMFORGE_DIR mode")
