# 03 — Mocks HTTP con MSW

Los mocks son la **implementación ejecutable del contrato**. Si el mock y el contrato
discrepan, el contrato manda y el mock se corrige.

## Estructura

```
src/mocks/
  browser.ts                 setupWorker (dev)
  server.ts                  setupServer (Vitest)
  handlers/
    index.ts                 [...parametrizacion, ...pos, ...]
    parametrizacion/
      index.ts               [...sedes, ...rbac, ...terceros]
      sedes.handlers.ts
      terceros.handlers.ts
  data/
    parametrizacion/
      sedes.fixtures.ts      semillas tomadas del Blueprint
  lib/
    paginate.ts              aplica page/pageSize/sort/q -> { data, meta }
    problem.ts               constructores de error RFC 7807
    db.ts                    store en memoria por colección, con reset()
    latency.ts               retardo configurable + inyección de fallos
```

Nunca un `handlers.ts` monolítico. Un archivo por submódulo, agregado por barriles.

## Reglas

1. Cada handler abre con el ID del contrato:

```ts
// contract: PARAM-SEDES-E02  (POST /api/v1/locations)
http.post('/api/v1/locations', async ({ request }) => { ... })
```

2. Las respuestas se construyen con los mismos tipos/esquemas Zod que consume la app
   (`import type { Location } from '@/features/parametrizacion/sedes/model'`). Si el mock no
   compila contra el tipo del contrato, el contrato está mal escrito o el tipo está desfasado.
3. **Estado mutable por sesión**: `db.ts` mantiene las colecciones en memoria para que crear,
   editar y borrar se vean reflejados en las listas. `resetDb()` corre en `beforeEach` de los
   tests para aislamiento total.
4. **Caminos infelices obligatorios** por submódulo: al menos un `422` de validación real
   (p.ej. código duplicado), un `404`, un `409` cuando exista la regla, y un `403` si la
   feature tiene RBAC. Sin ellos, la UI de error no está probada.
5. **Latencia y fallos** controlables por query flag o variable de entorno
   (`VITE_MOCK_LATENCY=800`, `VITE_MOCK_FAIL=locations:list`) para revisar a mano estados de
   carga y error sin tocar código.
6. Fixtures **deterministas**, sembradas con datos reales del Blueprint (`src/data/*`). Si se
   usa faker, siempre con semilla fija.
7. `onUnhandledRequest: 'error'` en el `server.ts` de tests: una llamada sin contrato debe
   romper la suite, no pasar de largo.

## Uso en la app

- Dev: arrancar el worker solo si `import.meta.env.VITE_API_MOCKS === 'true'`, en un módulo
  aparte importado dinámicamente. El código de producto **nunca** importa nada de `src/mocks`.
- Tests: `setupServer` en `src/test/setup.ts`, con `resetHandlers()` y `resetDb()` entre tests.
- Sobrescritura puntual en un test: `server.use(http.get(..., () => HttpResponse.json(...)))`
  para probar un error concreto sin ensuciar los handlers base.

## Migración al backend real

Requisito de diseño: apagar los mocks debe ser cambiar **una** variable de entorno. Para eso
la app siempre llama a través de `shared/api/http.ts` con `baseURL = import.meta.env.VITE_API_URL`
y rutas idénticas a las del contrato. Si algún componente conoce la existencia de los mocks,
está mal hecho.

Cuando un endpoint real esté disponible, se borra su handler (no se comenta) y se actualiza el
registry marcando ese endpoint como `live`. Los tests siguen usando MSW siempre.
