---
name: blueprint-module
description: >-
  Porta un módulo funcional del ERP desde el frontend actual (Blueprint, solo lectura)
  hacia el frontend nuevo, siguiendo el ciclo Blueprint -> análisis funcional -> contratos
  API -> mocks MSW -> implementación -> pruebas -> validación de paridad. Produce
  artefactos trazables y versionables por módulo, sin duplicar contratos ni mocks.
  Trigger phrases (ES): "empecemos el módulo X", "porta parametrizaciones al nuevo front",
  "analiza el blueprint de X", "define los contratos de X", "genera los mocks de X",
  "valida la paridad con el front actual", "siguiente módulo".
  NO toca el frontend actual (es fuente de verdad de solo lectura), NO implementa endpoints
  reales en el backend NestJS, NO crea ramas ni commits (usa `git-workflow`), NO ejecuta la
  suite de calidad (usa `quality-gates`), NO abre Pull Requests (usa `pr-workflow`).
allowed-tools: Read, Grep, Glob, Bash, Write, Edit, Skill, AskUserQuestion
metadata:
  version: 1.0.0
  owner: platform-engineering
  stability: beta
  pipeline-stage: "0"
  modes:
    - analyze
    - contracts
    - mocks
    - scaffold
    - implement
    - validate
    - full
  anti-triggers:
    - "crea la rama / haz el commit -> usar git-workflow"
    - "corre los tests y el build -> usar quality-gates"
    - "abre el PR -> usar pr-workflow"
    - "revisa el diff -> usar code-review"
    - "implementa el endpoint en NestJS -> trabajo de backend, fuera de esta skill"
    - "arregla un bug del front actual -> el Blueprint es de solo lectura"
  requires:
    - quality-gates@^1.0.0   # solo en modos `implement` y `full`
  provides:
    - api_contracts          # entrada de contract-to-api
    - msw_handlers           # entrada de contract-to-api
    - module_analysis
    - api_contracts
    - msw_handlers
    - module_implementation
    - parity_report
  consumes:
    - blueprint_source
    - module_registry
---

# Blueprint Module

> **Una frase:** convierte un módulo del frontend viejo en un módulo del frontend nuevo,
> pasando obligatoriamente por contratos API explícitos y mocks MSW, y dejando rastro
> auditable de cada decisión.

---

## 1. Contexto y Propósito

### 1.1 El problema que resuelve

El frontend actual (`manufacturing-retail-erp-app`) **no consume ningún backend**: todo el
estado vive en un `AppContext` monolítico y en `src/data/*`. Por lo tanto los contratos de
API **no se extraen, se derivan**. Esta skill estandariza esa derivación para que no cambie
de criterio entre módulos ni entre sesiones.

### 1.2 Qué hace

- Analiza un módulo del Blueprint y produce un documento funcional (pantallas, flujos,
  entidades, acciones, validaciones, reglas de negocio inferidas).
- Deriva los contratos de API que el **nuevo** frontend necesita consumir, con IDs estables.
- Genera/actualiza handlers MSW por submódulo, alineados 1:1 con los contratos.
- Propone e implementa la estructura del módulo en el frontend nuevo.
- Valida paridad funcional contra el Blueprint y emite un resumen con pendientes y supuestos.

### 1.3 Qué NO hace

| Fuera de alcance | Responsable |
| --- | --- |
| Modificar el frontend actual | nadie: el Blueprint es **solo lectura** |
| Implementar los endpoints reales | `contract-to-api` (repo backend NestJS) |
| Ramas, commits, push | `git-workflow` |
| Linter, tipos, tests, build | `quality-gates` |
| Crear/mergear PRs | `pr-workflow` |
| Auditar bugs y seguridad del diff | `code-review` |

### 1.4 Posición en el pipeline

**Etapa 0:** `[inicio]` -> **`blueprint-module [analyze..implement]`** -> `quality-gates` ->
`git-workflow [commit]` -> `pr-workflow [create]`

El backend real lo construye después `contract-to-api`, que consume los artefactos de esta
skill: `02-contracts.md` + los handlers MSW. **Los handlers son la especificación ejecutable
del backend**, no un andamio desechable: escríbelos con ese peso.

---

## 2. Activación

### 2.1 Activar cuando

- "empecemos el módulo X", "porta X al nuevo front", "analiza el blueprint de X",
  "define los contratos de X", "genera los mocks de X", "siguiente módulo".
