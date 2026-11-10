# JSX, props, feltételes megjelenítés, listák

## Komponens és props

```tsx
import type { ReactNode } from 'react';

type CardProps = {
  title: string;
  description?: string;      // opcionális
  children: ReactNode;       // bármi, ami JSX-ben a nyitó és záró tag között van
  onSelect: (id: string) => void;
};

export function Card({ title, description, children, onSelect }: CardProps) {
  return (
    <section className="panel">
      <h2>{title}</h2>
      {description && <p className="muted">{description}</p>}
      {children}
    </section>
  );
}
```

- A komponens neve nagybetűs. A JSX egyetlen gyökérelemet ad vissza, vagy fragmentet: `<>…</>`.
- `className`, nem `class`. Az inline stílus objektum: `style={{ maxHeight: 360 }}`.
- A `children`-t ne csomagold `<p>`-be: ha a tartalom blokkelem (`div`, `article`, `h3`), az érvénytelen HTML.

## Feltételes megjelenítés

```tsx
{isOpen && <Panel />}                        // csak ha igaz
{isFavorite ? '★' : '☆'}                     // két ág
{items.length > 0 && <List items={items} />} // NEM: {items.length && …} → üres listánál „0” jelenik meg
if (!data) return <p>Betöltés…</p>;          // korai return (a hookok UTÁN!)
```

Több állapot esetén egy `status` mező és egy ág állapotonként:

```tsx
{status === 'loading' && <p>Betöltés…</p>}
{status === 'error' && <p className="error">Hiba: {error}</p>}
{status === 'success' && <ul>…</ul>}
```

A ternary két állapotra való. Háromnál a `:` ág az összes maradékot elkapja, a betöltést is.

## Listák és key

```tsx
<ul>
  {events.map((event) => (
    <li key={event.id}>{event.title}</li>   // a key a map által visszaadott LEGKÜLSŐ elemen
  ))}
</ul>
```

- A `key` stabil és egyedi legyen (azonosító). Index csak statikus, soha nem változó listánál.
- Fragmentnek kulcs csak hosszú alakban adható: `<Fragment key={id}>…</Fragment>`.
- A `map` callbackje adjon vissza: `(x) => (<li/>)` vagy `(x) => { return <li/>; }`. A `{ <li/> }` `return` nélkül semmit sem ad vissza.

## Csoportosítás, rendezés, szűrés renderelés közben

```tsx
// csoportosítás: a filter a map-en belül, üres csoport kihagyása
{CATEGORIES.map((category) => {
  const inCategory = events.filter((e) => e.category === category);
  if (inCategory.length === 0) return null;
  return (
    <div key={category}>
      <h3>{CATEGORY_LABELS[category]} ({inCategory.length})</h3>
      <ul>{inCategory.map((e) => <li key={e.id}>{e.title}</li>)}</ul>
    </div>
  );
})}

// rendezés: a sort HELYBEN rendez, ezért előbb másolat
const sorted = [...events].sort((a, b) => a.date.localeCompare(b.date));
const soon = sorted.slice(0, 3);
const later = sorted.slice(3);

// szűrés: egy const, kis- és nagybetűtől függetlenül
const term = query.trim().toLocaleLowerCase('hu');
const visible = events.filter(
  (e) => (category === 'all' || e.category === category) && e.title.toLocaleLowerCase('hu').includes(term),
);
```

## HTML szabályok, amelyekre a React figyelmeztet

- `<p>`-ben nem lehet `div`, `h1–h6`, `ul`, `article`. `<span>`-ben nem lehet blokkelem.
- `<label>` vegye körbe a mezőt, vagy `htmlFor`-ral mutasson rá: `<label>Név <input … /></label>`.
- Gombon mindig `type="button"`, kivéve a küldő gombot (`type="submit"`).
