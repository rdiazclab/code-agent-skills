# `src/test/setup.ts` y render con providers

```ts
// setup.ts
import { server } from '@/mocks/server';
import { resetDb } from '@/mocks/lib/db';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
beforeEach(() => resetDb());
afterEach(() => { server.resetHandlers(); cleanup(); });
afterAll(() => server.close());
```

```tsx
// renderWithProviders.tsx
export function renderWithProviders(ui: ReactElement, { route = '/' } = {}) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false, gcTime: 0 } },   // sin reintentos en test
  });
  return {
    user: userEvent.setup(),
    ...render(ui, { wrapper: ({ children }) => (
      <QueryClientProvider client={queryClient}>
        <MemoryRouter initialEntries={[route]}>{children}</MemoryRouter>
      </QueryClientProvider>
    )}),
  };
}
```

Test tipo, con el ID de feature en el nombre:

```tsx
it('PARAM-SEDES-F01: muestra el estado vacío cuando no hay sedes', async () => {
  server.use(http.get('/api/v1/locations', () =>
    HttpResponse.json({ data: [], meta: { page: 1, pageSize: 25, total: 0, totalPages: 0 } })));
  renderWithProviders(<LocationsPage />);
  expect(await screen.findByText(/aún no hay sedes/i)).toBeInTheDocument();
});
```
