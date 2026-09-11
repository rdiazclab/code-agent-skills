# Mapa del Blueprint — `manufacturing-retail-erp-app`

Reconocimiento hecho el 2026-09-11 sobre el commit inicial `9d1eb7e`. Evita repetir la
exploración en cada módulo. Si el Blueprint cambia, reverificar antes de confiar en esto.

## Naturaleza del Blueprint

App React 19 + Vite + Tailwind 4 generada en AI Studio. **No consume backend REST**: el único
`fetch` del proyecto es `POST /api/analyze-quote` (Gemini) en
`src/components/purchasing/RequisitionQuotingWorkspaceModal.tsx:653`, servido por `server.ts`.
Todo lo demás es estado en memoria en `src/context/AppContext.tsx` (~4.672 líneas) sembrado
desde `src/data/*`. **Consecuencia: los contratos se derivan, no se extraen.**

## Módulos (tabs de `src/App.tsx`)

| Tab | Carpeta | Peso |
| --- | --- | --- |
| `dashboard` | `components/dashboard/` | 96K |
| `pos` | `components/pos/` | 80K |
| `customers` | `components/customers/` | 28K |
| `production` | `components/production/` | 536K |
| `raw_materials` | `components/rawmaterials/` | 116K |
| `suppliers` | `components/suppliers/` | 176K |
| `compras` | `components/purchasing/` | 184K |
| `inventory` | `components/inventory/` | 656K |
| `traceability` | `components/traceability/` | 48K |
| `alegra` | `components/alegra/` | 60K |
| `facturacion` / `cartera` | `components/receivables/` | 124K |
| `comisiones` / `gastos` | `components/finance/`, `components/labor/` | 228K / 48K |
| `contabilidad` | `components/accounting/` | 376K |
| `parametrization` | `components/parametrization/` | 144K |
| `manual` | `components/manual/` | 24K |

Transversales: `components/common/` (92K).

## Módulo `parametrization` (primer módulo a portar)

Archivos:

| Archivo | Rol |
| --- | --- |
| `ParametrizationHub.tsx` (63 KB) | hub con las sub-vistas; contiene costeo y taxonomías inline |
| `LocationManagerView.tsx` (30 KB) | sedes / puntos de venta |
| `RolePermissionsManagerView.tsx` (21 KB) | roles y permisos (RBAC) |
| `AdvisorManagerModal.tsx` (23 KB) | alta/edición de usuarios-asesores |

Sub-vistas (`ParametrizationHub.tsx:71`):
`'all' | 'sedes' | 'rbac' | 'terceros' | 'costing' | 'taxonomy'`.

Submódulos propuestos y su contrato de datos con el contexto:

| Submódulo | Lee | Muta |
| --- | --- | --- |
| `sedes` | `locations`, `users`, `activeLocationId` | `addLocation`, `updateLocation`, `deleteLocation`, `toggleLocationStatus`, `setActiveLocationId` |
| `rbac` | `rolePermissions`, `users`, `activeRole` | `updateRolePermissions`, `resetRolePermissionsToDefault`, `setActiveRole` |
| `terceros` (usuarios) | `users`, `locations` | `addUser`, `updateUser`, `deleteUser`, `toggleUserStatus` |
| `costing` | `parametrization`, `salesTransactions` | `updateParametrization`, `changeValuationMethod` |
| `taxonomy` | `rawMaterialCategories`, `rawMaterialSubcategories`, `rawMaterialUnits`, `rawMaterials` | `addRawMaterialCategory`, `removeRawMaterialCategory`, `addRawMaterialSubcategory`, `removeRawMaterialSubcategory`, `addRawMaterialUnit`, `removeRawMaterialUnit` |

Entidades en `src/types/index.ts`:

| Entidad | Línea | Notas para el contrato |
| --- | --- | --- |
| `Location` | 16 | `type: factory\|hub\|store`, `active`, `posTerminalsCount` |
| `PermissionAction` | 31 | 16 acciones booleanas; es el catálogo del RBAC |
| `RolePermissionsConfig` | 49 | `badgeClass` es **presentación**: no viaja en la API |
| `User` | 59 | `bankInfo` anidado; `customPermissions` sobreescribe el rol; `avatarUrl` es presentación |
| `StockBandsConfig` | 1203 | bandas de stock, parte de la configuración |
| `SystemParametrization` | 1220 | **singleton** de configuración: costeo, valuación, matriz de aging, descuentos, política Alegra (incluye `apiKeyConfigured`, `companyNit` → tratar como secreto), etapas de producción, atributos personalizados |
| `CoreUserRole` / `UserRole` | 1 / 3 | roles del sistema |
| `InventoryValuationMethod` | 14 | `FIFO \| LIFO \| AVERAGE` |

Datos semilla relevantes: `src/data/rolesData.ts` (permisos por defecto),
`src/data/mockData.ts`, `src/data/gorrasCaliforniaData.ts`, `src/data/masterReferencesEngine.ts`.

Observaciones tempranas para el análisis:

- `SystemParametrization` es configuración global singleton → `GET/PUT /system-parametrization`,
  no un CRUD de colección. Conviene partirlo en secciones para guardado parcial.
- La taxonomía (categorías / subcategorías / unidades de materia prima) es catálogo maestro
  compartido con el módulo de materias primas: candidato a `shared_entities` en el registry.
- `User` y `Location` los consumen casi todos los módulos: **este módulo es su dueño**.
- Las credenciales de Alegra dentro de `SystemParametrization` no deben devolverse nunca en
  claro; el contrato debe exponer solo `apiKeyConfigured: boolean`.
