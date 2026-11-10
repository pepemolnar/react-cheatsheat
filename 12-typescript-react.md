# TypeScript Reacthez

## Gyakori típusok

```tsx
import type { ReactNode, FormEvent, ChangeEvent, Dispatch, SetStateAction, ComponentProps } from 'react';

children: ReactNode
onClick: () => void
onSelect: (id: string) => void
onSubmit={(e: FormEvent<HTMLFormElement>) => …}
onChange={(e: ChangeEvent<HTMLInputElement>) => …}
setValue: Dispatch<SetStateAction<T>>                  // a useState beállítója
type ButtonProps = ComponentProps<'button'> & { variant: 'primary' | 'ghost' };  // natív propok + saját
useRef<HTMLInputElement>(null)
useRef<number | null>(null)
```

## Union, Record, Partial

```ts
type Status = 'idle' | 'loading' | 'succeeded' | 'failed';
type Category = 'rock' | 'jazz' | 'klasszikus' | 'pop';

const LABELS: Record<Category, string> = { rock: 'Rock', jazz: 'Jazz', klasszikus: 'Klasszikus', pop: 'Pop' };
type Errors = Partial<Record<keyof FormValues, string>>;   // minden mező opcionális

// HIBÁS: { [k: Category]: X } → index signature nem lehet literál unió. Helyette Record<Category, X>
```

## Megkülönböztetett unió (discriminated union)

```ts
type State =
  | { status: 'idle' }
  | { status: 'success'; message: string }
  | { status: 'error'; message: string; values: FormValues };

if (state.status === 'error') state.values;   // itt a TS tudja, hogy a values létezik
```

## Type guard az `as` helyett

```ts
const isCategory = (v: string): v is Category => (CATEGORIES as readonly string[]).includes(v);

setCategory(isCategory(e.target.value) ? e.target.value : 'all');   // ellenőriz
setCategory(e.target.value as Category);                            // csak állít: egy hibás érték is átmegy
```

## Generikus függvény és hook

```ts
async function fetchJson<T>(url: string, init?: RequestInit): Promise<T> { … }
fetchJson<ConcertEvent[]>('/api/events');          // típusparaméter nélkül: unknown → as kellene

function useDebouncedValue<T>(value: T, delay: number): T { … }
```

## as const

```ts
export const EVENTS_KEY = ['shared-events'] as const;              // readonly ['shared-events']
export const CATEGORIES = ['rock', 'jazz'] as const;
type Cat = (typeof CATEGORIES)[number];                            // 'rock' | 'jazz'
```

## Nem null állítás és opcionális lánc

```ts
event!.title          // „biztosan létezik”: ha mégsem, futásidejű hiba. Inkább ellenőrizd.
event?.title ?? id    // ha nincs, tartalék érték
AVAILABLE[0]?.id      // tömbindexnél a TS alapból nem jelez (noUncheckedIndexedAccess)
```

## Szűkítés után nem kell `?.`

```tsx
if (query.isPending) return …;
if (query.isError) return …;
query.data.map(…)      // itt a data már nem undefined
```

## Gyakori fordítási hibák

| Hibaüzenet | Ok |
| --- | --- |
| `'x' is declared but its value is never read` | fölösleges import vagy változó (`noUnusedLocals`) |
| `Parameter 'state' implicitly has an 'any' type` | hiányzik a `RootState` típus, vagy sima `useSelector` |
| `Type 'void' is not assignable to type 'ReactNode'` | `() => { <JSX/> }`: kapcsos zárójel, `return` nélkül |
| `This syntax is not allowed when 'erasableSyntaxOnly' is enabled` | `constructor(private x)` vagy `enum`: írd ki a mezőt, enum helyett union |
| `Cannot find name 'x'` | hiányzó import |
