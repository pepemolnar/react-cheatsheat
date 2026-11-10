# Űrlapok

## Kontrollált űrlap (klasszikus)

```tsx
type FormValues = { eventId: string; name: string; email: string; quantity: number };
type FormErrors = Partial<Record<keyof FormValues, string>>;

// a komponensen KÍVÜL: tiszta függvény, értékből hibát számol
function validate(values: FormValues): FormErrors {
  const errors: FormErrors = {};
  if (values.name.trim().length < 2) errors.name = 'A név legalább 2 karakter legyen.';
  if (!/^\S+@\S+\.\S+$/.test(values.email)) errors.email = 'Adj meg egy érvényes e-mail címet.';
  return errors;
}

export function OrderForm() {
  const [values, setValues] = useState<FormValues>({ eventId: 'e1', name: '', email: '', quantity: 1 });
  const [submitted, setSubmitted] = useState(false);          // volt-e küldési kísérlet
  const [confirmation, setConfirmation] = useState<string | null>(null); // sikeres volt-e

  const errors = validate(values);                            // renderelés közben, NEM state
  const hasErrors = Object.keys(errors).length > 0;

  // egy közös frissítő. Eseményváltáskor a darabszám visszaáll 1-re.
  const update = (key: keyof FormValues, value: string | number) =>
    setValues((prev) =>
      key === 'eventId' ? { ...prev, eventId: String(value), quantity: 1 } : { ...prev, [key]: value },
    );

  const submit = (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault();                                       // különben a böngésző újratölti az oldalt
    setSubmitted(true);
    if (hasErrors) return;
    setConfirmation(`Köszönjük, ${values.name}!`);
    setValues((prev) => ({ ...prev, name: '', email: '', quantity: 1 }));
    setSubmitted(false);
  };

  return (
    <form className="form" noValidate onSubmit={submit}>
      <label>
        Név <input value={values.name} onChange={(e) => update('name', e.target.value)} />
      </label>
      {submitted && errors.name && <p className="error">{errors.name}</p>}
      <select value={values.quantity} onChange={(e) => update('quantity', Number(e.target.value))}>
        {Array.from({ length: maxQuantity }, (_, i) => i + 1).map((n) => (
          <option key={n} value={n}>{n}</option>
        ))}
      </select>
      <button type="submit">Megrendelem</button>
    </form>
  );
}
```

- `<option>`-nek mindig legyen `value`-ja, különben az értéke a szövege lesz.
- `noValidate`: kikapcsolja a böngésző saját hibabuborékait.
- Legördülő lista a darabszámhoz: csak érvényes érték választható, nincs `NaN`.
- `e.target.value` mindig `string`. Számhoz: `Number(...)`. Típusos select-hez type guard:

```tsx
const isCategory = (v: string): v is Category => (CATEGORIES as readonly string[]).includes(v);
onChange={(e) => setCategory(isCategory(e.target.value) ? e.target.value : 'all')}
// az `as Category` csak „bízz bennem”, nem ellenőriz
```

## React 19: action és useActionState

```tsx
type State = { status: 'idle' } | { status: 'success'; message: string } | { status: 'error'; message: string; values: { name: string; eventId: string } };

// a komponensen kívül: előző state + FormData → új state
async function orderAction(_prev: State, formData: FormData): Promise<State> {
  const values = { name: String(formData.get('name') ?? ''), eventId: String(formData.get('eventId')) };
  try {
    const order = await sendJson<OrderResponse>('/api/orders', 'POST', values);
    return { status: 'success', message: `Sikeres: ${order.total}` };
  } catch (err) {
    return { status: 'error', message: (err as Error).message, values }; // a hibát állapotként adjuk vissza
  }
}

function OrderForm() {
  const [state, formAction, isPending] = useActionState(orderAction, { status: 'idle' });
  const values = state.status === 'error' ? state.values : undefined;

  return (
    <form action={formAction}>                                {/* nincs onSubmit, nincs preventDefault */}
      <input name="name" defaultValue={values?.name} />       {/* nem kontrollált: name + defaultValue */}
      <select name="eventId" defaultValue={values?.eventId ?? 'e1'} key={values?.eventId}>…</select>
      <button disabled={isPending}>{isPending ? 'Küldés…' : 'Megrendelem'}</button>
      {state.status === 'error' && <p className="error">Hiba: {state.message}</p>}
    </form>
  );
}
```

- Az action után a React **alaphelyzetbe állítja** a nem kontrollált mezőket. Hibánál ezért az értékeket a state-ben adjuk vissza, és `defaultValue` lesz belőlük.
- A `select` a `defaultValue`-t csak létrehozáskor veszi figyelembe: `key` kell, vagy maradjon kontrollált.
- Régi név: `useFormState` a `react-dom`-ból. Ma: `useActionState` a `react`-ból.

## useFormStatus

```tsx
import { useFormStatus } from 'react-dom';

// KÜLÖN komponens, a <form>-on BELÜL renderelve
function SubmitButton() {
  const { pending, data } = useFormStatus();
  return (
    <>
      <button type="submit" disabled={pending}>{pending ? 'Küldés…' : 'Küldés'}</button>
      {pending && <p>Küldés: {String(data?.get('email'))}</p>}
    </>
  );
}
```

Abban a komponensben, amelyik a `<form>`-ot rendereli, mindig `pending: false`. A hook felfelé keresi a legközelebbi `<form>`-ot.

## useOptimistic

```tsx
const [optimistic, addOptimistic] = useOptimistic(reviews, (current, item: Review) => [...current, item]);

const submitReview = async (formData: FormData) => {
  addOptimistic({ id: `temp-${Date.now()}`, text: String(formData.get('text')), pending: true });
  try {
    const created = await sendJson<Review>(url, 'POST', { … });
    startTransition(() => setReviews((r) => [...r, created])); // await UTÁN: transitionbe
  } catch (err) {
    setError((err as Error).message);                         // az optimista elem magától eltűnik
  }
};

<form action={submitReview}>…</form>
{optimistic.map((r) => <li key={r.id} className={r.pending ? 'pending' : undefined}>…</li>)}
```
