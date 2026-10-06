  

Props stands for properties, they are arguments passed into functions

```tsx
function Greeting(props) {
    return (
        <h1>{props.name}</h1>
    );
}
```

This lets us create our own custom tag called greeting with an argument of name

```tsx
<Greeting name=“Huz”> 
```

  

Another real example of using Types and props

```tsx


// 1. The type — your contract for incoming data
type Event = {
  id: number;
  name: string;
  location: string;
  date: string;
};

// 2. The page — fetch, then render
export default async function SchedulePage() {
  const res = await fetch('http://localhost:8080/api/ufc/upcoming', {
    cache: 'no-store',
  });
  if (!res.ok) throw new Error(`Request failed: ${res.status}`);

  const events: Event[] = await res.json();   // ← data now typed

  // 3. Render the typed data
  return (
    <main className="schedule-page">
      <h1>Upcoming UFC events</h1>
      {events.map((event) => (
        <div className="event-card" key={event.id}>
          <h3>{event.name}</h3>
          <p>{event.location}</p>
          <p>{event.date}</p>
        </div>
      ))}
    </main>
  );
}
```