# <Módulo> — Análisis funcional del Blueprint

- **Fuente:** `manufacturing-retail-erp-app@<commit>` · `src/components/<modulo>/`
- **Fecha:** <YYYY-MM-DD> · **Estado:** borrador | aprobado
- **Submódulos:** <lista>

## 1. Resumen

<Qué resuelve el módulo, quién lo usa, con qué frecuencia.>

## 2. Submódulo: `<submodulo>`

### 2.1 Pantallas
| Pantalla | Tipo | Archivo del Blueprint |
| --- | --- | --- |
| Listado de sedes | tabla + filtros | `LocationManagerView.tsx:120` |

### 2.2 Features
| ID | Feature | Disparador | Permiso | Blueprint |
| --- | --- | --- | --- | --- |
| `<MOD>-<SUB>-F01` | Listar X con búsqueda y filtro por estado | carga de la vista | `can_manage_locations` | `LocationManagerView.tsx:120` |

### 2.3 Campos
| Campo | Tipo | Requerido | Default | Editable | Validación | Notas |
| --- | --- | --- | --- | --- | --- | --- |

### 2.4 Reglas de negocio inferidas
| ID | Regla | Evidencia |
| --- | --- | --- |
| `R01` | No se puede desactivar una sede con usuarios activos | `AppContext.tsx:1234` |

### 2.5 Estados
| Estado | Comportamiento en el Blueprint | Comportamiento requerido en el nuevo |
| --- | --- | --- |
| carga | no existe (memoria) | skeleton de tabla |
| error | no existe | mensaje + reintento |
| vacío | <describir> | <describir> |

### 2.6 Permisos
| Rol | Ve | Puede |
| --- | --- | --- |

## 3. Entidades y relaciones
<Diagrama textual + remisión a `01-entities.md`.>

## 4. Dependencias con otros módulos
| Entidad/dato | Módulo dueño | Uso aquí |
| --- | --- | --- |

## 5. Preguntas abiertas
| # | Pregunta | Impacto si se decide mal | Estado |
| --- | --- | --- | --- |
