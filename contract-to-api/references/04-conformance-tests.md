# 04 — Tests de conformidad

Son los que permiten apagar el mock sin miedo. No son "unos e2e más": verifican que la API
real es **indistinguible** del mock contra el que se programó el frontend.

## Preparación

```bash
npm run db:up
# .env.test con DB_NAME=erp_test  -> los e2e usan una base aparte
npm run test:e2e
```

Semilla: traducir las fixtures MSW del módulo a inserciones. Mismos IDs, mismos nombres,
mismos casos límite. Así un fallo se lee igual en ambos lados.

## Qué verifica cada endpoint

1. **Código HTTP** del caso feliz, exacto (`201` en creación, `204` sin cuerpo en borrado).
2. **Forma del cuerpo**: colección con `{ data, meta: { page, pageSize, total, totalPages } }`;
   detalle como objeto pelado.
3. **Campos**: todos los del contrato presentes con el tipo correcto **y ninguno de más**.
   Comparar el conjunto de claves, no solo las que interesan — así se detecta la filtración de
   un campo sensible o interno.
4. **Query params**: `q`, filtros, `sort`, `page`/`pageSize`, incluidos límites (`pageSize`
   sobre el máximo).
5. **Errores**: provocar cada uno del contrato y verificar `status`, `code` y estructura.
6. **Permisos**: el rol sin la acción recibe `403` con el formato del contrato.

```ts
it('PARAM-SEDES-E01: lista sedes con el envelope del contrato', async () => {
  const res = await request(app.getHttpServer())
    .get('/api/v1/locations?page=1&pageSize=25')
    .set('Authorization', `Bearer ${adminToken}`)
    .expect(200);

  expect(Object.keys(res.body).sort()).toEqual(['data', 'meta']);
  expect(res.body.meta).toEqual({ page: 1, pageSize: 25, total: 7, totalPages: 1 });
  expect(Object.keys(res.body.data[0]).sort()).toEqual(LOCATION_CONTRACT_FIELDS);
});

it('PARAM-SEDES-E03: código duplicado devuelve 422 LOCATION_CODE_TAKEN', async () => {
  const res = await request(app.getHttpServer())
    .post('/api/v1/locations')
    .set('Authorization', `Bearer ${adminToken}`)
    .send({ ...validLocation, code: 'SUR-01' })
    .expect(422);

  expect(res.body.code).toBe('LOCATION_CODE_TAKEN');
  expect(res.body.errors.code).toBeDefined();
});
```

`LOCATION_CONTRACT_FIELDS` se declara **una vez** por recurso, tomado del contrato, y se
reutiliza en todos los tests: si alguien añade una columna a la entidad y la filtra a la API
sin actualizar el contrato, estos tests fallan. Esa es la red anti-deriva del proyecto, dado
que no hay OpenAPI ni tipos generados.

## Cobertura mínima

Cada ID del contrato aparece al menos una vez en un nombre de test. `grep -c "PARAM-SEDES-E"
test/parametrizacion.e2e-spec.ts` frente a la tabla del contrato: si el número no cuadra, hay
endpoints sin verificar.

## Informe

`docs/api/<modulo>/conformance.md`: tabla endpoint × (código, forma, campos, filtros, errores,
permisos) con veredicto, y lista de diferencias encontradas con su resolución. Ese documento
es la evidencia que autoriza el modo `switch`.
