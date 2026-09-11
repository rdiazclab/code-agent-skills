# 01 — Leer el contrato (y el mock como especificación ejecutable)

## Por qué el mock manda

`02-contracts.md` describe el contrato; los handlers MSW lo **ejecutan**. El frontend está
probado contra los handlers, no contra el markdown. Donde ambos discrepen en un detalle de
forma, el handler gana y el markdown se corrige (vía `blueprint-module`, no aquí).

## Qué extraer de cada fuente

### De `02-contracts.md`
Tabla resumen (el inventario de trabajo), y por endpoint: método, ruta, params, permiso,
códigos y catálogo de errores con su `code`. También las **notas de derivación**, que explican
por qué un campo del Blueprint cambió de forma (p.ej. `manager: string` → `managerUserId`).

### De los handlers MSW
Lo que el markdown suele omitir y el frontend sí depende:

- **Forma literal de la respuesta**: nombres exactos, anidamiento, qué va `null` vs. ausente.
- **Campos generados en servidor**: `id`, `createdAt`, defaults (`active: true` al crear).
- **Orden por defecto** de las colecciones.
- **Qué se filtra con `q`** y sobre qué columnas.
- **Reglas que disparan 409/422** y su `code` exacto (`LOCATION_CODE_TAKEN`).
- **Qué hace realmente el `PATCH`**: merge parcial, campos inmutables.

```bash
# lectura obligatoria antes de planificar
cat $FRONT/src/mocks/handlers/$MODULE/*.handlers.ts
cat $FRONT/src/mocks/data/$MODULE/*.fixtures.ts
grep -rn "contract: $MODULE" $FRONT/src
```

### De `00-analysis.md`
Las reglas `Rnn` con su evidencia. Son las que van al **servicio**. El mock a veces las
simplifica (un mock no gestiona transacciones ni concurrencia); el backend las implementa
completas. Esa es la única dirección en la que el backend puede ir *más allá* del mock:
más rigor, nunca otra forma.

### De las fixtures
Los datos con los que ya se probó el frontend. Se reutilizan como semilla de los e2e para que
los casos de prueba de ambos lados hablen de las mismas sedes, usuarios y roles.

## Salida del modo `plan`

Por cada endpoint: tabla de campos (columna, tipo SQL, nulabilidad, default, índice), reglas
que le aplican, errores que puede devolver y de dónde sale cada dato. Lo que no se pueda
resolver leyendo, va a **preguntas abiertas** — no a una suposición dentro del código.

## Señales de alarma

- El contrato expone un campo que en el mock es de presentación (`badgeClass`) → §7.3.
- Un endpoint del contrato no tiene handler → nadie lo consume todavía: confirmar si se
  implementa ahora o se difiere.
- Un handler hace algo que ningún endpoint del contrato documenta → el contrato está
  incompleto: corregirlo antes de implementar.