- Se detecta el estado: hay que construir en el frontend nuevo algo que ya existe,
  funcionalmente, en el frontend viejo.

### 2.2 Modos de operación

| Modo | Entrada mínima | Produce | Protocolo |
| --- | --- | --- | --- |
| `analyze` | nombre del módulo | `00-analysis.md`, `01-entities.md` | §5.A |
| `contracts` | análisis aprobado | `02-contracts.md` | §5.B |
| `mocks` | contratos aprobados | `src/mocks/handlers/<mod>/*` + fixtures | §5.C |
| `scaffold` | contratos aprobados | árbol de carpetas del módulo | §5.D |
| `implement` | scaffold + mocks | UI, hooks, tipos, tests | §5.E |
| `validate` | implementación | `04-parity.md` | §5.F |
| `full` | nombre del módulo | todo lo anterior, con 2 checkpoints humanos | §5 completo |

### 2.3 NO activar cuando

| Situación | Skill correcta |
| --- | --- |
| "commitea esto" / "crea la rama" | `git-workflow` |
| "pasa los gates" | `quality-gates` |
| "revisa el PR #N" | `pr-review` |
| "mejora el diseño visual de esta pantalla" | `impeccable` |

---

## 3. Prerrequisitos y Entradas Esperadas

### 3.1 Contrato de entrada

| Entrada | Tipo | Requerido | Origen | Default | Si falta |
| --- | --- | --- | --- | --- | --- |
| `module` | `string` | Sí | argumento del usuario | — | **PREGUNTAR**. No inferir del último módulo tocado |
| `submodules` | `string[]` | No | argumento / §5.A.2 | todos los detectados | derivar del Blueprint y **confirmar** |
| `mode` | `enum` | Sí | intención | `full` | inferir de §2.2 |
| `blueprint_path` | `path` | Sí | §3.2 | `../manufacturing-retail-erp-app` | **PREGUNTAR** |
| `target_path` | `path` | Sí (modos `scaffold`+) | §3.2 | repo actual | **PREGUNTAR** |
| `registry` | `yaml` | Sí | `docs/blueprint/registry.yaml` | crear vacío | crear en primer uso |

### 3.2 Resolución de rutas

```bash
# Blueprint: repo con src/components/<modulo>/ y src/context/AppContext.tsx
ls "$BLUEPRINT/src/context/AppContext.tsx" || ABORTAR -> §7.1
# Target: repo del frontend nuevo. Si no existe todavía -> §5.D.0 (bootstrap)
```

> **Regla dura:** ninguna escritura puede caer dentro de `blueprint_path`. Antes de cada
> `Write`/`Edit`, verificar que la ruta destino está bajo `target_path`. Violación -> §7.2.

### 3.3 Stack objetivo (fijo, no re-decidir por módulo)

React 19 + Vite + TypeScript estricto · TanStack Query (server state) · TanStack Router o
React Router · react-hook-form + Zod · Tailwind · MSW (mocks) · Vitest + Testing Library.
Detalles y racionales en `references/04-frontend-architecture.md`.

---

## 4. Principios de Ejecución

1. **El Blueprint es verdad funcional, nunca verdad arquitectónica.** Se copia el *qué*,
   jamás el *cómo*. Prohibido portar `AppContext`, props-drilling o componentes de 60 KB.
2. **Ningún endpoint sin justificación.** Cada endpoint del contrato cita el archivo y la
   línea del Blueprint que lo exige, o se marca explícitamente como `ASSUMPTION`.
3. **Idempotencia.** Cada artefacto tiene ruta estable y se actualiza en sitio. Antes de
   crear cualquier cosa, consultar el registry (§6) y buscar un equivalente existente.
4. **Reutilizar antes que crear.** Componente, hook, tipo o handler que ya exista en
   `shared/` se reutiliza; si hace falta generalizar, se generaliza y se documenta.
5. **Checkpoints humanos.** En modo `full`, parar y pedir aprobación tras `analyze` y tras
   `contracts`. Son los dos puntos donde un error se paga caro más adelante.
6. **Trazabilidad.** Feature `PARAM-SEDES-F01`, endpoint `PARAM-SEDES-E03`. Esos IDs
   aparecen en el análisis, el contrato, el handler MSW y el nombre del test.
7. **Sin invención de datos de negocio.** Umbrales, tarifas y catálogos salen de
   `src/data/*` del Blueprint; lo que no esté, se marca como supuesto en `03-decisions.md`.

