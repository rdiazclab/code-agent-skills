# `mocks/handlers/<mod>/<sub>.handlers.ts`

```ts
import { http, HttpResponse } from 'msw';
import { db } from '../../lib/db';
import { paginate } from '../../lib/paginate';
import { problem, validationProblem } from '../../lib/problem';
import { withLatency } from '../../lib/latency';

export const sedesHandlers = [
  // contract: PARAM-SEDES-E01  GET /locations
  http.get('/api/v1/locations', withLatency(({ request }) => {
    const url = new URL(request.url);
    const rows = db.locations.all()
      .filter(matchesQuery(url.searchParams.get('q')))
      .filter(matchesType(url.searchParams.get('type')));
    return HttpResponse.json(paginate(rows, url.searchParams));
  })),

  // contract: PARAM-SEDES-E03  POST /locations
  http.post('/api/v1/locations', withLatency(async ({ request }) => {
    const input = await request.json();
    if (db.locations.all().some((l) => l.code === input.code)) {
      return HttpResponse.json(
        validationProblem('LOCATION_CODE_TAKEN', { code: [`Ya existe una sede con el código ${input.code}.`] }),
        { status: 422 },
      );
    }
    const created = db.locations.insert({ ...input, id: crypto.randomUUID(), active: true });
    return HttpResponse.json(created, { status: 201 });
  })),

  // contract: PARAM-SEDES-E05  DELETE /locations/{id}
  http.delete('/api/v1/locations/:id', withLatency(({ params }) => {
    const found = db.locations.byId(String(params.id));
    if (!found) return HttpResponse.json(problem(404, 'LOCATION_NOT_FOUND'), { status: 404 });
    if (db.users.all().some((u) => u.locationId === found.id && u.active)) {
      return HttpResponse.json(
        problem(409, 'LOCATION_HAS_ACTIVE_USERS', 'No se puede eliminar una sede con usuarios activos.'),
        { status: 409 },
      );
    }
    db.locations.remove(found.id);
    return new HttpResponse(null, { status: 204 });
  })),
];
```

Barril del módulo (`handlers/parametrizacion/index.ts`):

```ts
export const parametrizacionHandlers = [...sedesHandlers, ...tercerosHandlers, ...rbacHandlers];
```
