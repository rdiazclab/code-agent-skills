# <Módulo> — Contratos API

Base `/api/v1` · Convenciones: `references/02-api-contracts.md` · Estado: borrador | aprobado

## Resumen
| ID | Método | Ruta | Feature | Estado |
| --- | --- | --- | --- | --- |
| `PARAM-SEDES-E01` | `GET` | `/locations` | `F01` | nuevo |

---

## `PARAM-SEDES-E01` — Listar sedes

- **Feature:** `PARAM-SEDES-F01`
- **Justificación:** `LocationManagerView.tsx:120` consume `locations` del contexto con
  búsqueda por nombre/ciudad y filtro por estado.
- **Método y ruta:** `GET /api/v1/locations`
- **Permiso:** `can_manage_locations`

**Query params**

| Nombre | Tipo | Req. | Default | Notas |
| --- | --- | --- | --- | --- |
| `page` | int | no | 1 | |
| `pageSize` | int | no | 25 | máx 100 |
| `q` | string | no | — | busca en `name`, `code`, `city` |
| `type` | enum | no | — | `factory\|hub\|store` |
| `active` | bool | no | — | |
| `sort` | string | no | `name:asc` | |

**Headers:** `Authorization: Bearer <jwt>`

**Request body:** ninguno

**Response `200`**

```json
{
  "data": [{
    "id": "0f2c…", "name": "Sede Sur", "code": "SUR-01", "city": "Cali",
    "type": "store", "address": "Cra 1 #2-3", "phone": "+57…",
    "managerUserId": "8a1b…", "active": true, "openingDate": "2024-03-01",
    "posTerminalsCount": 2, "notes": null
  }],
  "meta": { "page": 1, "pageSize": 25, "total": 7, "totalPages": 1 }
}
```

**Errores**

| Código | `code` | Cuándo |
| --- | --- | --- |
| 400 | `INVALID_QUERY` | query param fuera de rango |
| 401 | `UNAUTHENTICATED` | sin token o vencido |
| 403 | `FORBIDDEN` | rol sin `can_manage_locations` |

**Notas de derivación:** el Blueprint guarda `manager` como texto libre; aquí se normaliza a
`managerUserId` referenciando `User`. Ver `03-decisions.md#D03`.
