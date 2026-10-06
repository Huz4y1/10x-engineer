  

Sending a get request when a page loads (Asking for data)

```tsx
export default async function SchedulePage() {
  const response = await fetch('http://localhost:8080/api/events');

  if (!response.ok) {
    throw new Error(`Request failed: ${response.status}`);
  }

  const events: Event[] = await res.json();

  return (
    <main>
      {events.map((e) => <EventCard key={e.id} event={e} />)}
    </main>
  );
}
```

  

sending a get response when a button is clicked (Asking for data)

```tsx
'use client';
import { useState } from 'react';

export default function EventLoader() {
  const [events, setEvents] = useState<Event[]>([]);

  const load = async () => {
    const res = await fetch('http://localhost:8080/api/events');
    if (!res.ok) return;
    setEvents(await res.json());
  };

  return (
    <>
      <button onClick={load}>Load</button>
      {events.map((e) => <EventCard key={e.id} event={e} />)}
    </>
  );
}
```

  

sending a post request (Sending the data)

```tsx
const res = await fetch('http://localhost:8080/api/events', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ event: 'UFC 322', location: 'London', date: '2026-10-01' }),
});

const created: Event = await res.json();   // backends typically echo back the created object
```