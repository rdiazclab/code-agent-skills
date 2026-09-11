---
name: contract-to-api
description: >-
  Implementa en el backend NestJS los endpoints de un módulo a partir de los contratos ya
  definidos y mockeados en el frontend nuevo: lee `02-contracts.md` y los handlers MSW como
  especificación ejecutable, y produce entidad TypeORM, migración, DTOs Zod, servicio con las
  reglas de negocio, controlador, manejo de errores, tests unitarios y tests e2e de
  conformidad, hasta poder retirar el mock cambiando una variable de entorno.
  Trigger phrases (ES): "implementa los endpoints de X", "haz el backend de X", "pasa X de
  mocks a API real", "crea el módulo NestJS de sedes", "valida que la API cumple el contrato",
  "retira los mocks de X".
  NO define contratos nuevos ni analiza el frontend viejo (usa `blueprint-module`), NO crea
  ramas ni commits (usa `git-workflow`), NO ejecuta la suite de calidad (usa `quality-gates`),
  NO abre Pull Requests (usa `pr-workflow`).
allowed-tools: Read, Grep, Glob, Bash, Write, Edit, Skill, AskUserQuestion
metadata:
  version: 1.0.0
  owner: platform-engineering
  stability: beta
  pipeline-stage: "0b"
  modes:
    - plan
    - schema
    - implement
    - conformance
    - switch
    - full
  anti-triggers:
    - "analiza el blueprint / define los contratos -> usar blueprint-module"
    - "crea la rama / haz el commit -> usar git-workflow"
    - "corre los tests y el build -> usar quality-gates"
    - "abre el PR -> usar pr-workflow"
    - "cambia el contrato -> volver a blueprint-module [contracts], no parchear aquí"
  requires:
    - quality-gates@^1.0.0
    - blueprint-module@^1.0.0   # produce la entrada: contratos + handlers MSW
  provides:
    - backend_module
    - migrations
    - conformance_report
    - mock_retirement
  consumes:
    - api_contracts
    - msw_handlers
---

# Contract to API

> **Una frase:** convierte un contrato ya mockeado en un módulo NestJS real que responde
> exactamente igual que el mock, y solo entonces apaga el mock.

---

## 1. Contexto y Propósito

### 1.1 La fuente de verdad

Decisión del proyecto: **el contrato vive en el frontend nuevo**, no en el backend ni en un
repo aparte. Concretamente:

| Artefacto | Rol |
| --- | --- |
| `docs/blueprint/<modulo>/02-contracts.md` | contrato legible: rutas, params, códigos, errores |
| `src/mocks/handlers/<modulo>/*.handlers.ts` | **especificación ejecutable**: forma exacta de la respuesta, códigos de error reales, reglas de conflicto |
| `src/mocks/data/<modulo>/*.fixtures.ts` | casos de datos con los que el frontend ya está probado |
| `docs/blueprint/<modulo>/00-analysis.md` | reglas de negocio con su evidencia |

El backend **se adapta al contrato**, no al revés. Si al implementar aparece que el contrato
está mal, no se parchea aquí: se vuelve a `blueprint-module [contracts]`, se corrige contrato
y mock, y se retoma. Un contrato que diverge de la implementación es peor que no tenerlo.

### 1.2 Qué hace

Entidad TypeORM + migración · DTOs y validación Zod · servicio con las reglas de negocio del
análisis · controlador alineado al contrato · errores en el mismo formato que el mock ·
permisos · tests unitarios · **tests e2e de conformidad** · retirada del mock.

### 1.3 Qué NO hace

| Fuera de alcance | Responsable |
| --- | --- |
| Definir o cambiar contratos | `blueprint-module [contracts]` |
| Analizar el frontend viejo | `blueprint-module [analyze]` |
| UI, hooks, componentes | `blueprint-module [implement]` |
| Ramas, commits, PRs | `git-workflow`, `pr-workflow` |
| Linter, tipos, tests, build | `quality-gates` |

### 1.4 Posición en el pipeline

`blueprint-module [analyze -> contracts -> mocks -> implement]` -> **`contract-to-api`** ->
`quality-gates` -> `git-workflow [commit]` -> `pr-workflow [create]`

