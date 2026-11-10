# Teljesítmény: memo, useCallback, useMemo

**Alapszabály:** ha egy komponens újrarenderel, az összes gyereke is újrarenderel, akkor is, ha a propjaik nem változtak.

**Összehasonlítás `===`-vel:** a primitív (szám, szöveg, boolean) érték szerint, az objektum, a tömb és a függvény **referencia** szerint hasonlítódik. `(() => {}) === (() => {})` hamis.

## memo: a gyerek oldalán

```tsx
const Seat = memo(function Seat({ id, isSelected, onToggle }: SeatProps) {
  return <button aria-pressed={isSelected} onClick={() => onToggle(id)}>{id}</button>;
});
```

Kihagyja a renderelést, ha **minden** prop `===` szerint egyezik.

## useCallback: stabil függvény a szülőben

```tsx
const toggle = useCallback((id: string) => {
  setSelected((prev) => {                     // updater: így nem kell a selected a függőségek közé
    const next = new Set(prev);
    if (next.has(id)) next.delete(id);
    else next.add(id);
    return next;
  });
}, []);                                       // a setState függvény mindig stabil, nem kell a listába
```

## A két gyakori csapda

```tsx
// 1. a teljes halmaz/tömb a prop: minden kattintásnál új referencia → mind újrarenderel
<Seat selected={selected} />                 // HIBÁS
<Seat isSelected={selected.has(id)} />       // JÓ: boolean

// 2. soronként új nyílfüggvény: a memo minden sorban új propot lát
<Row onToggle={() => toggle(event.id)} />    // HIBÁS
<Row onToggle={toggle} />                    // JÓ: a sor adja át az id-t: onClick={() => onToggle(event.id)}
```

**Elavult lezárás (stale closure):** egy `useCallback(fn, [])` örökre az első renderelés state-jét látja. Ezért kell benne az updater forma.

## useMemo: drága számítás

```tsx
const stats = useMemo(() => computeStats(SALES, category), [category]);
```

- Akkor érdemes, ha a számítás mérhetően drága (kb. 1 ms fölött), vagy az eredmény egy `memo`-s gyerek propja, és stabil referencia kell.
- Egy rövid lista `filter`-éhez nem kell.
- A `useMemo` callbackje legyen tiszta: StrictMode-ban kétszer is lefuthat.

## Melyik mire

| Eszköz | Mit őriz meg | Mikor segít |
| --- | --- | --- |
| `memo` | egy komponens renderelését | ha a propjai `===` szerint nem változtak |
| `useCallback` | egy függvénypéldányt | ha `memo`-s gyerek propja, vagy effect függősége |
| `useMemo` | egy kiszámolt értéket | ha drága, vagy `memo`-s gyerek propja |

`memo` nélkül a `useCallback` fölösleges. `useCallback` nélkül a `memo` mindig új függvényt talál.

**React Compiler:** egy külön Babel plugin (`babel-plugin-react-compiler`), amely fordításkor beilleszti a memoizálást. Csak ha be van kapcsolva a build konfigurációban. A legtöbb meglévő kódbázisban nincs, ott a kézi változat van.

**Mérés:** React DevTools. A beállításokban (fogaskerék) a „Highlight updates when components render” keretezi a renderelő komponenseket. A Profiler fül rögzíti, ki és miért renderelt.
