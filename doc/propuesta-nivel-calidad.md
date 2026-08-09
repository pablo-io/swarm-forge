# Propuesta: Nivel de calidad configurable por proyecto

> **Estado**: propuesta pendiente de implementación — documento de trabajo.
> **Origen**: evaluación de la corrida `saas-prototype` (ver `REPORT.md` del proyecto): los gates
> fijos (CRAP≤6, mutation run, DRY, soft Gherkin) cuestan tokens que no todos los proyectos
> necesitan. Esta propuesta añade un **eje de calidad** configurable por proyecto.
> **Nota**: la revisión de two-pack (sección 5) acota la propuesta — el valor real es la
> profundidad dentro de un pack, no sustituir la elección de pack.

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

## 5. Revisión: ¿two-pack ya resuelve parte de esto?

**Resultado de la revisión del alcance real de two-pack (proyecto original):**

- **coder (two-pack)**: TDD + unit tests SOLO — excluye explícitamente acceptance, Gherkin, IR,
  Gherkin mutation, property tests, CRAP, DRY y language mutation.
- **cleaner (two-pack, batch)**: coverage, **CRAP≤6**, **DRY**, estructura/encapsulación/dependencias
  y **mutation run sobre comportamiento no cubierto** + tests para matar mutantes.

**Perfil de calidad de two-pack**: unit tests ✓ · CRAP≤6 ✓ · DRY ✓ · mutation run ✓ ·
estructura ✓ (dentro del cleaner) · acceptance/Gherkin ✗ · property ✗ · QA separado ✗.

### Conclusión

1. **two-pack NO es `minimal`**: mantiene los gates duros de hardening (CRAP≤6, DRY, mutation
   run). Es "hardening completo sin especificación" — no es la opción barata en el eje de
   profundidad.
2. **Los packs ya codifican un eje de calidad**: qué gates EXISTEN (two-pack: sin spec;
   four-pack: spec + arquitectura; six-pack: + hardender + QA).
3. **El eje de nivel añade lo que los packs NO cubren**: la PROFUNDIDAD de cada gate activo
   (CRAP≤6 vs "mejorar razonable"; mutation run vs scan-only; soft Gherkin on/off).
4. **Implicación práctica**: para "barato", elegir two-pack ya descarta las capas caras
   (spec/arquitectura) — la palanca principal de coste es el pack. El nivel sirve para escalar
   profundidad DENTRO de un pack (p. ej. two-pack sin mutation run, four-pack sin Gherkin
   mutation). La corrida lo confirmó: el derroche principal fue four-pack para un login form,
   no la profundidad de los gates.

**Veredicto**: la propuesta del nivel sigue siendo válida pero es **más estrecha** de lo que
parecía: su valor real es la profundidad dentro de un pack, no sustituir la elección de pack.
Posible simplificación: empezar por solo dos niveles (standard = actual, light = sin mutation
run ni Gherkin mutation) y dejar que la elección de pack haga el resto.

## 6. Análisis: prioridad de spec vs hardening (la crítica a two-pack)

**La lógica de two-pack**: TDD ya especifica el comportamiento a nivel de unit tests; Gherkin es
una segunda capa (contrato revisable + acceptance end-to-end) que es cara (pipeline APS); los
gates de hardening (CRAP≤6, DRY, mutation run) son el piso de calidad del código.

**La crítica (válida)**: para una tarea pequeña la prioridad está invertida — el mutation run es
caro y protege código que quizá se tire en un prototipo; la spec barata asegura que se construya
lo CORRECTO. Una feature bien endurecida pero equivocada sigue siendo equivocada. Orden lógico:
primero QUÉ (spec), después CÓMO (gates).

| | two-pack | variante spec-first (propuesta) |
|---|---|---|
| Spec base | unit tests (TDD) | Gherkin ligero (contrato revisable + aprobación humana) + TDD |
| Protección código | CRAP≤6 + DRY + **mutation run** | CRAP/DRY razonable, **sin mutation run** |
| Coste | ~2-3x | ~1.5-2x |
| Riesgo que cubre | código sucio/incambiable | **construir lo incorrecto** |

**El hueco que revela**: two-pack asume que Gherkin viene con el coste completo del pipeline APS
(parser + entrypoint generator + runtime + step handlers). No ofrece la variante "spec-lite":
escribir el Gherkin como contrato revisable sin construir el pipeline ni correr mutation.

**Refinamiento del nivel `light`**:

> `light` = Gherkin escrito como contrato + aprobación humana + TDD + CRAP/DRY razonable —
> **sin pipeline APS, sin mutation run, sin Gherkin mutation, sin property tests**.

Esta variante cubre el riesgo más importante (¿es lo correcto? ¿lo aprueba el humano?) a menor
coste que "mutation run sin spec".
