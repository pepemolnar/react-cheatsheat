# React cheat sheet

Gyors referencia a React alapozó (`react-projekt`) és haladó (`react-halado`) gyakorlás fejezeteihez. Minden fájlban a szintaxis, egy-egy rövid szabály és a tipikus csapdák. A kódrészletek a fejezetek ellenőrzött megoldásaiból valók.

## Tartalom

| Fájl | Téma | Fejezetek |
| --- | --- | --- |
| [01-jsx-props-listak.md](01-jsx-props-listak.md) | komponens, props, `children`, feltételes megjelenítés, `map` és `key`, HTML szabályok | alap 1–2 |
| [02-state-immutabilitas.md](02-state-immutabilitas.md) | `useState`, updater, eseménykezelők, tömbök és objektumok immutábilisan | alap 3–4 |
| [03-urlapok.md](03-urlapok.md) | kontrollált űrlap, validáció, `useActionState`, `useFormStatus`, `useOptimistic` | alap 5, haladó 7 |
| [04-kozos-allapot-context.md](04-kozos-allapot-context.md) | állapot felemelése, context, saját context hook | alap 6 |
| [05-effectek-ref.md](05-effectek-ref.md) | adatbetöltés effectben, cleanup, `ignore`, `AbortController`, `useRef` | alap 7 |
| [06-teljesitmeny.md](06-teljesitmeny.md) | `memo`, `useCallback`, `useMemo`, React Compiler | alap 8 |
| [07-redux-toolkit.md](07-redux-toolkit.md) | store, slice, Immer, selectorok, `createSelector`, `createAsyncThunk` | haladó 1–3 |
| [08-redux-saga.md](08-redux-saga.md) | effectek, `takeLatest`, debounce, polling, `race` | haladó 4 |
| [09-tanstack-query.md](09-tanstack-query.md) | `useQuery`, kulcsok, `staleTime`, `useMutation`, invalidálás, optimista frissítés | haladó 5–6 |
| [10-suspense-transition.md](10-suspense-transition.md) | `Suspense`, `use`, error boundary, `useTransition`, `useDeferredValue` | haladó 8 |
| [11-sajat-hookok-mintak.md](11-sajat-hookok-mintak.md) | saját hookok, `useSyncExternalStore`, compound component | haladó 9 |
| [12-typescript-react.md](12-typescript-react.md) | React típusok, union, `Record`, type guard, generikusok, fordítási hibák | mind |
| [GYAKORI-HIBAK.md](GYAKORI-HIBAK.md) | a saját feladataimban előjött hibák, és leadás előtti ellenőrzőlista | mind |
| [HIBAKERESES.md](HIBAKERESES.md) | hibakeresés lépésenként, tünet → ok táblázat | mind |

## Mikor mit?

| Kérdés | Válasz |
| --- | --- |
| Hol legyen ez az állapot? | egy komponens UI-állapota → `useState`; néhány testvér → felemelés; távoli, ritkán változó → context; közös kliensállapot → Redux; szerveradat → TanStack Query |
| Kiszámolható a meglévőkből? | akkor nem state, hanem `const` renderelés közben |
| Kell `useEffect`? | csak külső rendszerrel való szinkronhoz (kérés, időzítő, DOM, feliratkozás). Számításhoz és eseményre reagáláshoz nem |
| `useState` vagy `useRef`? | látszik a képernyőn vagy effectet indít → state; csak meg kell jegyezni → ref |
| Updater forma kell? | ha az új érték az előzőből számolódik |
| Sima selector vagy `createSelector`? | meglévő state-darab vagy primitív → sima; új tömb vagy objektum → `createSelector` |
| Thunk vagy saga? | egyszerű kérés → thunk; csak a legutolsó számít, debounce, polling → saga |
| Invalidálás, `setQueryData` vagy optimista? | alapból invalidálás; ha a válasz pontosan kell → `setQueryData`; kis, gyakori művelet → optimista |
| `memo`/`useCallback`/`useMemo`? | csak ha mérhetően gond van, vagy egy `memo`-s gyerek stabil propot vár |
| `useTransition` vagy `useDeferredValue`? | a frissítést te indítod → transition; az értéket kapod → deferred |

## Az öt legfontosabb szabály

1. **A state immutábilis.** Új tömb, új objektum. A Redux reducerben az Immer miatt szabad „módosítani”, de akkor ne adj vissza semmit.
2. **Minden összehasonlítás `===`.** A `memo`, a `useSelector`, a függőségi lista és a context is referencia szerint hasonlít.
3. **Függvényt adsz át, nem hívsz meg.** `onClick={fn}` vagy `onClick={() => fn(x)}`.
4. **Hookok: legfelső szinten, feltétel nélkül, komponensben.** Korai `return` csak utánuk.
5. **A hiba kezelése ott van, ahol a hiba van.** `res.ok`, `try`/`catch` a sagában, error boundary a felfüggesztő rész körül.
