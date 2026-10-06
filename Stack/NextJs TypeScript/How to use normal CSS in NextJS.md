Creating a CSS global file so it can be used in the nextjs project

In the app/layout.tsx file put this at the top of the file

```tsx
import "./globals.css";
```

Now when you want to use CSS just do this in the page you are coding

```tsx
<main className="home-main">
        
        <div>
          <h1>View the upcoming UFC schedule</h1>
          <button>GO</button>
        </div>

        <div>
          <h1>View the upcoming F1 schedule</h1>
          <button>GO</button>
        </div>

        <div>
          <h1>View the upcoming WEC schedule</h1>
          <button>GO</button>
        </div>
     
      </main>
```

Add the css elements into globals.css file

```css
.home-main {
  display: flex;
  flex-direction: row;
}
```