# `shared/api/http.ts` — cliente y error normalizado

```ts
export type Problem = {
  type: string; title: string; status: number; detail?: string;
  code?: string; errors?: Record<string, string[]>; traceId?: string;
};

export class ProblemError extends Error {
  constructor(readonly problem: Problem) {
    super(problem.detail ?? problem.title);
    this.name = 'ProblemError';
  }
  get status() { return this.problem.status; }
  get code() { return this.problem.code; }
  get fieldErrors() { return this.problem.errors ?? {}; }
}

const BASE_URL = import.meta.env.VITE_API_URL ?? '/api/v1';

type Options = Omit<RequestInit, 'body'> & { body?: unknown; query?: QueryParams };

export async function request<T>(path: string, options: Options = {}): Promise<T> {
  const { body, query, headers, ...rest } = options;
  const url = `${BASE_URL}${path}${buildQuery(query)}`;

  const response = await fetch(url, {
    ...rest,
    headers: {
      Accept: 'application/json',
      'Accept-Language': 'es-CO',
      ...(body !== undefined ? { 'Content-Type': 'application/json' } : {}),
      ...authHeader(),
      ...headers,
    },
    body: body !== undefined ? JSON.stringify(body) : undefined,
  });

  if (response.status === 204) return undefined as T;
  const payload = await response.json().catch(() => null);
  if (!response.ok) throw new ProblemError(toProblem(payload, response.status));
  return payload as T;
}

export const http = {
  get:   <T>(p: string, o?: Options) => request<T>(p, { ...o, method: 'GET' }),
  post:  <T>(p: string, body?: unknown, o?: Options) => request<T>(p, { ...o, method: 'POST', body }),
  patch: <T>(p: string, body?: unknown, o?: Options) => request<T>(p, { ...o, method: 'PATCH', body }),
  put:   <T>(p: string, body?: unknown, o?: Options) => request<T>(p, { ...o, method: 'PUT', body }),
  delete:<T>(p: string, o?: Options) => request<T>(p, { ...o, method: 'DELETE' }),
};

export type Paginated<T> = {
  data: T[];
  meta: { page: number; pageSize: number; total: number; totalPages: number };
};
```

`toProblem` garantiza un `Problem` incluso ante un 500 con HTML: nunca se propaga un error sin
forma. Es el único sitio del frontend que conoce `fetch`.
