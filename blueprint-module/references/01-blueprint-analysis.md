# 01 — Análisis del Blueprint

Cómo leer el frontend actual para extraer **funcionalidad**, no arquitectura.

## Regla previa

El Blueprint es de **solo lectura**. No se edita, no se refactoriza, no se arregla. Si tiene
un bug, se documenta como observación y se decide si el frontend nuevo replica o corrige el
comportamiento (queda anotado en `03-decisions.md`).

## Dónde vive cada cosa

| Qué buscas | Dónde |
| --- | --- |
| Pantallas del módulo | `src/components/<modulo>/` |
| Registro de la pantalla en la navegación | `src/App.tsx` (`case '<tab>':`) |
| Estado, acciones y reglas de negocio | `src/context/AppContext.tsx` (~4.7k líneas) |
| Entidades y campos | `src/types/index.ts`, `src/types/accounting.ts` |
| Datos semilla / catálogos reales | `src/data/*.ts` |
| Cálculos de negocio | `src/utils/*.ts` y funciones dentro del contexto |
| Permisos por rol | `src/data/rolesData.ts` + `hasPermission()` del contexto |

## Procedimiento

### 1. Delimitar el módulo

```bash
ls $BLUEPRINT/src/components/<modulo>/
grep -n "<ComponenteHub>" $BLUEPRINT/src/App.tsx
grep -n "activeSubView\|activeTab\|useState<'" $BLUEPRINT/src/components/<modulo>/*.tsx
```

Los submódulos suelen estar en un `useState` con una unión de literales en el componente hub.
Confirmar el listado con el usuario antes de invertir tiempo en el análisis completo.

### 2. Extraer el consumo del contexto

El bloque `const { ... } = useApp()` al inicio de cada componente es **el mejor resumen del
contrato de datos del módulo**: lo que lee y las acciones que dispara. Copiarlo entero y
clasificar cada símbolo en: `dato de lectura`, `mutación`, `derivado/calculado`, `UI`.

Después, para cada mutación, leer su implementación en `AppContext.tsx`
(`grep -n "const updateParametrization" src/context/AppContext.tsx`) y anotar:
validaciones, efectos colaterales sobre otras entidades, y cálculos. **Ahí están las reglas
de negocio**; la UI casi nunca las tiene.

### 3. Inventariar por submódulo

Para cada submódulo, rellenar en `00-analysis.md`:

- **Pantallas y layout**: lista, detalle, modal, wizard, tabs.
- **Acciones** con su disparador (botón, toggle, fila clicable) y confirmaciones.
- **Campos**: nombre, tipo, requerido, default, editable/solo-lectura, formato.
- **Validaciones**: las del formulario y las que hace la mutación en el contexto.
- **Filtros, búsqueda y orden**: qué columnas, cómo se combina.
- **Permisos**: qué esconde o deshabilita `hasPermission(...)` / `activeRole`.
- **Estados**: vacío, carga, error, éxito. En el Blueprint casi nunca existen carga/error
  porque todo es memoria — **eso es una carencia, no un requisito**: el módulo nuevo los
  necesita y se diseñan aquí.
- **Relaciones**: qué otras entidades referencia (p.ej. `User.locationId -> Location`).
- **Reglas de negocio inferidas**, cada una con cita `archivo:línea`.

### 4. Entidades

En `01-entities.md`, por entidad: campos con tipo, obligatoriedad, invariantes, relaciones,
ciclo de vida (creación, edición, baja lógica vs física) y de dónde salen los datos semilla.
Marcar qué campos son **de presentación** y no deben viajar en la API (p.ej. `badgeClass`,
`avatarUrl` generado, iconos) — error clásico al derivar contratos de un front.

### 5. Zonas ambiguas

Lo que no se pueda determinar leyendo el código va a una sección **Preguntas abiertas** del
análisis, no a una suposición silenciosa. En el checkpoint 1 se resuelven con el usuario.

## Señales de que estás copiando arquitectura (parar)

- Estás replicando el árbol de props o un contexto global gigante.
- Estás portando un componente de más de 300 líneas "tal cual".
- Estás copiando nombres de campos de presentación al contrato de API.
- Estás modelando la API alrededor de la forma del estado de React en vez de alrededor de los
  recursos del dominio.
