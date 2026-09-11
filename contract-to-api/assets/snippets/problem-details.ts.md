# Errores: excepciones de dominio + filtro

```ts
// src/common/errors/domain.error.ts
export abstract class DomainError extends Error {
  abstract readonly status: number;
  constructor(readonly code: string, message: string,
              readonly fieldErrors?: Record<string, string[]>) { super(message); }
}
export class NotFoundError extends DomainError { readonly status = 404; }
export class ConflictError extends DomainError { readonly status = 409; }
export class ValidationError extends DomainError { readonly status = 422; }
export class ForbiddenError extends DomainError { readonly status = 403; }
```

```ts
// src/common/filters/problem-details.filter.ts
@Catch()
export class ProblemDetailsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const res = host.switchToHttp().getResponse<Response>();
    const { status, code, detail, errors } = normalize(exception);
    res.status(status).json({
      type: `https://errors.caps-os.local/${code.toLowerCase().replaceAll('_', '-')}`,
      title: TITLES[status] ?? 'Error',
      status, detail, code,
      ...(errors ? { errors } : {}),
      traceId: randomUUID(),
    });
  }
}
```

Registrar con `app.useGlobalFilters(new ProblemDetailsFilter())`. `normalize` cubre
`DomainError`, `HttpException` de Nest y lo desconocido (500 genérico, **sin** filtrar el
mensaje interno al cliente; ese va al log).
