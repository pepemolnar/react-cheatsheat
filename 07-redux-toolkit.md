# Redux Toolkit

## Store és típusos hookok

```ts
// store.ts
import { configureStore } from '@reduxjs/toolkit';

export const store = configureStore({
  reducer: { cart: cartReducer, filters: filtersReducer },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// hooks.ts
import { useDispatch, useSelector } from 'react-redux';
export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();

// main.tsx / App
<Provider store={store}><App /></Provider>
```

Egy selector `state` paraméterére „implicitly has an 'any' type” hiba: a sima `useSelector`-t használod, vagy a selectornak nincs `(state: RootState)` típusa.

## Slice

```ts
import { createSlice, type PayloadAction } from '@reduxjs/toolkit';

type CartState = { items: Record<string, number> };       // azonosító → darabszám
const initialState: CartState = { items: {} };

const cartSlice = createSlice({
  name: 'cart',                                          // action típus: 'cart/addToCart'
  initialState,
  reducers: {
    addToCart(state, action: PayloadAction<string>) {
      const id = action.payload;
      state.items[id] = Math.min(6, (state.items[id] ?? 0) + 1);  // Immer: szabad „módosítani”
    },
    removeFromCart(state, action: PayloadAction<string>) {
      delete state.items[action.payload];
    },
    reset() {
      return initialState;                               // vagy új értéket adsz vissza
    },
  },
});

export const { addToCart, removeFromCart, reset } = cartSlice.actions;
export const cartReducer = cartSlice.reducer;
```

**Immer szabályai:**
- Vagy **módosítod** a piszkozat mezőit, vagy **visszaadsz** egy új értéket. A kettőt egyszerre nem.
- A `state = initialState` **semmit sem csinál**: csak a helyi paramétert állítja át.
- Egy zárójel nélküli nyílfüggvény a kifejezés értékét vissza is adja: a `(state) => state.count++` módosít ÉS visszaad egy számot, amire az Immer hibát dob („returned a new value *and* modified its draft”). Reducerben használj kapcsos zárójelet.
- A payload típusa legyen pontos: `PayloadAction<Category | 'all'>`, ne `PayloadAction<string>` és `as`. Az action creator típusa ebből jön, így minden `dispatch` ellenőrzött.
- Mellékhatáshoz `if`/`else`, ne ternary: `cond ? state.ids.push(id) : state.ids.splice(i, 1)` működik, de lint hiba.

## Komponensben

```tsx
const dispatch = useAppDispatch();
const count = useAppSelector((state) => state.cart.items[id] ?? 0);

<button onClick={() => dispatch(addToCart(event.id))}>Kosárba</button>
```

A származtatott érték (összeg, szűrt lista) nem kerül a store-ba: selectorban vagy renderelés közben számolódik.

## Selectorok és újrarenderelés

Minden dispatch után **minden** selector lefut, és az eredményt `===`-vel hasonlítja az előzőhöz. Ha eltér, a komponens újrarenderel.

```ts
// sima selector: meglévő state-darab vagy primitív → stabil
export const selectCartItems = (state: RootState) => state.cart.items;
export const selectCartCount = (state: RootState) =>
  Object.values(state.cart.items).reduce((sum, q) => sum + q, 0);           // szám → stabil
export const selectCartTotal = (state: RootState) =>
  Object.entries(state.cart.items).reduce((sum, [id, q]) => {
    const event = state.events.find((e) => e.id === id);
    return sum + (event ? event.price * q : 0);
  }, 0);

// új tömb/objektum → createSelector
import { createSelector } from '@reduxjs/toolkit';
export const selectVisibleEvents = createSelector(
  [selectEvents, selectCategory],                                          // bemenetek
  (events, category) => (category === 'all' ? events : events.filter((e) => e.category === category)),
);

// paraméteres, primitív eredmény → sima függvény, nyílfüggvénnyel hívva
export const selectIsFavorite = (state: RootState, id: string) => state.favorites.ids.includes(id);
const isFavorite = useAppSelector((state) => selectIsFavorite(state, eventId));
```

| A selector visszatérési értéke | Megoldás |
| --- | --- |
| meglévő state-darab (`state.events`) | sima |
| primitív (szám, boolean, szöveg) | sima, akkor is, ha közben `Object.values`-t használ |
| `filter`, `map`, `{ … }`, `Object.keys` eredménye | `createSelector`, vagy bontás primitívekre, vagy `useAppSelector(fn, shallowEqual)` |
| drága számítás, primitív eredménnyel | `createSelector` |

- `Record<string, number>`-en a `.length` nem hiba a TS szerint, de `undefined`. Darabszámhoz `Object.keys(...).length`.
- Egy komponens csak azt olvassa ki, amire szüksége van. Ha a szülő olvassa ki a listát, és a gyerekeknek adja, minden gyerek újrarenderel.
- A paraméteres `createSelector` csak az utolsó bemenetet jegyzi meg: sok komponensben különböző paraméterrel hatástalan.
- Fejlesztői figyelmeztetés: „Selector unknown returned a different result when called with the same parameters” → a selector új referenciát ad.

## createAsyncThunk

```ts
export const fetchEvents = createAsyncThunk<
  ConcertEvent[],              // fulfilled payload
  { fail: boolean },           // paraméter
  { rejectValue: string }      // rejected payload
>('events/fetch', async ({ fail }, { rejectWithValue, signal }) => {
  try {
    return await fetchJson<ConcertEvent[]>(fail ? '/api/events?fail=1' : '/api/events', { signal });
  } catch (err) {
    return rejectWithValue((err as Error).message);
  }
});

// a slice-ban
extraReducers: (builder) => {
  builder
    .addCase(fetchEvents.pending, (state) => { state.status = 'loading'; state.error = null; })
    .addCase(fetchEvents.fulfilled, (state, action) => { state.status = 'succeeded'; state.items = action.payload; })
    .addCase(fetchEvents.rejected, (state, action) => {
      if (action.meta.aborted) return;                    // megszakítás nem hiba
      state.status = 'failed';
      state.error = action.payload ?? 'Ismeretlen hiba';
    })
    .addCase(placeOrder.fulfilled, (state, action) => {   // egy MÁSIK slice thunkjára is reagálhat
      const event = state.items.find((e) => e.id === action.payload.eventId);
      if (event) event.remaining -= action.payload.quantity;
    });
},
```

```tsx
// komponensben
useEffect(() => {
  dispatch(fetchEvents({ fail: simulateError }));
}, [dispatch, simulateError]);

// megszakítható
useEffect(() => {
  const promise = dispatch(searchEvents(term));
  return () => promise.abort();
}, [dispatch, term]);

// eredmény ellenőrzése
const result = await dispatch(placeOrder(order));
if (placeOrder.fulfilled.match(result)) { /* űrlap ürítése */ }
```

`state.status === 'loading'` a reducerben összehasonlítás, nem értékadás: semmi sem változik.

**RTK Query:** a Redux Toolkit saját TanStack Query-szerű eszköze (`createApi`). Ha egy reduxos kódbázisban cache-elt szerveradat kell, gyakran ezt használják.

**Redux DevTools:** böngészőbővítmény, F12 → Redux fül. Az „Action” fül a payloadot, a „Diff” fül a state változását mutatja.
