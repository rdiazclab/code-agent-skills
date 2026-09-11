# Paginación idéntica al mock

```ts
// src/common/pagination/page-query.schema.ts
export const pageQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(100).default(25),
  sort: z.string().regex(/^[a-zA-Z]+:(asc|desc)(,[a-zA-Z]+:(asc|desc))*$/).optional(),
  q: z.string().trim().min(1).optional(),
});
```

```ts
// src/common/pagination/paginate.ts
export async function paginate<T>(qb: SelectQueryBuilder<T>, query: PageQuery): Promise<Paginated<T>> {
  const [rows, total] = await qb
    .skip((query.page - 1) * query.pageSize)
    .take(query.pageSize)
    .getManyAndCount();

  return {
    data: rows,
    meta: {
      page: query.page,
      pageSize: query.pageSize,
      total,
      totalPages: Math.ceil(total / query.pageSize),
    },
  };
}
```

`applySort(qb, query.sort, ALLOWED_SORT_FIELDS)` valida el campo contra una lista blanca —
nunca interpolar el nombre de columna que llega del cliente en el SQL.

El `meta` debe salir con **las mismas cuatro claves y los mismos nombres** que devuelve
`mocks/lib/paginate.ts`. Es el punto donde más fácil se cuela una deriva.
