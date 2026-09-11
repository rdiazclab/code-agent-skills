# 03 — Persistencia: entidades y migraciones

## Reglas duras

- `synchronize: false` siempre; en producción se fuerza pase lo que pase. **Toda** evolución
  del esquema es una migración revisada a mano.
- Una migración por cambio lógico y por módulo. Nombre descriptivo: `CreateLocations`,
  `AddManagerUserIdToLocations`.
- Revisar el SQL generado **antes** de aplicarlo. Si toca tablas de otro módulo → §7.5.
- Verificar el `down()`: `migration:revert` + `migration:run` debe dejar la base igual.

```bash
npm run migration:generate -- src/database/migrations/CreateLocations
npm run migration:run
npm run migration:revert && npm run migration:run   # verificación del down()
npm run migration:show
```

## Convenciones de esquema

| Aspecto | Convención |
| --- | --- |
| Tablas | plural, `snake_case`: `locations`, `role_permissions` |
| Columnas | `snake_case` en la base, `camelCase` en la entidad |
| PK | `uuid` con `@PrimaryGeneratedColumn('uuid')` |
| Timestamps | `created_at`, `updated_at` (`@CreateDateColumn`/`@UpdateDateColumn`), `timestamptz` |
| Baja lógica | `@DeleteDateColumn` cuando el contrato distingue "inactivo" de "eliminado" |
| Enums | columna `varchar` + `CHECK` o enum de Postgres; valores idénticos a los del contrato (`snake_case`) |
| Dinero | `bigint` en unidades menores + columna `currency` `char(3)`. **Nunca** `float`/`real` |
| Porcentajes | `numeric(5,2)` |
| Booleanos | `boolean NOT NULL DEFAULT` explícito |
| JSON | `jsonb` solo para estructuras sin consulta relacional (`custom_attributes`); si se filtra por dentro, es tabla |

## Índices y unicidad

Toda columna que el contrato permita filtrar u ordenar necesita índice. Toda regla de
unicidad del contrato (`LOCATION_CODE_TAKEN`) necesita **índice único en la base**, no solo
una comprobación en el servicio: la comprobación es para dar un 422 bonito, el índice es el
que garantiza la invariante bajo concurrencia.

```ts
@Entity('locations')
@Index(['code'], { unique: true })
export class Location {
  @PrimaryGeneratedColumn('uuid') id: string;
  @Column({ length: 120 }) name: string;
  @Column({ length: 20 }) code: string;
  @Column({ type: 'varchar', length: 16 }) type: LocationType;
  @Column({ name: 'manager_user_id', type: 'uuid', nullable: true }) managerUserId: string | null;
  @Column({ default: true }) active: boolean;
  @CreateDateColumn({ name: 'created_at', type: 'timestamptz' }) createdAt: Date;
}
```

## Relaciones

- FK con `ON DELETE RESTRICT` por defecto: el 409 del contrato
  (`LOCATION_HAS_ACTIVE_USERS`) debe poder cumplirse.
- Relaciones cargadas de forma explícita (`relations: [...]` o `QueryBuilder`); nada de
  `eager: true`, que arrastra datos que el contrato no pide.
- `@ManyToOne`/`@OneToMany` solo **dentro** del mismo módulo. Cruzar módulos es id + FK.

## Entidades de otros módulos

`User` y `Location` los consumen casi todos los módulos, pero **pertenecen a `parametrizacion`**.
Los demás módulos guardan `userId` / `locationId` con su FK en la base y consultan los datos
por el servicio que el módulo dueño exporta: no importan la entidad ajena ni la registran en su
`forFeature`, y no declaran `@ManyToOne` hacia ella. Extensiones de una entidad ajena: migración
aditiva **en su módulo dueño**, jamás una tabla paralela.

Las reglas completas, con ejemplos de lo prohibido y lo permitido, en
`06-modular-monolith.md`.

## Transacciones

Toda operación que escriba en más de una tabla va en transacción (`dataSource.transaction` o
`QueryRunner`). El mock no las necesita; el backend sí, y es exactamente el tipo de rigor
adicional que se espera respecto del mock.
