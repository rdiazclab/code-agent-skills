# `features/<mod>/<sub>/api` — endpoints, keys y hooks

```ts
// locations.api.ts
import { http, type Paginated } from '@/shared/api/http';
import type { Location, LocationFilters, LocationInput } from '../model';

// contract: PARAM-SEDES-E01
export const listLocations = (filters: LocationFilters) =>
  http.get<Paginated<Location>>('/locations', { query: filters });

// contract: PARAM-SEDES-E03
export const createLocation = (input: LocationInput) =>
  http.post<Location>('/locations', input);
```

```ts
// locations.queries.ts
export const locationKeys = {
  all: ['locations'] as const,
  lists: () => [...locationKeys.all, 'list'] as const,
  list: (f: LocationFilters) => [...locationKeys.lists(), f] as const,
  details: () => [...locationKeys.all, 'detail'] as const,
  detail: (id: string) => [...locationKeys.details(), id] as const,
};

export const useLocations = (filters: LocationFilters) =>
  useQuery({ queryKey: locationKeys.list(filters), queryFn: () => listLocations(filters) });

export const useCreateLocation = () => {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: createLocation,
    onSuccess: () => qc.invalidateQueries({ queryKey: locationKeys.lists() }),
  });
};
```

Los componentes consumen **solo** los hooks. Nadie fuera de `api/` importa `http`.
