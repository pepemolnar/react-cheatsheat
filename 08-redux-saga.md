# Redux-saga

## Bekötés

```ts
import createSagaMiddleware from 'redux-saga';

const sagaMiddleware = createSagaMiddleware();

export const store = configureStore({
  reducer: { … },
  middleware: (getDefaultMiddleware) => getDefaultMiddleware({ thunk: false }).concat(sagaMiddleware),
});

sagaMiddleware.run(rootSaga);   // a store létrehozása UTÁN
```

## Worker és watcher

```ts
import type { PayloadAction } from '@reduxjs/toolkit';
import type { SagaIterator } from 'redux-saga';
import { all, call, cancel, delay, fork, put, race, select, take, takeLatest } from 'redux-saga/effects';

// worker: egy feladat
function* loadEvents(action: PayloadAction<{ fail: boolean }>): SagaIterator {
  try {
    const url = action.payload.fail ? '/api/events?fail=1' : '/api/events';
    const events: ConcertEvent[] = yield call(fetchJson<ConcertEvent[]>, url);  // függvény + paraméter, NEM fetchJson(url)
    yield put(eventsLoaded(events));
  } catch (err) {
    yield put(eventsFailed((err as Error).message));
  }
}

// watcherek
export function* rootSaga(): SagaIterator {
  yield all([
    takeLatest(eventsRequested.type, loadEvents),
    takeLatest(searchChanged.type, runSearch),
    takeLatest(watchStarted.type, watchAvailability),
  ]);
}
```

- A slice-ban csak sima actionök vannak (`eventsRequested`, `eventsLoaded`, `eventsFailed`). A kérést a saga indítja.
- A `SagaIterator` miatt a `yield` eredménye `any`: írd ki a változó típusát.
- **Le nem kezelt hiba a sagában leállítja a szülőt és az összes testvérét.** Minden workerben, és minden végtelen ciklusban `try`/`catch`.

## Effectek

| Effect | Mit csinál |
| --- | --- |
| `call(fn, ...args)` | meghív egy függvényt, a promise-t megvárja. Tesztelhető: csak leírja a hívást |
| `put(action)` | dispatch |
| `select(selector)` | kiolvas a store-ból |
| `takeEvery(type, worker)` | minden actionre új worker, a régiek futnak tovább |
| `takeLatest(type, worker)` | új actionnél megszakítja az előző, futó workert |
| `take(type)` | megvárja a következő ilyen actiont |
| `delay(ms)` | megszakítható várakozás |
| `race({ a, b })` | az első befejeződő nyer, a többit megszakítja |
| `fork(fn)` / `cancel(task)` | háttérfeladat indítása / leállítása |
| `all([...])` | párhuzamosan |
| `debounce(ms, type, worker)` | kész debounce (a futó kérést nem szakítja meg) |

Megszakításkor a worker a következő `yield`-nél megáll. A HTTP kérés nem szakad meg, de az eredménye nem jut el a `put`-ig.

## Debounce: takeLatest + delay

```ts
function* runSearch(action: PayloadAction<string>): SagaIterator {
  const q = action.payload.trim();
  if (!q) {
    yield put(searchCleared());
    return;
  }
  yield delay(400);                       // új leütésnél a takeLatest itt megszakítja
  yield put(searchStarted());
  try {
    const response: SearchResponse = yield call(fetchJson<SearchResponse>, `/api/search?q=${encodeURIComponent(q)}`);
    yield put(searchSucceeded(response));
  } catch (err) {
    yield put(searchFailed((err as Error).message));
  }
}
```

## Polling leállítással

```ts
function* pollAvailability(eventId: string): SagaIterator {
  while (true) {                          // a yield miatt nem fagy le
    try {
      const data: Availability = yield call(fetchJson<Availability>, `/api/events/${eventId}/availability`);
      yield put(availabilityUpdated(data));
    } catch (err) {
      console.error(err);                 // egy hiba ne állítsa le a teljes sagafát
    }
    yield delay(1000);
  }
}

function* watchAvailability(action: PayloadAction<string>): SagaIterator {
  yield race({
    poll: call(pollAvailability, action.payload),
    stop: take(watchStopped.type),        // ez nyer, és megszakítja a pollt
  });
}
// a rootSagában: takeLatest(watchStarted.type, watchAvailability) → új esemény = a régi figyelés leáll
```

Ugyanez `fork`/`cancel`-lel:

```ts
function* watchAvailability(action: PayloadAction<string>): SagaIterator {
  const task = yield fork(pollAvailability, action.payload);
  yield take(watchStopped.type);
  yield cancel(task);
}
```

```tsx
// komponens: indítás és leállítás, eltűnéskor is
useEffect(() => {
  if (!live) return;
  dispatch(watchStarted(eventId));
  return () => { dispatch(watchStopped()); };
}, [dispatch, live, eventId]);
```

## Thunk vagy saga?

| | thunk | saga |
| --- | --- | --- |
| egyszerű kérés | egyszerűbb | több kód |
| csak a legutolsó számít | kézzel (`abort`) | `takeLatest` |
| debounce, polling, versenyhelyzet | körülményes | erre való |
| tesztelés | mockolt fetch | a `call` effect összehasonlítható |
