# 02 — Convenciones del módulo NestJS

## Bootstrap global (una sola vez en todo el backend)

Antes del primer módulo de dominio hay que dejar montada la infraestructura transversal.
Se hace una vez; los módulos siguientes solo la consumen.

| Pieza | Dónde | Qué resuelve |
| --- | --- | --- |
| Prefijo `api/v1` | `main.ts`: `app.setGlobalPrefix('api/v1')` | las rutas del contrato incluyen la versión |
| Filtro de errores | `src/common/filters/problem-details.filter.ts` | **toda** respuesta de error sale con el formato del contrato |
| Pipe Zod | `src/common/pipes/zod-validation.pipe.ts` | valida body/query/params y traduce a 422 con `errors` por campo |
| Paginación | `src/common/pagination/` | `page/pageSize/sort/q` → `{ data, meta }` idéntico al mock |
| CORS | `main.ts` | el frontend corre en otro puerto |
| Guard de permisos | `src/common/auth/` | `@RequirePermission('can_manage_locations')` |
| Excepciones de dominio | `src/common/errors/` | `ConflictError`, `NotFoundError`, `ValidationError` con `code` |

Sigue la línea del repo: nada de literales de configuración; todo vía `ConfigType` inyectado
desde `src/config/`.

## Estructura de un módulo

```
src/modules/<modulo>/
  <submodulo>/
    entities/location.entity.ts
    dto/location.schema.ts          Zod: create, update, query, response
    locations.service.ts            reglas de negocio (Rnn)
    locations.controller.ts         rutas del contrato, sin lógica
    locations.service.spec.ts
  <modulo>.module.ts                agrega los submódulos y los registra
```

Registrar el módulo en `AppModule`. Un módulo Nest por módulo funcional del ERP, con sus
submódulos dentro: el mapa mental del backend queda igual al del frontend.

## Validación con Zod (no `class-validator`)

El repo ya valida el entorno con Zod y no tiene `class-validator`. Se mantiene Zod: un solo
paradigma de validación, y los esquemas son **espejo** de los del frontend, lo que hace obvio
cualquier desajuste.

```ts
export const createLocationSchema = z.object({
  name: z.string().min(1).max(120),
  code: z.string().regex(/^[A-Z0-9-]{2,20}$/),
  type: z.enum(['factory', 'hub', 'store']),
  managerUserId: z.uuid().nullable().optional(),
});
export type CreateLocationDto = z.infer<typeof createLocationSchema>;
```

El `ZodValidationPipe` convierte el `ZodError` en el problem details del contrato, con
`errors` indexado por ruta de campo — la misma clave que el formulario del frontend espera.

## Esquema de salida (obligatorio)

Cada recurso tiene un `responseSchema` y el servicio devuelve `schema.parse(entity)` o un
mapeador explícito. Dos motivos: impide filtrar campos sensibles (credenciales Alegra,
hashes) y hace que una entidad que crece no se filtre sola a la API.

## Capas

| Capa | Puede | No puede |
| --- | --- | --- |
| Controlador | mapear HTTP, invocar el servicio, fijar el código de estado | tener `if` de negocio, tocar el repositorio |
| Servicio | reglas, transacciones, orquestación | conocer `Request`/`Response` |
| Repositorio/entidad | persistencia | reglas de negocio |

## Errores

El servicio lanza excepciones de dominio con `code`; el filtro las traduce a HTTP.

```ts
// R03: no puede haber dos sedes con el mismo código
if (await this.repo.existsBy({ code: dto.code })) {
  throw new ConflictError('LOCATION_CODE_TAKEN', `Ya existe una sede con el código ${dto.code}.`);
}
```

El `code` debe ser **literalmente** el del mock. Es la clave con la que el frontend enciende
mensajes concretos; cambiarlo rompe la UI sin que falle ningún tipo.

## Permisos

`@RequirePermission('can_manage_locations')` con la acción exacta del contrato. El guard
devuelve `403` con el formato del contrato. El permiso del frontend es ergonomía; **este** es
el control real.

## Tests unitarios

Una regla de negocio, un test, con el ID en el nombre:
`it('R03: rechaza una sede con código duplicado', ...)`. Repositorio mockeado; la integración
con la base se prueba en los e2e.
