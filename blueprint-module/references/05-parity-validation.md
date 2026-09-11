# 05 — Validación de paridad contra el Blueprint

Objetivo: demostrar que el módulo nuevo **hace lo mismo**, no que se ve igual.

## Procedimiento

1. Levantar el Blueprint (`npm run dev` en `manufacturing-retail-erp-app`) y el frontend nuevo
   con mocks activos.
2. Recorrer `00-analysis.md` feature por feature (`F01..Fnn`) y comparar en vivo.
3. Registrar el veredicto de cada una en `04-parity.md`.

| Veredicto | Significado |
| --- | --- |
| `paridad` | comportamiento equivalente |
| `mejorada` | el nuevo hace más o mejor (estados, validación, accesibilidad). Anotar qué |
| `diferida` | pendiente, con motivo y dónde queda registrada |
| `descartada` | deliberadamente no se porta, con justificación aprobada por el usuario |

## Checklist por feature

- [ ] La pantalla existe y es alcanzable por la misma ruta lógica.
- [ ] Mismos datos visibles y mismas columnas relevantes.
- [ ] Mismas acciones disponibles, con las mismas confirmaciones.
- [ ] Mismas validaciones (y las del contexto del Blueprint, no solo las del formulario).
- [ ] Mismos efectos colaterales sobre otras entidades.
- [ ] Mismas reglas de permisos por rol.
- [ ] Filtros, búsqueda y orden equivalentes.
- [ ] Los cálculos dan **exactamente** el mismo resultado con los mismos datos (comparar con
      un caso concreto tomado de `src/data/*`, no "a ojo").
- [ ] Estados vacío / carga / error presentes (superan al Blueprint: se marca `mejorada`).
- [ ] Sin regresiones de accesibilidad ni teclado.

## Diferencias intencionales

Toda diferencia deliberada va a `03-decisions.md` con: qué hacía el Blueprint, qué hace el
nuevo, por qué, y quién lo aprobó. Si no está escrito, no está aprobado.

## Cierre

- `quality-gates` en verde.
- Registry actualizado (`validate: done`).
- Resumen final con `assets/summary-template.md`, incluyendo la lista de endpoints que el
  backend debe implementar para retirar los mocks.
