# Hibakeresés egy meglévő kódbázisban

## A sorrend

1. **Reprodukáld.** Pontosan mit kell csinálni, hogy előjöjjön? Mi a várt, és mi a tényleges eredmény?
2. **Konzol (F12 → Console).** Van piros hiba vagy figyelmeztetés? Az első sor a lényeg, a stack trace mutatja a fájlt.
3. **Network fül.** Elment a kérés? Mi az URL, a státusz (200/400/500), mi a válasz? `(canceled)` = megszakítva.
4. **React DevTools → Components.** A komponens propjai és state-je azok-e, amire számítasz?
5. **Redux DevTools.** Elment az action? A payload jó? A „Diff” fülön változott-e a state?
6. **Szűkítsd le.** Az adat rossz (store, szerver), vagy a megjelenítés (komponens)?
7. **Javítsd a legkisebb változtatással,** és ellenőrizd a környező eseteket is.

## Tünet → gyakori ok

| Tünet | Valószínű ok |
| --- | --- |
| a képernyő nem frissül | helyben módosított state (`push`, `obj.x = …`), ref a state helyett, rossz selector |
| fölösleges újrarenderelés | új objektum/tömb a selectorban vagy propban, új függvény propként, context `value={{…}}` |
| „Too many re-renders” | `setState` renderelés közben, `onClick={fn()}` |
| az effect végtelenül fut | objektum vagy függvény a függőségi listában, amely minden rendereléskor új |
| régi adat villan fel / rossz eredmény marad | versenyhelyzet: hiányzó `ignore`/`abort`/`takeLatest` |
| a lista egy pillanatra „0”-t mutat | `{count && …}` |
| „Each child in a list should have a unique key” | hiányzó vagy nem egyedi `key` |
| „cannot be a descendant of” | blokkelem `p`-ben vagy `span`-ben |
| „Cannot read properties of undefined” | adat még nincs meg (`isPending` ág hiányzik), `!` operátor, rossz kulcs |
| „Invalid hook call” | hook komponensen kívül, feltételben, vagy modul szinten |
| „Rendered more hooks than during the previous render” | hook egy korai `return` vagy `if` után |
| „A component was suspended by an uncached promise” | renderelés közben létrehozott promise a `use`-nak |
| a fetch hibánál is sikeresnek látszik | hiányzó `res.ok` ellenőrzés |
| az űrlap küldéskor újratölti az oldalt | hiányzó `e.preventDefault()` |
| a select rossz értéket ad | `<option>`-nek nincs `value`-ja |
| egy sagás funkció leállt, és több másik is | le nem kezelt hiba a sagában (hiányzó `try`/`catch`) |
| a mutáció után régi adat látszik | hiányzó `invalidateQueries` vagy `setQueryData` |
| az optimista érték visszaugrik | hiányzó `cancelQueries` az `onMutate`-ben |

## Feature fejlesztéskor

1. Keresd meg a **hasonló meglévő funkciót**, és kövesd a mintáját (mappák, elnevezés, store vagy query, stílus).
2. Hol legyen az állapot? Helyi `useState`, store, vagy szerver (TanStack Query)?
3. Típusok előbb: payload, válasz, props.
4. Az elfogadási feltételek lesznek a teszt-ellenőrzőlistád.
5. Kis lépések, minden lépés után a böngészőben ellenőrizve.
