# Effectek és useRef

## Adatbetöltés effectben (TanStack Query nélkül)

```tsx
const [status, setStatus] = useState<'loading' | 'error' | 'success'>('loading');
const [events, setEvents] = useState<ConcertEvent[]>([]);
const [error, setError] = useState('');
const [reloadKey, setReloadKey] = useState(0);              // az „Újra” gomb ezt növeli

useEffect(() => {
  let ignore = false;
  setStatus('loading');
  setError('');

  fetch(simulateError ? '/api/events?fail=1' : '/api/events')
    .then(async (res) => {
      if (!res.ok) {                                        // a fetch 500-nál NEM dob hibát
        const body = await res.json();
        throw new Error(body.message);
      }
      return res.json();
    })
    .then((data: ConcertEvent[]) => {
      if (ignore) return;                                   // minden state-állítás elé
      setEvents(data);
      setStatus('success');
    })
    .catch((err: Error) => {
      if (ignore) return;
      setError(err.message);
      setStatus('error');
    });

  return () => {
    ignore = true;                                          // TRUE: a régi választ eldobjuk
  };
}, [simulateError, reloadKey]);

<button onClick={() => setReloadKey((k) => k + 1)}>Újra</button>
```

- Sorrend új függőségnél: **előbb a régi effect cleanupja**, utána az új effect.
- StrictMode fejlesztéskor kétszer futtatja az effectet (mount, cleanup, mount). Ha ettől elromlik, hiányzik a cleanup.
- Kérés megszakítása `AbortController`-rel:

```tsx
useEffect(() => {
  const controller = new AbortController();
  fetch(url, { signal: controller.signal }).then(…).catch((err) => {
    if (err.name === 'AbortError') return;
    …
  });
  return () => controller.abort();
}, [url]);
```

- Keresés URL-ben: `encodeURIComponent(term)`. Üres mezőnél ne induljon kérés: `if (!term) return;`.

## Függőségi lista

```tsx
useEffect(() => { … }, []);          // egyszer, megjelenéskor
useEffect(() => { … }, [a, b]);      // amikor a vagy b változik
useEffect(() => { … });              // minden renderelés után (ritkán kell)
```

Amit az effect a komponens hatóköréből olvas (prop, state, függvény), az a listába kerül.

## useRef: DOM elem

```tsx
const inputRef = useRef<HTMLInputElement>(null);

useEffect(() => {
  inputRef.current?.focus();          // effectben: renderelés közben a ref még null
}, []);                               // egy külön, megnyitáskor létrejövő komponensben: []
                                      // ugyanabban a komponensben: [open] és if (open)
<input ref={inputRef} />
```

## useRef: érték, amely nem renderel újra

```tsx
const intervalRef = useRef<number | null>(null);

const start = () => {
  if (intervalRef.current !== null) return;                 // ne induljon kettő
  intervalRef.current = window.setInterval(() => {
    setLeft((l) => Math.max(0, l - 1));                     // updater: a callback a régi left-et látná
  }, 1000);
  setRunning(true);
};

const stop = () => {
  if (intervalRef.current !== null) clearInterval(intervalRef.current);
  intervalRef.current = null;
  setRunning(false);
};

useEffect(() => () => {                                     // eltűnéskor leállítás
  if (intervalRef.current !== null) clearInterval(intervalRef.current);
}, []);

useEffect(() => {
  if (left === 0) stop();                                   // reakció egy state-változásra
}, [left]);
```

| | `useState` | `useRef` |
| --- | --- | --- |
| a változás újrarenderel | igen | nem |
| az új érték mikor látszik | a következő renderelésben | azonnal (`.current`) |
| mire | ami látszik a képernyőn, vagy effectet indít | DOM elem, időzítő azonosító, előző érték |

`window.setInterval` → `number`. A sima `setInterval` típusa Node-os környezetben `Timeout` lehet.
