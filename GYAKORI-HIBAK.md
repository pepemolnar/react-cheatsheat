# Gyakori hibák: a saját feladataimból

Ezek a hibák a gyakorlás közben többször is előjöttek. Feladat leadása előtt érdemes végigfutni rajtuk.

## Szintaxis, ami lefut, de nem csinál semmit

| Ami a kódban áll | Mit csinál valójában | Helyesen |
| --- | --- | --- |
| `state.status === 'loading';` | összehasonlít, az eredmény elvész | `state.status = 'loading';` |
| `Math.min((x += 1), 6);` | a `+=` megtörtént, a min eredménye elvész | `x = Math.min(6, x + 1);` |
| `cond ? arr.push(a) : arr.splice(i, 1);` | működik, de lint hiba, és nem olvasható | `if (cond) … else …` |
| `state = initialState;` (reducerben) | csak a helyi paramétert állítja át | `return initialState;` |
| `(x) => { <li/> }` | nincs `return`: `undefined` | `(x) => (<li/>)` |
| `arr.reduce((line) => { line.a * line.b; })` | nincs gyűjtő, nincs `return`, nincs kezdőérték | `arr.reduce((sum, l) => sum + l.a * l.b, 0)` |
| `const exists = arr.map((x) => x.id === id);` | mindig tömb, és a tömb mindig igaz | `arr.find(...)` vagy `arr.some(...)` |
| `onClick={doIt()}` / `prop={fn()}` | renderelés közben lefut | `onClick={doIt}` / `onClick={() => doIt(x)}` |
| `const useX = useQuery({...})` (modul szint) | „Invalid hook call” | `function useX() { return useQuery({...}); }` |
| `const q = useX;` | a függvényt veszi át, nem hívja | `const q = useX();` |
| `startTransition(...)` a komponens törzsében | minden rendereléskor fut | eseménykezelőben |

## Típusok

- **`Record<string, number>` + `.length`:** a TS szerint `number`, valójában `undefined`. Darabszámhoz `Object.values(...).reduce(...)`.
- **`as` a hiányzó generikus helyett:** `fetchJson('/x') as T[]` helyett `fetchJson<T[]>('/x')`.
- **`PayloadAction<string>` + `as Category`:** a payload legyen pontos típusú.
- **Saját másolat egy közös típusból** (`type Category = 'rock' | …`): importáld a közöset, mert a másolat elavul.
- **Fölösleges import:** a `pnpm typecheck` hibát ad (`noUnusedLocals`).

## React

- **Hook egy korai `return` után:** a hookok mindig ugyanabban a sorrendben fussanak.
- **Hiányzó `key`** a `map`-ben, vagy nem a legkülső elemen.
- **Blokkelem `p`-ben vagy `span`-ben** (`<p>{children}</p>`, `<span><h3/></span>`).
- **A ref változtatása nem renderel újra:** ami látszik vagy effectet indít, az state.
- **`ignore = false` a cleanupban:** `true` kell.
- **Az `ignore` csak egy sort véd:** `if (!ignore) setA(); setB();`. Kell `if (ignore) return;` minden ág elején.
- **Az előző eredmény új kérés előtt nem törlődik:** az effect elején `setStatus('loading')`, `setError('')`.
- **A `queryFn` nem használja a kulcs paraméterét:** ami a `queryKey`-ben van, az a kérésben is legyen.
- **A `Suspense` a felfüggesztő komponens belsejében:** a szülőbe kell.
- **A `useFormStatus` abban a komponensben, amelyik a `<form>`-ot rendereli:** mindig `false`.
- **Error boundary `key` nélkül:** a régi hiba új paramétereknél is ott marad.
- **Az updater nélküli `useCallback([], …)`:** elavult lezárás, az első renderelés state-jét látja.
- **Soronként új nyílfüggvény `memo`-s gyereknek:** `onToggle={() => toggle(id)}` → a memo hatástalan.

## Megjelenés és feltételek

- **Elírt név átnevezés után** (`onlyFree` → `onlyAvailable`): VS Code-ban F2-vel nevezz át.
- **Egy elem eltűnik ahelyett, hogy tiltott lenne:** `disabled={isFetching}`, ne `{!isFetching && <button/>}`.
- **Beégetett szöveg a szerver üzenete helyett:** `error.message`.
- **Formázatlan érték:** `formatPrice`, `formatDate`.
- **A felirat pontossága:** idézőjel („…”), nagykötőjel (–), pont a mondat végén, kis- és nagybetű.
- **Határeset:** „legfeljebb” = `<=`.

## Leadás előtt

1. `pnpm typecheck`.
2. Konzol (F12): nincs piros hiba és figyelmeztetés (key, nesting, selector, uncached promise).
3. Az elfogadási feltételek egyenként végigkattintva, a határesetekkel együtt (0, max, üres, hiba, gyors váltás).
4. Fölösleges importok, elavult `TODO` és megjegyzések törölve.
