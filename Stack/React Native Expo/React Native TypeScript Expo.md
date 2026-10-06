Installing expo

```bash
npm install -g expo-cli
```

creating a project

```bash
npx create-expo-app@latest frontend
```

run it

```bash
cd my-app
npx expo start
```

Then on your phone go to safari and type this URL in

```rust
exp://IPv4_Address_of_computer:PORT
```

Project structure

```bash
app/
├── _layout.tsx        ← root layout (like app/layout.tsx in Next)
├── modal.tsx          ← a screen that opens as a modal
└── (tabs)/
    ├── _layout.tsx    ← defines the bottom tab bar
    ├── index.tsx      ← first tab (this is "/")
    └── explore.tsx    ← second tab ("/explore")
```

everything starts inside the app/

### What is (tabs)/ ?

```tsx
app/
  _layout.tsx          ← root <Stack>, wraps everything
  (user)/
    _layout.tsx        ← <Tabs>  → user gets a bottom tab bar
    index.tsx          ← a user tab
    profile.tsx        ← another user tab
  (admin)/
    _layout.tsx        ← <Tabs> or <Stack> → admin's own nav
    dashboard.tsx
    reports.tsx
```

It’s like having specific stuff for a specific group of screens so only the admin pages will have an admin layout which may contain admin specific headers or footers

### Now if you don’t want the boiler plate stuff:

- just delete all the things you don’t need
- delete the (tabs) folder aswell

create a file called index.tsx in app/

Inside here put this in here to start the page this is going to be the main home page of the app

```tsx
import { Text, View } from 'react-native';

export default function Index() {
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <Text>weather nice</Text>
    </View>
  );
}
```

leave _layout.tsx file

replace the boiler plate code in here and replace it with this

```tsx
import { Stack } from 'expo-router';

export default function RootLayout() {
  return <Stack />;
}
```

How to make a new screen:

create a new file in the app/ dir

```tsx
// app/settings.tsx
import { View, Text } from 'react-native';

export default function Settings() {
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <Text>Settings screen</Text>
    </View>
  );
}
```

Now when you want to go to this page just create a link on any page to take you to it

```tsx
import { Link } from 'expo-router';

<Link href="/settings">Go to settings</Link>
```

This is the mobile UI when you open App.tsx

```tsx
import { View, Text } from "react-native";

export default function App() {
  return (
    <View>
      <Text>Hello Expo 👋</Text>
    </View>
  );
}
```

  

Styling

React native uses JS object

```tsx
import { View, Text, StyleSheet } from "react-native";

export default function App() {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Hello Expo 👋</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,              // fill screen
    justifyContent: "center",
    alignItems: "center",
  },
  title: {
    fontSize: 24,
    fontWeight: "bold",
  },
});
```

  

NextJS useState equivalent in react native

```tsx
import { useState } from "react";
import { View, Text, Pressable } from "react-native";

export default function App() {
  const [count, setCount] = useState<number>(0);

  return (
    <View style={{ flex: 1, justifyContent: "center", alignItems: "center" }}>
      <Text>Count: {count}</Text>

      <Pressable onPress={() => setCount(count + 1)}>
        <Text>Increase</Text>
      </Pressable>
    </View>
  );
}
```

  

Creating components

```tsx
type GreetingProps = {
  name: string;
};

function Greeting({ name }: GreetingProps) {
  return <Text>Hello {name}</Text>;
}
```

  

[[Expo]]

[[navigation system setup]]

[[react native tags]]

[[Expo blur]]

[[How to make components and then use them on screens]]

[[Making API requests in the frontend from a button]]

[[CSS for React Native]]

[[TWRNC (Tailwind for React Native)]]