---

## 5. Protocolo

### 5.A — Modo `analyze`

1. Localizar el módulo en el Blueprint:
   `ls $BLUEPRINT/src/components/<modulo>/` y `grep -n "<Modulo>" $BLUEPRINT/src/App.tsx`.
2. Detectar submódulos: buscar `activeSubView`/`activeTab`/tabs en el hub del módulo y las
   vistas/modales hermanos. Listarlos y **confirmar el alcance con el usuario**.
3. Por cada submódulo, extraer lo que indica `references/01-blueprint-analysis.md`:
   pantallas, acciones, campos, validaciones, estados vacíos/carga/error, permisos (RBAC),
   y las reglas de negocio inferibles del `AppContext`.
4. Extraer entidades y relaciones desde `src/types/index.ts` (interfaces citadas por el
   módulo) y los datos semilla de `src/data/*`.
5. Escribir `docs/blueprint/<modulo>/00-analysis.md` y `01-entities.md` usando
   `assets/analysis-template.md`. Asignar IDs `F01..Fnn`.
6. **CHECKPOINT 1:** presentar resumen (submódulos, nº de features, entidades, zonas
   ambiguas) y pedir aprobación antes de continuar.

### 5.B — Modo `contracts`

1. Leer `00-analysis.md`. Por cada feature `Fnn`, decidir qué operaciones de servidor exige.
2. Consultar el registry: si otra entidad/endpoint ya cubre el caso (p.ej. `GET /users`
   definido por otro módulo), **reutilizar el contrato existente** y referenciarlo. Nunca
   redefinir.
3. Aplicar íntegramente las convenciones de `references/02-api-contracts.md` (versionado,
   naming, paginación, filtros, envelope, errores RFC 7807, códigos, concurrencia, dinero,
   fechas). No improvisar variantes por módulo.
4. Escribir `docs/blueprint/<modulo>/02-contracts.md` con `assets/contract-template.md`:
   un bloque por endpoint con ID, método, ruta, params, headers, request, response, códigos,
   errores y **la cita del Blueprint que lo justifica**.
5. Registrar supuestos y decisiones en `03-decisions.md`.
6. **CHECKPOINT 2:** presentar la tabla de endpoints y pedir aprobación. Ese documento es lo
   que se entrega al equipo de backend.

### 5.C — Modo `mocks`

1. Un archivo de handlers por submódulo: `src/mocks/handlers/<modulo>/<submodulo>.handlers.ts`.
   Registrarlo en `src/mocks/handlers/<modulo>/index.ts` y este en `src/mocks/handlers/index.ts`.
   Nunca un archivo de handlers monolítico.
2. Fixtures deterministas en `src/mocks/data/<modulo>/<submodulo>.fixtures.ts`, sembrados con
   los datos reales del Blueprint (`src/data/*`), no con `faker` aleatorio sin semilla.
3. Cada handler abre con `// contract: PARAM-SEDES-E03`. Debe cumplir el contrato literal:
   mismos códigos, mismo envelope, mismos errores.
4. Cubrir también los caminos infelices: 400/422 de validación, 404, 409 de conflicto, y un
   interruptor de latencia/fallo para probar estados de carga y error.
5. Reglas de organización y utilidades compartidas en `references/03-msw-mocks.md`.
6. Verificar: los handlers tipan contra los mismos tipos Zod/TS que consume la app.

### 5.D — Modo `scaffold`

0. **Bootstrap (solo la primera vez):** si `target_path` no tiene proyecto, crearlo según
   `references/04-frontend-architecture.md` §Bootstrap (Vite + TS estricto, Query, Router,
   Tailwind, MSW, Vitest, alias `@/`, ESLint/Prettier, estructura base `app/ shared/
   features/ mocks/`). Este paso ocurre **una vez en todo el proyecto**, no por módulo.
1. Crear `src/features/<modulo>/<submodulo>/{api,model,ui,lib,__tests__}` solo con las
   carpetas que el submódulo realmente usa.
2. Declarar tipos y esquemas Zod en `model/` derivados del contrato, no del Blueprint.
3. Registrar las rutas del módulo en el router, protegidas por permiso cuando aplique.

### 5.E — Modo `implement`

1. Implementar en este orden: `model/` (tipos+Zod) -> `api/` (endpoints + query keys +
   hooks) -> `ui/` (componentes) -> `__tests__/`.
