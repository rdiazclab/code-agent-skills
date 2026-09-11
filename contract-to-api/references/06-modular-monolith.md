# 06 — Monolito modular: fronteras entre módulos

La arquitectura del backend es un **monolito modular pragmático**: un solo deployable, una
sola base de datos PostgreSQL, y dentro un módulo Nest por módulo funcional del ERP, con
fronteras explícitas y verificables entre ellos.

No es una decisión por omisión: se elige porque el equipo es pequeño, la integridad
referencial en una sola base vale mucho, y porque permite extraer un módulo a un servicio
aparte más adelante sin haber pagado antes el coste de la distribución.

## Las cuatro reglas

### 1. Un módulo solo habla con otro a través de su servicio público

```ts
// ❌ PROHIBIDO: alcanzar el repositorio de otro módulo
constructor(@InjectRepository(Location) private locations: Repository<Location>) {}

// ❌ PROHIBIDO: consultar la tabla de otro módulo con QueryBuilder
this.dataSource.createQueryBuilder().from('locations', 'l')…

// ✅ CORRECTO: el módulo dueño expone su API
constructor(private readonly locationsApi: LocationsService) {}
```

### 2. Cada módulo declara su API pública en `exports`

Se exporta el **servicio**, nunca `TypeOrmModule.forFeature` ni el repositorio. Lo que no
está en `exports` es privado del módulo, y esa es la única superficie que otros módulos
pueden usar.

```ts
@Module({
  imports: [TypeOrmModule.forFeature([Location])],
  controllers: [LocationsController],
  providers: [LocationsService],
  exports: [LocationsService],   // ← la frontera, explícita
})
export class SedesModule {}
```

Si un módulo necesita de otro algo que su servicio no expone, se **añade el método al módulo
dueño**; no se rodea la frontera.

### 3. FK en la base, sin relaciones TypeORM cruzadas

La integridad referencial la garantiza Postgres — es lo que hace viable el `409` del
contrato (`LOCATION_HAS_ACTIVE_USERS`). Pero la entidad **no** declara `@ManyToOne` hacia una
entidad de otro módulo: guarda el id.

```ts
// entidad de otro módulo que referencia una sede
@Column({ name: 'location_id', type: 'uuid' })
locationId: string;          // FK real en la BD (creada en la migración), sin @ManyToOne
```

Así el acoplamiento es a un **id**, no al esquema completo de la otra entidad, y un cambio en
`Location` no se propaga por media aplicación. Cuando se necesiten datos de la sede, se piden
al `LocationsService`.

Excepción: dentro de un mismo módulo, las relaciones TypeORM se usan con normalidad.

### 4. Dependencias acíclicas; lo común vive en `common/`

Si A necesita B y B necesita A, la solución no es `forwardRef`: es que hay un concepto mal
ubicado. Se extrae a `common/` o se replantea quién es el dueño. `forwardRef` entre módulos de
dominio se trata como un defecto de diseño, no como una herramienta.

## Propiedad de las entidades

Cada entidad pertenece a **un** módulo, declarado en el registry. `User` y `Location` son de
`parametrizacion`. Los demás módulos:

- guardan `userId` / `locationId`,
- consultan por el servicio del dueño,
- **nunca** importan la entidad ajena ni la registran en su propio `forFeature`.

Esto sustituye a la formulación anterior ("reutiliza la entidad existente"), que era correcta
en su intención —jamás duplicar la tabla— pero ambigua en el medio: no duplicar **no**
significa importar la entidad a placer.

## Verificación automatizable

Regla de ESLint que convierte las fronteras en algo que falla en CI, no en una convención que
se erosiona:

```js
// eslint.config.mjs
{
  files: ['src/modules/**/*.ts'],
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [{
        group: ['**/modules/*/**'],
        message:
          'Un módulo no importa internals de otro módulo. Usa el servicio que el módulo dueño exporta.',
      }],
    }],
  },
}
```

Se afina con excepciones para el propio módulo y para los barriles públicos (`modules/*/index.ts`)
según se vayan añadiendo módulos. Complemento útil: un test de arquitectura que recorra
`src/modules/*` y falle si una entidad se registra en el `forFeature` de un módulo que no es su
dueño.

## Cuándo dejaría de ser suficiente

Señales de que el monolito modular se queda corto: dos módulos con ciclos de despliegue
incompatibles, un módulo que necesita escalar por separado, o equipos distintos pisándose. Con
las cuatro reglas anteriores cumplidas, extraer uno a un servicio es un trabajo acotado: ya se
comunica por una API pública y no comparte entidades.