El frontend **no espera** a esta skill: avanza con mocks. Esta skill entra cuando el módulo de
frontend ya funciona.

---

## 2. Activación

### 2.1 Activar cuando

- "implementa los endpoints de X", "haz el backend de X", "pasa X de mocks a API real",
  "valida que la API cumple el contrato", "retira los mocks de X".

### 2.2 Modos

| Modo | Entrada | Produce | Protocolo |
| --- | --- | --- | --- |
| `plan` | contratos del módulo | `docs/api/<modulo>/plan.md` | §5.A |
| `schema` | plan aprobado | entidad TypeORM + migración | §5.B |
| `implement` | schema aplicado | DTOs, servicio, controlador, módulo, tests unitarios | §5.C |
| `conformance` | endpoints vivos | `<modulo>.e2e-spec.ts` + `conformance.md` | §5.D |
| `switch` | conformidad verde | mocks retirados, registry actualizado | §5.E |
| `full` | nombre del módulo | todo, con checkpoint tras `plan` | §5 |

### 2.3 NO activar cuando

| Situación | Skill correcta |
| --- | --- |
| "el contrato está mal, cámbialo" | `blueprint-module [contracts]` |
| "falta una pantalla" | `blueprint-module [implement]` |
| "commitea / abre el PR" | `git-workflow` / `pr-workflow` |

---

## 3. Prerrequisitos y Entradas

### 3.1 Contrato de entrada

| Entrada | Tipo | Requerido | Origen | Default | Si falta |
| --- | --- | --- | --- | --- | --- |
| `module` | `string` | Sí | usuario | — | **PREGUNTAR** |
| `contracts_source` | `path` | Sí | repo del frontend nuevo | `../<frontend-nuevo>` | **PREGUNTAR** |
| `endpoints` | `string[]` | No | IDs del contrato | todos los del módulo | confirmar el alcance |
| `mode` | `enum` | Sí | intención | `full` | inferir de §2.2 |

### 3.2 Precondiciones verificables

```bash
test -f "$FRONT/docs/blueprint/$MODULE/02-contracts.md"   # si no -> §7.1
ls "$FRONT/src/mocks/handlers/$MODULE/"                   # si no -> §7.1
git -C . rev-parse --show-toplevel                        # debe ser el repo backend
npm run db:up && npm run migration:show                   # base viva y migraciones al día
```

> **Regla dura:** esta skill **no escribe nada** fuera del repo backend, salvo en modo
> `switch`, donde su única escritura permitida en el frontend es borrar handlers MSW ya
> sustituidos y tocar el `.env`. Cualquier otra escritura en el frontend -> §7.2.

### 3.3 Arquitectura objetivo

**Monolito modular pragmático**: un deployable, una base PostgreSQL, un módulo Nest por módulo
funcional del ERP, con fronteras explícitas. Las cuatro reglas (comunicación solo vía servicio
público, `exports` como frontera, FK en la base sin relaciones TypeORM cruzadas, dependencias
acíclicas) están en `references/06-modular-monolith.md` y son **de obligado cumplimiento** en
los modos `plan`, `schema` e `implement`.

### 3.4 Stack objetivo (del repo, no re-decidir)

NestJS 11 · TypeORM + PostgreSQL 16 · **Zod v4 para validación** (el repo ya lo usa en
`src/config/env.ts`; no hay `class-validator` y no se introduce: además refleja los esquemas
Zod del frontend) · Jest + supertest · ESLint + Prettier (`singleQuote`, `trailingComma: all`)
· `synchronize: false`, migraciones siempre explícitas.

---

## 4. Principios de Ejecución

1. **El mock es el oráculo.** Ante cualquier duda de forma —nombre de campo, envelope, código
   de error, orden por defecto— gana lo que hace el handler MSW, porque es contra eso que el
   frontend ya está probado.
2. **Ningún endpoint de más.** Se implementa lo que el contrato lista. Nada "por si acaso".
3. **Migración siempre.** Ningún cambio de esquema sin migración generada y aplicada.
   `synchronize` permanece en `false`.
4. **Las reglas de negocio viven en el servicio**, no en el controlador ni en la entidad.
   Cada regla cita su ID del análisis (`R01`) en un comentario.