2. Respetar las reglas de `references/04-frontend-architecture.md`: componentes tontos +
   hooks con la lógica, estados explícitos (`pending | error | empty | success`), errores
   normalizados, formularios con RHF+Zod, accesibilidad (labels, foco, roles, teclado).
3. Tests mínimos por submódulo: render de lista (éxito, vacío, error), flujo de creación
   feliz, validación que falla, y una mutación con invalidación de caché. Corren contra los
   handlers MSW ya escritos.
4. Al terminar, invocar `quality-gates`. No dar el módulo por hecho sin gate verde.

### 5.F — Modo `validate`

1. Recorrer la checklist de `references/05-parity-validation.md` feature por feature (`Fnn`).
2. Marcar cada una: `paridad | mejorada | diferida | descartada (con razón)`.
3. Escribir `docs/blueprint/<modulo>/04-parity.md` y actualizar el registry (§6).
4. Emitir el resumen final con `assets/summary-template.md`: qué se hizo, decisiones,
   contratos creados/reutilizados, supuestos, pendientes y qué necesita el backend.
5. Handoff: la lista de endpoints del resumen es la entrada de `contract-to-api`. Mientras el
   backend no exista, el módulo se considera **terminado y entregable** en modo mock.

---

## 6. Registry y Trazabilidad

`docs/blueprint/registry.yaml` es la fuente única del estado del proyecto. Se lee al empezar
cualquier modo y se actualiza al terminarlo.

```yaml
modules:
  parametrizacion:
    status: { analyze: done, contracts: done, mocks: done, scaffold: done, implement: partial, validate: pending }
    submodules: [sedes, rbac, terceros, costing, taxonomy]
    entities: [Location, User, RolePermissionsConfig, SystemParametrization, Taxonomy]
    endpoints: [PARAM-SEDES-E01, PARAM-SEDES-E02]
shared_entities:          # evita contratos duplicados entre módulos
  User:   { owner: parametrizacion, endpoints_prefix: /api/v1/users }
  Location: { owner: parametrizacion, endpoints_prefix: /api/v1/locations }
```

Antes de definir un endpoint sobre una entidad listada en `shared_entities` cuyo `owner` es
otro módulo: reutilizar, y si hace falta extenderlo, anotar la extensión en el módulo dueño.

---

## 7. Errores y Bloqueos

| # | Situación | Acción |
| --- | --- | --- |
| 7.1 | No se encuentra el Blueprint | **ABORTAR**. Pedir la ruta al usuario |
| 7.2 | Una escritura apunta dentro del Blueprint | **ABORTAR** la operación. Reportar. El Blueprint es inmutable |
| 7.3 | El módulo no existe en el Blueprint | Preguntar si es funcionalidad nueva; si lo es, esta skill no aplica al paso `analyze` y se arranca en `contracts` con requisitos del usuario |
| 7.4 | Contrato en conflicto con uno existente | **PARAR**. Mostrar ambos y pedir decisión. Nunca duplicar |
| 7.5 | El submódulo es ambiguo o gigantesco (>1 pantalla compleja) | Proponer partirlo y confirmar antes de seguir |
| 7.6 | `quality-gates` falla | Corregir y reintentar. No avanzar a `validate` en rojo |
| 7.7 | Falta un dato de negocio (umbral, tarifa) | Marcar `ASSUMPTION` en `03-decisions.md` y seguir. No inventar en silencio el valor en el código |

---

## 8. Salida

Al cerrar cualquier modo, imprimir un bloque corto: modo ejecutado, módulo/submódulos,
artefactos escritos (rutas), IDs creados, checkpoint pendiente si lo hay, y siguiente paso
sugerido. El resumen largo solo al final de `validate` o `full`.

---

## 9. Referencias

| Archivo | Cuándo leerlo |
| --- | --- |
| `references/01-blueprint-analysis.md` | modo `analyze`, siempre |
| `references/02-api-contracts.md` | modo `contracts`, siempre |
| `references/03-msw-mocks.md` | modo `mocks`, siempre |
| `references/04-frontend-architecture.md` | modos `scaffold` e `implement`, siempre |
| `references/05-parity-validation.md` | modo `validate` |
| `references/blueprint-map.md` | mapa del frontend actual: dónde vive cada módulo |
| `assets/*` | plantillas de los documentos y snippets de código base |
