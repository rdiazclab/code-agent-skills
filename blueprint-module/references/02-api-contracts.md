# 02 — Convenciones de contratos API

Estas convenciones son **del proyecto**, no del módulo. No se re-deciden en cada iteración.
Cualquier desviación se justifica en `03-decisions.md` del módulo.

## Principios

1. El contrato modela **recursos del dominio**, no la forma del estado de React.
2. Solo se define lo que el frontend nuevo va a consumir. Nada especulativo.
3. Todo endpoint cita la feature que lo exige (`PARAM-SEDES-F02`) y el archivo:línea del
   Blueprint que la respalda. Sin respaldo -> etiqueta `ASSUMPTION` y entra en decisiones.
4. Una entidad tiene **un solo dueño**. El registry dice quién. Los demás módulos reutilizan.

## Forma

- Base: `/api/v1`. El versionado es de toda la API, no por recurso.
- Recursos en **plural, kebab-case**: `/locations`, `/role-permissions`, `/point-of-sales`.
- Jerarquía solo cuando el hijo no existe sin el padre: `/locations/{id}/pos-terminals`.
  Si el hijo se consulta globalmente, es recurso de primer nivel con filtro.
- Acciones que no son CRUD: sub-recurso en imperativo, `POST /users/{id}/deactivate`.
  Preferible a un `PATCH` con un flag mágico cuando dispara reglas de negocio propias.
- IDs: UUID v4 en string. Nada de índices de array ni IDs derivados del nombre.

## Colecciones

```
GET /api/v1/locations?page=1&pageSize=25&sort=name:asc&q=cali&type=store&active=true
```

- Paginación: `page` (1-based) y `pageSize` (default 25, máx 100).
- Orden: `sort=<campo>:<asc|desc>`, admite varios separados por coma.
- Búsqueda libre: `q`. Filtros exactos: un query param por campo.
- Respuesta **siempre envuelta**:

```json
{
  "data": [ { "id": "...", "name": "..." } ],
  "meta": { "page": 1, "pageSize": 25, "total": 7, "totalPages": 1 }
}
```

- Recurso individual: **sin envelope**, el objeto directo.
- `POST` devuelve `201` + el recurso creado. `PATCH` devuelve `200` + el recurso actualizado.
  `DELETE` devuelve `204` sin cuerpo.
- `PATCH` para edición parcial (lo normal en formularios). `PUT` solo para reemplazo total
  de configuraciones singleton (p.ej. `PUT /system-parametrization`).

## Errores

Formato único, basado en RFC 7807, para todos los errores:

```json
{
  "type": "https://errors.caps-os.local/validation-failed",
  "title": "Validation failed",
  "status": 422,
  "detail": "El código de sede ya está en uso.",
  "code": "LOCATION_CODE_TAKEN",
  "errors": { "code": ["Ya existe una sede con el código SUR-01."] },
  "traceId": "b3f1..."
}
```

- `code` es el identificador estable que consume el frontend para mensajes específicos.
  `detail` es texto para humanos y puede cambiar sin romper nada.
- `errors` solo en errores de validación, con clave = ruta del campo del formulario.

| Código | Cuándo |
| --- | --- |
| `200` | lectura o actualización correcta |
| `201` | creación |
| `204` | borrado / acción sin respuesta |
| `400` | petición malformada (query param inválido) |
| `401` | sin sesión o token vencido |
| `403` | autenticado pero sin permiso (mapea al RBAC del módulo) |
| `404` | recurso inexistente |
| `409` | conflicto de estado (borrar una sede con usuarios activos) |
| `422` | validación de negocio/campos |
| `500` | fallo del servidor |

## Tipos de datos

- Fechas y timestamps: **ISO-8601 en UTC** con sufijo `Z`. Fechas sin hora: `YYYY-MM-DD`.
- Dinero: entero en **unidades menores** + campo de moneda explícito
  (`{ "amountMinor": 2500000, "currency": "COP" }`). Nunca float.
  El Blueprint usa enteros COP sueltos (`*_COP`); al portar, se normaliza a este objeto.
- Porcentajes: número decimal `0..100`, no fracción. Nombre con sufijo `Percent`.
- Enums: `snake_case` en minúsculas en el cable (`punto_venta`), traducidos a etiquetas en el
  frontend. Nunca enviar la etiqueta visible como valor.
- Booleanos: sin negación en el nombre (`active`, no `inactive`).
- Campos de presentación del Blueprint (`badgeClass`, iconos, colores) **no viajan**: son
  responsabilidad del frontend nuevo.

## Cabeceras

- `Authorization: Bearer <jwt>` en todo endpoint autenticado.
- `Content-Type: application/json`. `Accept-Language: es-CO`.
- `Idempotency-Key` en `POST` que generan efectos de negocio no repetibles.
- Concurrencia optimista donde haya edición concurrente real: `ETag` en la lectura +
  `If-Match` en la escritura, con `412` si no coincide. No aplicarlo por defecto a todo.

## Documento del contrato

`docs/blueprint/<modulo>/02-contracts.md`, un bloque por endpoint (ver
`assets/contract-template.md`), precedido de una tabla resumen:

| ID | Método | Ruta | Feature | Estado |
| --- | --- | --- | --- | --- |
| `PARAM-SEDES-E01` | `GET` | `/api/v1/locations` | `F01` | nuevo |
| `PARAM-TERC-E04` | `GET` | `/api/v1/users` | `F07` | reutiliza `PARAM-TERC-E01` |

Esa tabla es el entregable que recibe backend.
