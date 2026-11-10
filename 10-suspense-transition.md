# Suspense, use(), error boundary, transition

## Suspense és use

```tsx
function EventDetails({ eventId }: { eventId: string }) {
  const event = use(getEventPromise(eventId));   // felfüggeszt, amíg a promise függőben van
  return <h3>{event.title}</h3>;                 // nincs null-ellenőrzés, nincs useEffect
}

// a Suspense a felfüggesztő komponens FELETT, a szülőben
<Suspense fallback={<p>Betöltés…</p>}>
  <EventDetails eventId={eventId} />
</Suspense>
```

- A `Suspense`, az error boundary és a provider mindig **kívül** van: a React felfelé keresi őket. A felfüggesztő komponens saját `return`-jében lévő `<Suspense>` meg sem születik.
- **A promise-nak stabilnak kell lennie.** A `use(fetch(url))` minden rendereléskor új promise-t hoz létre: végtelen ciklus. Figyelmeztetés: „A component was suspended by an uncached promise”.
- Éles kódban a promise-t könyvtár vagy keretrendszer adja: `useSuspenseQuery` (TanStack Query), Next.js Server Component propként, React Router `loader`.
- A `use` a contextet is tudja olvasni, akár feltételesen is: `use(ThemeContext)`.

## Error boundary

Csak osztálykomponens lehet (vagy a `react-error-boundary` csomag).

```tsx
class ErrorBoundary extends Component<Props, { error: Error | null }> {
  state = { error: null as Error | null };
  static getDerivedStateFromError(error: Error) {
    return { error };
  }
  reset = () => {
    this.props.onReset?.();
    this.setState({ error: null });
  };
  render() {
    if (this.state.error) return this.props.fallback(this.state.error, this.reset);
    return this.props.children;
  }
}

<ErrorBoundary
  key={url}                                       // új paraméternél új boundary: a régi hiba eltűnik
  onReset={() => invalidate(url)}                 // a hibás promise törlése újrapróbálás előtt
  fallback={(error, reset) => (                   // ZÁRÓJEL, nem kapcsos: vissza kell adnia a JSX-et
    <div role="alert">
      <p className="error">Hiba: {error.message}</p>
      <button onClick={reset}>Újra</button>
    </div>
  )}
>
  <Suspense fallback={<p>Betöltés…</p>}>          {/* a Suspense a boundaryn BELÜL */}
    <EventSummary url={url} />
  </Suspense>
</ErrorBoundary>
```

Az error boundary nem kapja el: az eseménykezelők hibáit, az aszinkron kód hibáit (`setTimeout`, `fetch.then`), és a saját hibáját. Ezeket `try`/`catch` és state kezeli.

## useTransition

```tsx
const [isPending, startTransition] = useTransition();

const selectTab = (next: Tab) => {
  startTransition(() => setTab(next));          // ESEMÉNYKEZELŐBEN, nem a komponens törzsében
};

<div className="tabs">
  {/* … gombok … */}
  {isPending && <span>Betöltés…</span>}
</div>
<div className={isPending ? 'stale' : undefined}>
  <Suspense fallback={<p>Fül betöltése…</p>}>   {/* a fallback állandó: csak a legelső betöltéskor látszik */}
    {tab === 'reviews' && <ReviewsTab />}
  </Suspense>
</div>
```

- Ha egy transition miatt felfüggeszt egy komponens, a React **nem** mutatja a fallbacket, hanem a régi tartalmat hagyja kint.
- A `startTransition`-ön belüli frissítés nem sürgős. Egy közben jövő sürgős frissítés (gépelés) megelőzi.
- Az `aria-selected` a transition alatt még a régi fület mutatja. Ha a gombnak azonnal váltania kell: két state, egy sürgős és egy transitionben frissülő.

## useDeferredValue

Ha az érték nem tőled jön (prop, beírt szöveg), hanem csak a lassú részt akarod késleltetni:

```tsx
const deferredQuery = useDeferredValue(query);
const isStale = query !== deferredQuery;

<input value={query} onChange={(e) => setQuery(e.target.value)} />   {/* a mező azonnal frissül */}
<div className={isStale ? 'stale' : undefined}>
  <SlowList query={deferredQuery} />                                  {/* memo-val együtt hatékony */}
</div>
```

| | `useTransition` | `useDeferredValue` |
| --- | --- | --- |
| mit késleltet | egy state-frissítést, amit te indítasz | egy értéket, amit kapsz |
| jelző | `isPending` | `value !== deferredValue` |