5. **Fronteras de módulo inviolables.** Un módulo nunca alcanza el repositorio, la entidad ni
   la tabla de otro: pide al servicio que el módulo dueño exporta
   (`references/06-modular-monolith.md`).
6. **Errores idénticos al mock**, incluido el campo `code`. El frontend enciende mensajes
   específicos con ese `code`: cambiarlo es una rotura silenciosa.
7. **Conformidad antes que conmutación.** No se retira un mock sin su e2e verde.
8. **Idempotencia.** Re-ejecutar un modo actualiza en sitio; nunca duplica módulo, entidad ni
   migración. Antes de crear, buscar lo existente (`src/modules/<modulo>/`).

---

## 5. Protocolo

### 5.A — Modo `plan`

1. Leer `02-contracts.md` (tabla resumen + bloque de cada endpoint) y **los handlers MSW
   completos**, que contienen detalles que el markdown suele omitir: el orden por defecto, qué
   campos se generan en servidor, qué validaciones disparan 422 y con qué `code`.
2. Leer las reglas `Rnn` del `00-analysis.md` y mapear cada una al endpoint donde se aplica.
3. Derivar el modelo de datos: tablas, columnas, tipos, nulabilidad, índices, unicidad,
   claves foráneas y qué se guarda vs. qué se calcula.
4. Detectar entidades de otros módulos que este necesita (`User`, `Location`). Determinar el
   **módulo dueño** en el registry: este módulo guardará solo el id y consultará por el
   servicio del dueño. **Jamás** duplicar la tabla ni importar la entidad ajena
   (`references/06-modular-monolith.md`).
5. Escribir `docs/api/<modulo>/plan.md` con `assets/module-plan-template.md`: endpoints,
   modelo, reglas, índices, riesgos y preguntas abiertas.
6. **CHECKPOINT:** presentar el plan (tablas, endpoints, reglas, migraciones previstas) y
   pedir aprobación. Es el último punto barato para corregir el modelo de datos.

### 5.B — Modo `schema`

1. Entidad(es) en `src/modules/<modulo>/entities/*.entity.ts` según `references/03-persistence.md`.
2. Registrar en el módulo con `TypeOrmModule.forFeature([...])` (`autoLoadEntities` ya está en
   `DatabaseModule`; no tocar globs).
3. Generar y revisar la migración **a mano** antes de aplicarla:

```bash
npm run migration:generate -- src/database/migrations/Create<Modulo>
npm run migration:run
npm run migration:show
```

4. Verificar que `down()` revierte de verdad: `migration:revert` y `migration:run` otra vez.
5. Datos semilla: solo si el contrato exige catálogos fijos (p.ej. acciones de permisos), y
   siempre como migración, no como script suelto.

### 5.C — Modo `implement`

Orden obligatorio, de dentro hacia fuera:

1. **Esquemas Zod** en `dto/`: uno de entrada por operación y uno de salida por recurso. El de
   salida es el que garantiza que no se filtren campos (contraseñas, credenciales Alegra).
2. **Servicio**: reglas de negocio, con el ID `Rnn` en comentario. Lanza excepciones de dominio
   tipadas, no `HttpException` crudas.
3. **Controlador**: rutas exactas del contrato, `ZodValidationPipe` en params/query/body,
   códigos HTTP explícitos (`@HttpCode(204)` en delete), sin lógica.
4. **Guards de permisos** con la acción del contrato (`can_manage_locations`).
5. **Módulo Nest** con `exports` mínimos (el servicio, nunca el repositorio) y registro en
   `AppModule`.
6. **Tests unitarios** del servicio: una regla de negocio, un test. Repositorio mockeado.
7. Bootstrap global si es el primer módulo: ver `references/02-nest-module-conventions.md`
   §Bootstrap (prefijo `api/v1`, filtro de errores RFC 7807, pipe Zod global, CORS, paginación).

### 5.D — Modo `conformance`

El modo que hace que apagar el mock sea seguro. Detalle en `references/04-conformance-tests.md`.

1. `test/<modulo>.e2e-spec.ts` con supertest contra la app real y base de datos de test.
2. Por **cada** endpoint del contrato, verificar: código HTTP, forma exacta del cuerpo
   (envelope `data`/`meta` en colecciones, objeto pelado en detalle), tipos de campo, y
   ausencia de campos que no están en el contrato.
