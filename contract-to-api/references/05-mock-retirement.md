# 05 — Retirada del mock

Apagar el mock es un acto explícito y verificado, no un efecto colateral.

## Precondiciones (todas)

- [ ] e2e de conformidad en verde para **todos** los endpoints del módulo.
- [ ] `quality-gates` en verde en ambos repos.
- [ ] Migraciones aplicadas y `down()` verificado.
- [ ] `conformance.md` escrito, sin diferencias abiertas.

## Procedimiento

1. Frontend: `VITE_API_MOCKS=false` y `VITE_API_URL` al backend real. Ninguna otra línea de
   código de producto cambia — si hiciera falta tocar un componente, la capa de API estaba mal
   construida y eso se arregla antes.
2. Smoke manual de los flujos principales del módulo contra la API real.
3. **Borrar** los handlers MSW sustituidos, no comentarlos. Limpiar el barril del submódulo; si
   queda vacío, borrarlo también.
4. Fixtures: conservarlas si algún test las usa; borrarlas si quedan huérfanas.
5. Registry del frontend: esos endpoints pasan a `live`.
6. Los tests del frontend **siguen usando MSW**: pero ahora sus handlers deben reflejar lo que
   hace la API real. Si se borra un handler que un test usaba, el test se recablea con
   `server.use(...)` local o con un handler mínimo de test.

## Retirada parcial

Es normal que un módulo tenga endpoints vivos y otros mockeados a la vez. MSW solo intercepta
lo que tiene handler: lo demás pasa de largo al backend real. Por eso los handlers se borran
**uno a uno** según se implementan, y el registry lleva el estado por endpoint, no por módulo.

## Vuelta atrás

Si algo falla en real, volver a `VITE_API_MOCKS=true` no basta si los handlers ya se borraron.
Por eso: se borran solo tras el smoke, y el commit que los borra es independiente del que
cambia la configuración — revertirlo restituye el mock en un paso.
