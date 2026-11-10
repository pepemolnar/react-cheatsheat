# Közös állapot: felemelés és context

## Állapot felemelése

Az állapot a legközelebbi közös szülőbe kerül. A szülő az értéket és egy módosító függvényt ad le propként.

```tsx
function SeatPicker() {
  const [selected, setSelected] = useState<string[]>([]);
  const toggleSeat = (seat: string) =>
    setSelected((prev) => (prev.includes(seat) ? prev.filter((s) => s !== seat) : [...prev, seat]));

  return (
    <>
      <SeatSelector selected={selected} onToggle={toggleSeat} />
      <Summary selected={selected} />
    </>
  );
}
```

## Context, alapértékkel

Ha van értelmes alapérték (téma, nyelv, pénznem):

```tsx
const CurrencyContext = createContext<Currency>('HUF');

// megadás (React 19-ben maga a context a provider; régen: <CurrencyContext.Provider value={…}>)
<CurrencyContext value={currency}>
  <EventPage />
</CurrencyContext>

// olvasás bármelyik leszármazottban
const currency = useContext(CurrencyContext);
```

## Context, null alapértékkel és saját hookkal

Ha provider nélkül értelmetlen (kosár, bejelentkezett felhasználó):

```tsx
type CartValue = { ticketCount: number; addTicket: () => void };

export const CartContext = createContext<CartValue | null>(null);

export function useCart(): CartValue {
  const value = useContext(CartContext);
  if (!value) throw new Error('A useCart csak CartContext-en belül használható');
  return value;
}

// a provider
function Shop() {
  const [ticketCount, setTicketCount] = useState(0);
  const addTicket = useCallback(() => setTicketCount((n) => n + 1), []);
  const value = useMemo(() => ({ ticketCount, addTicket }), [ticketCount, addTicket]); // stabil value

  return <CartContext value={value}>…</CartContext>;
}

// a fogyasztó: a hookot használja, NEM a nyers useContext-et
const { addTicket } = useCart();
```

- A `useContext(CartContext)` + `?.` csendben semmit sem csinál provider nélkül. A saját hook azonnal hibát dob.
- A `value={{ … }}` minden rendereléskor új objektum: **minden fogyasztó újrarenderel**. Nagyobb alkalmazásban `useMemo`.

## Mikor melyiket?

| Helyzet | Megoldás |
| --- | --- |
| 2–3 szint, néhány komponens | felemelés, propokkal |
| távoli komponensek, ritkán változó érték | context |
| gyakran változó, sok helyről módosított kliensállapot | Redux Toolkit (lásd 07) |
| szerveradat (cache, betöltés, frissítés) | TanStack Query (lásd 09) |
| egy komponens saját UI-állapota (mező, nyitott panel) | helyi `useState` |