3. Por **cada** error del contrato, provocarlo y verificar `status` + `code` + estructura del
   problem details. Los casos salen de los handlers MSW: mismos escenarios, mismos códigos.
4. Sembrar la base con las **mismas fixtures** que usa MSW, traducidas a inserciones.
5. Escribir `docs/api/<modulo>/conformance.md`: tabla endpoint × verificación, con el veredicto
   y las diferencias encontradas.

Cualquier diferencia se resuelve **cambiando el backend**, salvo que el contrato esté
objetivamente mal — en cuyo caso §7.3.

### 5.E — Modo `switch`

1. Confirmar: e2e verde, `quality-gates` verde, migraciones aplicadas.
2. En el frontend: apuntar `VITE_API_URL` al backend real y `VITE_API_MOCKS=false`.
3. Ejecutar la suite del frontend contra la API real (los tests siguen usando MSW; esta es una
   verificación manual/smoke de los flujos principales del módulo).
4. **Borrar** los handlers MSW sustituidos (no comentarlos) y sus barriles vacíos. Las fixtures
   se conservan si otros tests las usan.
5. Actualizar el registry del frontend: endpoints a `live`.
6. Emitir el resumen con `assets/backend-summary-template.md`.

---

## 6. Trazabilidad

El ID del contrato es la costura que une los tres repos de artefactos:

```
PARAM-SEDES-E03
  docs/blueprint/parametrizacion/02-contracts.md   (frontend: definición)
  src/mocks/handlers/parametrizacion/sedes.handlers.ts  (frontend: mock)
  src/modules/parametrizacion/sedes.controller.ts  (backend: // contract: PARAM-SEDES-E03)
  test/parametrizacion.e2e-spec.ts                 (backend: it('PARAM-SEDES-E03: ...'))
```

Un `grep -r PARAM-SEDES-E03` en ambos repos debe devolver las cuatro apariciones. Si falta la
del backend, el endpoint no está implementado; si falta la del e2e, no está verificado.

---

## 7. Errores y Bloqueos

| # | Situación | Acción |
| --- | --- | --- |
| 7.1 | No hay contrato ni handlers para el módulo | **ABORTAR**. El módulo aún no pasó por `blueprint-module`. Proponer arrancar por ahí |
| 7.2 | Una escritura apunta al frontend fuera de lo permitido en `switch` | **ABORTAR**. Reportar |
| 7.3 | El contrato es inconsistente o imposible de cumplir | **PARAR**. Documentar el choque y volver a `blueprint-module [contracts]`. Prohibido "arreglarlo" solo en el backend |
| 7.4 | La entidad pertenece a otro módulo | Guardar el id + FK y consultar por el servicio del dueño. Si falta un método, **añadirlo al módulo dueño**; nunca rodear la frontera ni crear una tabla paralela |
| 7.5 | La migración generada incluye cambios ajenos al módulo | **PARAR**. Señal de esquema desincronizado; revisar antes de aplicar |
| 7.6 | e2e falla por diferencia con el mock | Corregir el backend. Si el mock está mal, §7.3 |
| 7.7 | Falta una regla de negocio en el análisis | Marcar supuesto, implementar lo mínimo defendible y anotarlo en el resumen |
| 7.8 | La base de test no está levantada | `npm run db:up` y reintentar; los e2e la requieren |

---

## 8. Salida

Al cerrar un modo: modo, módulo, endpoints tocados, archivos escritos, migraciones generadas,
estado de los tests y siguiente paso. Resumen completo solo al final de `conformance` o `full`.

---

## 9. Referencias

| Archivo | Cuándo |
| --- | --- |
| `references/01-contract-reading.md` | modo `plan`, siempre |
| `references/02-nest-module-conventions.md` | modos `implement` y bootstrap |
| `references/03-persistence.md` | modo `schema` |
| `references/04-conformance-tests.md` | modo `conformance` |
| `references/05-mock-retirement.md` | modo `switch` |
| `references/06-modular-monolith.md` | modos `plan`, `schema` e `implement`, siempre |
| `assets/*` | plantillas y snippets |
