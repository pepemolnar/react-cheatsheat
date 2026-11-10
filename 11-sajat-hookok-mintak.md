# Saját hookok és komponensminták

## Saját hook szabályai

- `use` előtaggal kezdődik, és más hookokat hív. Nem renderel semmit.
- **A logikát osztja meg, nem az adatot:** két komponensben két külön state.
- Egy hook egy felelősség, a bemenetét **paraméterként** kapja, nem maga hozza létre.
- A hookok szabályai rá is vonatkoznak: legfelső szinten, feltétel nélkül, és csak komponensből vagy hookból hívható. A modul szintjén hívva: „Invalid hook call”.
- Az API legyen ismerős: ha `useState`-szerű, adjon `[érték, beállító]` párt.

## useDebouncedValue

```ts
export function useDebouncedValue<T>(value: T, delay: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);              // új érték előtt a régi időzítő törlődik
  }, [value, delay]);

  return debounced;
}

// használat: a state a komponensben marad
const [query, setQuery] = useState('');
const debouncedQuery = useDebouncedValue(query, 300);
```

## useLocalStorageState

```ts
export function useLocalStorageState<T>(key: string, initialValue: T): [T, Dispatch<SetStateAction<T>>] {
  const [value, setValue] = useState<T>(() => {           // lusta: csak az első rendereléskor olvas
    try {
      const stored = localStorage.getItem(key);
      return stored === null ? initialValue : (JSON.parse(stored) as T);
    } catch {
      return initialValue;                                // sérült JSON
    }
  });

  useEffect(() => {
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch {
      // privát mód, betelt tárhely: az alkalmazás működjön tovább
    }
  }, [key, value]);

  return [value, setValue];                               // a useState beállítója: az updatert is tudja
}
```

## Külső forrásra feliratkozás: useSyncExternalStore

```ts
export function useOnlineStatus(): boolean {
  return useSyncExternalStore(
    (onChange) => {
      window.addEventListener('online', onChange);
      window.addEventListener('offline', onChange);
      return () => {
        window.removeEventListener('online', onChange);
        window.removeEventListener('offline', onChange);
      };
    },
    () => navigator.onLine,                               // a pillanatnyi érték
  );
}
```

## Összetett komponens (compound component)

```tsx
type TabsContextValue = { selected: string; select: (v: string) => void; baseId: string };
const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext(part: string) {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error(`A Tabs.${part} csak Tabs-on belül használható`);
  return ctx;
}

function TabsRoot({ defaultValue, children }: { defaultValue: string; children: ReactNode }) {
  const [selected, setSelected] = useState(defaultValue);
  const baseId = useId();                                 // oldalszinten egyedi azonosító
  return <TabsContext value={{ selected, select: setSelected, baseId }}>{children}</TabsContext>;
}

function List({ children }: { children: ReactNode }) {
  return <div role="tablist">{children}</div>;
}

function Tab({ value, children }: { value: string; children: ReactNode }) {
  const { selected, select, baseId } = useTabsContext('Tab');
  return (
    <button role="tab" id={`${baseId}-tab-${value}`} aria-selected={selected === value}
      aria-controls={`${baseId}-panel-${value}`} onClick={() => select(value)}>
      {children}
    </button>
  );
}

function Panel({ value, children }: { value: string; children: ReactNode }) {
  const { selected, baseId } = useTabsContext('Panel');
  if (selected !== value) return null;
  return <div role="tabpanel" id={`${baseId}-panel-${value}`} aria-labelledby={`${baseId}-tab-${value}`}>{children}</div>;
}

export const Tabs = Object.assign(TabsRoot, { List, Tab, Panel });

// használat
<Tabs defaultValue="details">
  <Tabs.List>
    <Tabs.Tab value="details">Részletek</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel value="details">…</Tabs.Panel>
</Tabs>
```

Mikor jobb, mint a propokkal konfigurált (`<Tabs tabs={[…]} />`): ha a felhasználónak szabadon kell összeraknia a részeket, vagy egy részt máshová akar tenni a JSX-ben.

## Kontrollált és nem kontrollált komponens

```tsx
<Toggle defaultOn={false} />                         // nem kontrollált: saját state
<Toggle on={on} onChange={setOn} />                  // kontrollált: a szülő a tulajdonos
```

## key mint újraindítás

```tsx
<ProfileForm key={userId} userId={userId} />         // új userId → új komponens, tiszta state
```
