# <Módulo> — Plan de implementación backend

**Fuente del contrato:** `<front>/docs/blueprint/<modulo>/02-contracts.md` + handlers MSW
**Fecha:** <YYYY-MM-DD> · **Estado:** borrador | aprobado

## 1. Endpoints en alcance
| ID | Método | Ruta | Permiso | Reglas | Estado |
| --- | --- | --- | --- | --- | --- |
| `PARAM-SEDES-E01` | GET | `/locations` | `can_manage_locations` | — | pendiente |

## 2. Modelo de datos
### Tabla `locations`
| Columna | Tipo | Null | Default | Índice | Origen |
| --- | --- | --- | --- | --- | --- |
| `id` | uuid | no | gen | PK | servidor |
| `code` | varchar(20) | no | — | único | contrato |

**Migraciones previstas:** `CreateLocations`, `AddManagerUserIdToLocations`

## 3. Reglas de negocio
| ID | Regla | Endpoint | Error | Dónde se implementa |
| --- | --- | --- | --- | --- |
| `R03` | Código de sede único | E03, E04 | 422 `LOCATION_CODE_TAKEN` | servicio + índice único |

## 4. Entidades compartidas
| Entidad | ¿Existe ya? | Módulo dueño | Acción |
| --- | --- | --- | --- |

## 5. Diferencias detectadas entre contrato y handlers
| # | Contrato dice | Handler hace | Resolución |
| --- | --- | --- | --- |

## 6. Riesgos y preguntas abiertas
| # | Tema | Impacto | Estado |
| --- | --- | --- | --- |
