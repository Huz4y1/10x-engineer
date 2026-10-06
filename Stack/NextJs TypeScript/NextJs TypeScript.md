Our purpose for using NextJS is for frontend so here is usually everything we would need:

- Displaying data
- Creating forms
- making tables
- creating dashboards
- calling APIs

  

Creating a new NextJS project

```bash
npx create-next-app@latest frontend
```

Here is what a typical folder structure would look like

```
src/
│
├── app/
│   ├── page.tsx
│   ├── login/
│   │   └── page.tsx
│   ├── dashboard/
│   │   └── page.tsx
│   │
│   ├── globals.css
│   └── layout.tsx
│
├── components/
│   ├── Navbar.tsx
│   ├── Button.tsx
│   ├── UserTable.tsx
│   └── Card.tsx
│
├── services/
│   └── api.ts
│
├── types/
│   └── user.ts
│
├── hooks/
│   └── useUsers.ts
│
└── lib/
    └── utils.ts
```

All components of the frontend are split into individual components and then rendered to the page making the project modular and reusable.

  

Every page is just a function there is no need for a URL route as it’s all done automatically with NextJS. Here is how to create new pages in NextJS

```bash
app/
├── page.tsx               → /
├── about/
│   └── page.tsx           → /about
├── blog/
│   ├── page.tsx           → /blog
│   └── [slug]/
│       └── page.tsx       → /blog/hello-world, /blog/anything
└── dashboard/
    └── settings/
        └── page.tsx       → /dashboard/settings
```

To create a new page you make a new folder in app dir with the name of the page, and then make a page.tsx file inside that dir you made. This page.tsx file is where all the code for the page goes.

  

Example of a page in NextJS

```tsx
export default function Dashboard() {
    return (
        <h1>Dashboard</h1>
    );
}
```

  

[[Using props]]

[[Using UseState]]

[[Sending requests and receiving responses]]

[[TSX]]

[[How to use normal CSS in NextJS]]