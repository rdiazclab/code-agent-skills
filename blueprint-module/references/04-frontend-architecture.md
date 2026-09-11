# 04 — Arquitectura del frontend nuevo

Decisiones tomadas una vez para todo el proyecto. Los módulos las siguen, no las renegocian.

## Stack

| Área | Elección | Por qué |
| --- | --- | --- |
| Base | React 19 + Vite + TS `strict` | continuidad con el Blueprint, arranque rápido |
| Estado de servidor | TanStack Query | caché, reintentos, invalidación; elimina el 90% del estado global |
| Estado de UI | `useState`/`useReducer` local, Zustand solo si hay estado global real | evita repetir el `AppContext` monolítico |
| Rutas | React Router (o TanStack Router) con rutas por módulo | code-splitting natural |
| Formularios | react-hook-form + Zod | validación declarativa compartida con los tipos |
| Estilos | Tailwind | igual que el Blueprint |
| Mocks | MSW | mismos handlers en dev y en tests |
| Tests | Vitest + Testing Library + `user-event` | pruebas por comportamiento, no por implementación |

## Bootstrap (una sola vez)

```
src/
  app/            main.tsx, providers (QueryClient, Router, ErrorBoundary), router raíz
  shared/
    api/          http.ts (cliente fetch), problem.ts (error normalizado), types.ts (Paginated<T>)
    ui/           Button, Input, Select, Modal, DataTable, EmptyState, ErrorState, Skeleton…
    hooks/        useDebounce, usePagination, useConfirm…
    lib/          format (dinero, fechas), permissions, cn
  features/       un directorio por módulo
  mocks/          ver 03-msw-mocks.md
  test/           setup.ts, render con providers, helpers
```

Reglas de dependencia (verificables con ESLint `import/no-restricted-paths`):

- `features/*` puede importar de `shared/*`. **Nunca al revés.**
- Un feature **no** importa de otro feature; lo común sube a `shared/`.
- Nada de producto importa de `mocks/`.

## Anatomía de un submódulo

```
features/parametrizacion/sedes/
  model/      location.schema.ts (Zod + tipos inferidos), constants.ts
  api/        locations.api.ts (llamadas), locations.queries.ts (queryKeys + hooks)
  ui/         LocationsPage.tsx, LocationsTable.tsx, LocationForm.tsx, LocationFormModal.tsx
  lib/        mapeos y cálculos puros del submódulo
  __tests__/  locations-list.test.tsx, location-create.test.tsx
  index.ts    API pública del submódulo (solo lo que consume el router)
```

- `model/`: los tipos se derivan del **contrato**, no del Blueprint. Zod es la frontera:
  se valida lo que entra del servidor en desarrollo y siempre lo que entra del formulario.
- `api/`: una función por endpoint, nombrada por el ID del contrato en un comentario.
  Query keys jerárquicas y centralizadas:
  `locationKeys.all / .list(filters) / .detail(id)`.
- `ui/`: componentes de presentación sin `fetch`. La página compone hooks + componentes.
- Mutaciones: invalidación explícita de las keys afectadas; optimista solo donde el
  Blueprint lo justifica (toggles) y siempre con rollback.

## Estados explícitos

Toda vista que dependa de datos remotos maneja los cuatro: `pending` (skeleton, no spinner
suelto), `error` (mensaje accionable + reintento), `empty` (vacío con llamada a la acción) y
`success`. El Blueprint casi no los tiene porque trabaja en memoria: aquí son obligatorios.

## Errores

`shared/api/http.ts` normaliza cualquier fallo a un `ProblemError` con `status`, `code`,
`detail`, `errors`. Los formularios mapean `errors` a los campos con `setError` de RHF. Los
`403` se traducen a la misma UI que el permiso denegado local. No hay `catch` silenciosos ni
`alert()`.

## Permisos

Un único `usePermissions()` en `shared/`. La UI **oculta o deshabilita** con él, y de todos
modos maneja el `403` del servidor: el permiso del cliente es ergonomía, no seguridad.

## Accesibilidad (mínimos no negociables)

Etiquetas asociadas a todo input; errores enlazados con `aria-describedby` y
`aria-invalid`; modales con foco atrapado, `Esc` para cerrar y retorno del foco al disparador;
tablas con `<th scope>`; acciones solo-icono con `aria-label`; foco visible; contraste AA;
todo lo clicable alcanzable por teclado.

## Tests mínimos por submódulo

1. Lista: éxito con datos, estado vacío, estado de error.
2. Creación: flujo feliz de punta a punta contra MSW.
3. Validación: un campo inválido muestra el mensaje y no envía.
4. Mutación: tras editar/borrar, la lista refleja el cambio (invalidación correcta).

Se prueba comportamiento observable por el usuario. Nada de assertions sobre estado interno.

## Anti-patrones heredados del Blueprint (prohibidos)

- Un contexto global con todo el dominio dentro.
- Componentes de miles de líneas con varias pantallas.
- Reglas de negocio dentro de un `onClick`.
- Datos semilla importados desde el código de producto.
- Campos de presentación (`badgeClass`) mezclados con el modelo de dominio.
