# Propuesta: Nivel de calidad configurable por proyecto

> **Estado**: propuesta pendiente de implementación — documento de trabajo.
> **Origen**: evaluación de la corrida `saas-prototype` (ver `REPORT.md` del proyecto): los gates
> fijos (CRAP≤6, mutation run, DRY, soft Gherkin) cuestan tokens que no todos los proyectos
> necesitan. Esta propuesta añade un **eje de calidad** configurable por proyecto.
> **Pendiente**: revisar si el alcance de `two-pack` ya cubre parte de esto (sección 5).

## 1. Los 3 niveles

| Nivel | Coste relativo | Uso típico |
|---|---|---|
| `minimal` | ~1x | Prototipos, spikes, código desechable |
| `standard` | ~2x | Default razonable para features de producto |
| `maximum` | ~3-4x | **El rigor actual** — librerías críticas, seguridad/pagos, código consumido por otros |

`maximum` = lo que ya hace el pipeline hoy (nada más allá por ahora).

## 2. Gates por nivel

| Gate | minimal | standard | maximum (actual) |
|---|---|---|---|
| TDD + unit tests | ✓ | ✓ | ✓ |
| Acceptance Gherkin | opcional | ✓ (sin pipeline APS completo) | ✓ pipeline APS completo |
| CRAP | — | mejorar lo razonable, sin gate duro | **≤6** |
| DRY | — | reducir duplicación razonable | herramienta, estricto |
| Mutation scan + split >100 sitios | — | ✓ (solo conteo — barato) | ✓ |
| Mutation run completo | — | — | ✓ diferencial, matar no-equivalentes |
| Soft Gherkin mutation | — | — | ✓ |
| Property tests | — | — | soporte |
| Reporte escrito | — | opcional | ✓ |

## 3. Responsabilidad de cada rol por nivel (four-pack)

| Rol | minimal | standard | maximum |
|---|---|---|---|
| **specifier** | Solo scoping + gate humano (sin Gherkin obligatorio) | Gherkin ✓ | Gherkin + QA suite |
| **coder** | Implementa con TDD — imprescindible | ✓ | ✓ |
| **refactorer** | Sin gates → **sin trabajo real** | CRAP razonable + DRY + mutation scan | gates completos |
| **architect** | Sin gates → **sin trabajo real** | solo revisión estructural ligera | gates completos |

**Conclusión**: el nivel y el workflow están correlacionados. En `minimal`, refactorer/architect
no tienen gates que aplicar — configurar 4 roles sería desperdiciar tokens sin beneficio.

## 4. Mecanismo (solo prompts/artículos, cero código)

```
1. PROYECTO (project.prompt):        "## Quality Level → Quality level: standard"
2. COMPARTIDO (quality.prompt):      la tabla de gates ON/OFF por nivel
3. CADA ROLE PROMPT (una línea):     "Aplica solo los gates que estén ON para
                                     el nivel del proyecto (ver artículo de calidad)"
```

El agente lee el nivel en `project.prompt`, la tabla en `quality.prompt`, y su role prompt le
dice que aplique solo los gates ON → interpretación consistente entre roles.

El artículo `quality.prompt` incluye además el **mapeo de roles por nivel**: "en minimal,
configura solo specifier+coder (2 windows en `swarmforge.conf`); en standard, añade refactorer;
en maximum, los 4".

**Matiz honesto**: el ajuste es *prompt-soft* — los agentes siguen el nivel por instrucción.
Para enforcement duro haría falta un `tools/quality-check` (validar artefactos del nivel), paso
opcional posterior.

## 5. Pregunta abierta: ¿two-pack ya resuelve parte de esto?

Pendiente de revisar el alcance real de `two-pack` en el proyecto original. Hipótesis: los packs
ya codifican una elección de calidad (qué gates existen), y el eje de nivel añadiría *profundidad*
dentro de un pack. La sección 6 registrará la comparación.
