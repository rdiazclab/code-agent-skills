# `ZodValidationPipe`

```ts
@Injectable()
export class ZodValidationPipe implements PipeTransform {
  constructor(private readonly schema: ZodType) {}

  transform(value: unknown) {
    const result = this.schema.safeParse(value);
    if (result.success) return result.data;

    const errors: Record<string, string[]> = {};
    for (const issue of result.error.issues) {
      const path = issue.path.join('.') || '_';
      (errors[path] ??= []).push(issue.message);
    }
    throw new ValidationError('VALIDATION_FAILED', 'Los datos enviados no son válidos.', errors);
  }
}
```

Uso en el controlador, con el ID del contrato a la vista:

```ts
@Controller('locations')
export class LocationsController {
  // contract: PARAM-SEDES-E03
  @Post()
  @RequirePermission('can_manage_locations')
  create(@Body(new ZodValidationPipe(createLocationSchema)) dto: CreateLocationDto) {
    return this.service.create(dto);
  }

  // contract: PARAM-SEDES-E05
  @Delete(':id')
  @HttpCode(204)
  @RequirePermission('can_manage_locations')
  remove(@Param('id', ParseUUIDPipe) id: string) {
    return this.service.remove(id);
  }
}
```

Las claves de `errors` deben coincidir con los nombres de campo del formulario del frontend:
es lo que permite a RHF pintar el error en el input correcto.
