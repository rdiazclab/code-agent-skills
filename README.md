# code-agent-skills

Skills de Claude Code para el desarrollo del ERP de manufactura y retail (CAPS OS).

Estandarizan el proceso de reconstruir el frontend módulo por módulo, usando el frontend
actual como Blueprint funcional, y de implementar después el backend contra los contratos ya
mockeados.

## Skills

### `blueprint-module` — del frontend viejo al frontend nuevo

Ciclo **Blueprint → análisis funcional → contratos API → mocks MSW → implementación →
pruebas → validación de paridad**.

| Modo | Produce |
| --- | --- |
| `analyze` | `00-analysis.md`, `01-entities.md` ◆ checkpoint humano |
| `contracts` | `02-contracts.md`, `03-decisions.md` ◆ checkpoint humano |
| `mocks` | handlers MSW + fixtures, por submódulo |
| `scaffold` | estructura del módulo (y bootstrap del proyecto la primera vez) |
| `implement` | model → api → ui → tests |
| `validate` | `04-parity.md` + resumen |

Reglas duras: el Blueprint es **inmutable** (fuente de verdad funcional, nunca arquitectónica);
ningún endpoint sin cita `archivo:línea` que lo justifique; el registry impide redefinir
entidades compartidas entre módulos.

### `contract-to-api` — del contrato mockeado al backend NestJS

| Modo | Produce |
| --- | --- |
| `plan` | `docs/api/<mod>/plan.md` ◆ checkpoint humano |
| `schema` | entidad TypeORM + migración revisada |
| `implement` | DTOs Zod, servicio, controlador, guards, tests unitarios |
| `conformance` | e2e que verifican que la API real es indistinguible del mock |
| `switch` | retirada del mock, endpoint a `live` |

Reglas duras: **el mock es el oráculo**; si el contrato está mal se corrige en
`blueprint-module [contracts]`, nunca se parchea en el backend; ninguna migración implícita;
no se apaga un mock sin su e2e de conformidad en verde.

## Cómo se encadenan

```
blueprint-module [analyze → contracts → mocks] ──▶ contract-to-api [plan → … → switch]
                          │                                        │
                          └─▶ [scaffold → implement → validate]    └─▶ mock retirado
```

El backend arranca en cuanto existen los mocks; frontend y backend avanzan en paralelo contra
el mismo contrato. El ID del contrato es la costura que une todo:

```
PARAM-SEDES-E03
  docs/blueprint/<mod>/02-contracts.md          definición
  src/mocks/handlers/<mod>/*.handlers.ts        mock (especificación ejecutable)
  src/modules/<mod>/*.controller.ts             implementación
  test/<mod>.e2e-spec.ts                        verificación
```

## Instalación

```bash
git clone git@github.com:rdiazclab/code-agent-skills.git ~/work/code-agent-skills
ln -s ~/work/code-agent-skills/blueprint-module ~/.claude/skills/blueprint-module
ln -s ~/work/code-agent-skills/contract-to-api  ~/.claude/skills/contract-to-api
```

## Convenciones del proyecto que estas skills fijan

- **Frontend nuevo:** React + Vite + TS estricto, TanStack Query, react-hook-form + Zod,
  Tailwind, MSW, Vitest + Testing Library.
- **Backend:** NestJS 11 + TypeORM + PostgreSQL 16, validación con **Zod** (no
  `class-validator`), migraciones siempre explícitas, Jest + supertest.
- **API:** `/api/v1`, recursos plurales kebab-case, colecciones con envelope `{ data, meta }`,
  errores estilo RFC 7807 con `code` estable, dinero en unidades menores + `currency`,
  fechas ISO-8601 UTC.
- **Fuente de verdad del contrato:** el repo del frontend nuevo. No hay OpenAPI ni repo de
  contratos aparte; la red anti-deriva son los tests de conformidad.

## Skills relacionadas (otra fuente, no viven aquí)

`git-workflow`, `quality-gates`, `code-review`, `pr-workflow`, `pr-review`.
