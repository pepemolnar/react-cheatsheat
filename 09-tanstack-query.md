# TanStack Query

## Bekötés

```tsx
const [queryClient] = useState(() => new QueryClient({
  defaultOptions: { queries: { retry: false } },   // alapból 3 újrapróbálás hibánál
}));

<QueryClientProvider client={queryClient}><App /></QueryClientProvider>
```

## useQuery

```tsx
const { data, isPending, isError, error, isFetching, refetch } = useQuery({
  queryKey: ['event', id],                                         // MINDEN, amitől a kérés függ
  queryFn: () => fetchJson<ConcertEvent>(`/api/events/${id}`),     // generikus típus → data típusos, nem kell as
  enabled: id !== null,                                            // amíg false, nem indul
  staleTime: 30_000,                                               // ennyi ideig friss (alap: 0)
});

if (id === null) return <p>Válassz.</p>;   // enabled: false mellett az isPending IGAZ, ezért ez legyen előbb
if (isPending) return <p>Betöltés…</p>;
if (isError) return <p className="error">Hiba: {error.message}</p>;
return <h3>{data.title} {isFetching && <span>(frissítés…)</span>}</h3>;   // itt a data már nem undefined
```

| Mező | Jelentése |
| --- | --- |
| `isPending` | még nincs adat ehhez a kulcshoz |
| `isFetching` | fut egy kérés (háttérfrissítésnél is) |
| `isError`, `error` | a `queryFn` dobott |
| `refetch()` | újrakéri |

- A `queryFn`-nek hibánál **dobnia** kell. A `fetch` 500-nál nem dob: kell a `res.ok` ellenőrzés (a `fetchJson` ezt teszi).
- A kulcs és a `queryFn` együtt mozog: ami a kulcsban van, az a kérésben is szerepeljen.
- A kulcsok **előtag szerint** is egyeznek: az `invalidateQueries({ queryKey: ['events'] })` az `['events', { fail }]`-et is érinti.
- Gomb háttérfrissítés alatt: maradjon látható, és legyen tiltott: `disabled={isFetching}`.
- `gcTime`: meddig marad a cache-ben a már nem használt adat (alap: 5 perc).

## Saját hook a közös lekérdezéshez

```ts
export const EVENTS_KEY = ['shared-events'] as const;

export function useEventsQuery() {                // FÜGGVÉNY, és a komponens hívja meg: useEventsQuery()
  return useQuery({
    queryKey: EVENTS_KEY,
    queryFn: () => fetchJson<ConcertEvent[]>('/api/events'),
    staleTime: 30_000,
  });
}
```

Ugyanaz a kulcs két komponensben = egy kérés (deduplikáció). A `const x = useQuery(…)` a modul szintjén „Invalid hook call”.

## Kulcsok egy helyen (query key factory)

```ts
export const eventKeys = {
  all: ['events'] as const,
  detail: (id: string) => ['events', 'detail', id] as const,
  reviews: (id: string) => ['events', 'reviews', id] as const,
};
```

## useMutation

```tsx
const queryClient = useQueryClient();

const order = useMutation({
  mutationFn: (request: OrderRequest) => sendJson<OrderResponse>('/api/orders', 'POST', request),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['events'] }),   // minden hívásnál
});

order.mutate(values, {
  onSuccess: () => resetForm(),            // csak ennél a hívásnál; nem fut le, ha a komponens közben eltűnt
});

<button disabled={order.isPending}>{order.isPending ? 'Küldés…' : 'Küldés'}</button>
{order.isError && <p className="error">Hiba: {order.error.message}</p>}
{order.isSuccess && <p>Összesen: {formatPrice(order.data.total)}</p>}   // a szerver válaszából
```

Siker és hiba: a mutáció saját állapota (`isSuccess`, `data`, `isError`, `error`), nem kell külön state.

## A cache frissítése mutáció után

```ts
// 1. invalidálás: újrakéri (a legbiztosabb)
queryClient.invalidateQueries({ queryKey: ['events'] });

// 2. a válasz beírása: nincs plusz kérés
onSuccess: (created) => {
  queryClient.setQueryData<Review[]>(['reviews', created.eventId], (old = []) => [...old, created]);
},

// 3. optimista frissítés
useMutation({
  mutationFn: …,
  onMutate: async (vars) => {
    await queryClient.cancelQueries({ queryKey: KEY });          // egy futó lekérdezés ne írja felül
    const previous = queryClient.getQueryData<string[]>(KEY);
    queryClient.setQueryData<string[]>(KEY, (old = []) => [...old, vars.id]);
    return { previous };                                        // az onError megkapja
  },
  onError: (_err, _vars, context) => queryClient.setQueryData(KEY, context?.previous),
  onSettled: () => queryClient.invalidateQueries({ queryKey: KEY }),  // a végén a szerver a mérvadó
});
```

Optimista frissítés: kis, gyakori, ritkán hibázó műveletekhez (kedvenc, lájk). Fizetéshez, rendeléshez nem.

## Suspense-szel

```tsx
const { data } = useSuspenseQuery({ queryKey: ['event', id], queryFn: … });   // data: T, nem T | undefined
// <Suspense> és error boundary kell köré; újrapróbáláshoz QueryErrorResetBoundary
```

## Redux vagy TanStack Query?

- **TanStack Query:** szerverállapot. A szerver a tulajdonosa, bármikor megváltozhat, nálad egy másolat van.
- **Redux / useState / context:** kliensállapot. Kosár, szűrők, nyitott panel, UI-beállítások.
