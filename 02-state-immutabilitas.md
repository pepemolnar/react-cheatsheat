# State, események, immutábilis frissítés

## useState

```tsx
const [qty, setQty] = useState(1);
const [type, setType] = useState<'standing' | 'seated' | 'vip'>('standing'); // union: kell a generikus
const [items, setItems] = useState<CartItem[]>([]);                          // üres tömb: kell a generikus
const [stored] = useState(() => expensiveRead());                            // lusta kezdőérték: csak egyszer fut

setQty(5);                    // új érték
setQty((q) => q + 1);         // updater: ha az új érték az előzőből számolódik
```

- A `set` nem változtat azonnal: a változó ebben a renderelésben a régi marad.
- Az updater forma kell, ha egymás után többször hívod, intervallumban/időzítőben, vagy `useCallback([], …)` belsejében.
- Az updater paraméterének adj saját nevet (`q`), ne takarja el a state-et (`qty => qty + 1`).
- **Ami kiszámolható, az nem state:** összeg, szűrt lista, darabszám, hibák. Sima `const` renderelés közben.

## Eseménykezelők

```tsx
<button type="button" onClick={() => setQty((q) => q + 1)}>+</button>  // függvényt adsz át
<button onClick={handleClick}>…</button>                                // a függvény maga, zárójel NÉLKÜL
// HIBÁS: onClick={setQty(qty + 1)} → renderelés közben lefut → „Too many re-renders”

<input value={text} onChange={(e) => setText(e.target.value)} />
<input type="checkbox" checked={on} onChange={(e) => setOn(e.target.checked)} />  // checked, nem value
<input type="range" value={price} onChange={(e) => setPrice(Number(e.target.value))} />
```

Függvény átadása propként, paraméterrel:

```tsx
// szülő: a függvény maga
<SeatSelector onToggle={toggleSeat} />
// gyerek: itt hívódik meg, kattintáskor
<button onClick={() => onToggle(seat)}>{seat}</button>
```

A prop neve `onValami` (mikor hívják), a szülőben a függvény neve azt mondja meg, mit csinál (`toggleSeat`).

## Tömbök immutábilisan

| Művelet | Helyben (TILOS state-en) | Új tömbbel |
| --- | --- | --- |
| hozzáadás | `push` | `[...arr, item]` |
| törlés | `splice` | `arr.filter((x) => x.id !== id)` |
| egy elem módosítása | `arr[i].qty += 1` | `arr.map((x) => (x.id === id ? { ...x, qty: x.qty + 1 } : x))` |
| rendezés | `sort` | `[...arr].sort(…)` |
| kapcsolás (van/nincs) | | `arr.includes(id) ? arr.filter((x) => x !== id) : [...arr, id]` |

## Objektumok immutábilisan

```tsx
setForm((prev) => ({ ...prev, name: 'Péter' }));         // egy mező
setForm((prev) => ({ ...prev, [key]: value }));          // dinamikus kulcs
setCart((c) => ({ ...c, [id]: (c[id] ?? 0) + 1 }));       // Record<string, number> darabszámokhoz
setCart((c) => { const next = { ...c }; delete next[id]; return next; }); // kulcs kivétele: másolaton
```

## Határértékek

```tsx
const next = Math.min(MAX, Math.max(MIN, value + delta));  // a Math.min/max ÚJ értéket AD, nem módosít
x = Math.min(6, x + 1);                                    // az eredményt értékül kell adni
// HIBÁS: Math.min((x += 1), 6)  → a += már megtörtént, a min eredménye elvész
```

## Döntés az updaterben

Ha a döntés a state-től függ (van-e már a kosárban?), a döntés is az updaterbe kerüljön:

```tsx
setCart((prev) => {
  const existing = prev.find((x) => x.id === event.id);   // find: elem vagy undefined. NEM map (az mindig tömb → mindig igaz)
  if (existing) return prev.map((x) => (x.id === event.id ? { ...x, qty: Math.min(6, x.qty + 1) } : x));
  return [...prev, { id: event.id, qty: 1 }];
});
```

## Konstansok a komponensen kívül

```tsx
const MAX_TICKETS = 6;
const SEAT_MULTIPLIER: Record<SeatType, number> = { standing: 1, seated: 1.5, vip: 3 }; // a TS szól, ha egy típus hiányzik

function Card() { … }
```
