---
tags: [expo, react-native, mobile, typescript]
---

# Expo

The toolchain that makes React Native usable. Stack: [[Primary Native App Stack]] · Language: [[TypeScript]]

---

## What it is

**React Native** lets you write mobile apps in JavaScript/TypeScript. **Expo** is the toolkit around it that handles the parts you don't want to deal with — native builds, device APIs, updates, and publishing.

> **Without Expo you need Xcode and Android Studio configured correctly before you can see anything.** With Expo you scan a QR code and your app is running on your phone.

## Starting

```bash
npx create-expo-app@latest myapp --template
cd myapp
npx expo start
```

Scan the QR code with **Expo Go** (App Store / Play Store) and it runs on your phone, hot-reloading as you save.

```bash
npx expo start --tunnel     # if phone and laptop aren't on the same network
npx expo start --clear      # clear the Metro bundler cache
```

## Project layout (Expo Router)

```
app/
├── _layout.tsx          # root layout, wraps everything
├── index.tsx            # route: /
├── settings.tsx         # route: /settings
└── (tabs)/
    ├── _layout.tsx      # tab bar
    ├── home.tsx         # route: /home
    └── profile.tsx
components/
assets/
```

> **File-based routing, like [[NextJs TypeScript|Next.js]].** A file in `app/` becomes a screen. `(tabs)` in brackets is a *group* — it organises files without adding a URL segment.

```tsx
// app/_layout.tsx
import { Stack } from "expo-router";

export default function RootLayout() {
  return <Stack screenOptions={{ headerShown: true }} />;
}
```

```tsx
import { Link, useRouter, useLocalSearchParams } from "expo-router";

<Link href="/settings">Settings</Link>

const router = useRouter();
router.push("/detail/42");
router.back();

const { id } = useLocalSearchParams();     // in app/detail/[id].tsx
```

## Components — not HTML

```tsx
import { View, Text, Pressable, ScrollView, FlatList, Image, TextInput } from "react-native";

export default function Home() {
  return (
    <View style={{ flex: 1, padding: 16 }}>
      <Text style={{ fontSize: 24, fontWeight: "600" }}>Readings</Text>
      <Pressable onPress={() => console.log("tapped")}>
        <Text>Refresh</Text>
      </Pressable>
    </View>
  );
}
```

| Web | React Native |
|---|---|
| `<div>` | `<View>` |
| `<p>` `<span>` `<h1>` | `<Text>` |
| `<button>` | `<Pressable>` |
| `<input>` | `<TextInput>` |
| `<img>` | `<Image>` |
| scrolling `<div>` | `<ScrollView>` / `<FlatList>` |

> ⚠️ **All text must be inside `<Text>`.** A bare string inside `<View>` crashes the app. This is the first error everyone hits.

> **Use `<FlatList>` for long lists, not `<ScrollView>`.** ScrollView renders every child immediately; FlatList only renders what's visible. On a 1,000-row list that's the difference between smooth and unusable.

```tsx
<FlatList
  data={readings}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <Text>{item.temp}</Text>}
  refreshing={loading}
  onRefresh={reload}
/>
```

## Styling

```tsx
import { StyleSheet } from "react-native";

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16, backgroundColor: "#0e1117" },
  title: { fontSize: 24, fontWeight: "600", color: "#fff" },
});

<View style={styles.container}>
```

Or Tailwind via TWRNC — see [[CSS for React Native]] and [[Tailwindcss]].

> ⚠️ **This is not [[CSS]].** It's a JS object subset. **[[Flexbox]] works** (and `flexDirection` defaults to `column`, not `row`). Grid, floats, and most selectors do not exist.

## Device APIs

```bash
npx expo install expo-camera expo-location expo-secure-store expo-notifications
```

```tsx
import * as SecureStore from "expo-secure-store";
await SecureStore.setItemAsync("token", jwt);      // encrypted keychain
const token = await SecureStore.getItemAsync("token");

import * as Location from "expo-location";
const { status } = await Location.requestForegroundPermissionsAsync();
const pos = await Location.getCurrentPositionAsync({});
```

> **`npx expo install`, not `npm install`, for expo-* packages.** It picks the version matching your Expo SDK. Using npm directly is the most common cause of "it worked yesterday".

> **Secrets go in `expo-secure-store`** (iOS Keychain / Android Keystore), never `AsyncStorage` — that's plaintext ([[26 — SECURITY]]).

## Calling your API

```tsx
const API_URL = process.env.EXPO_PUBLIC_API_URL;

async function getReadings() {
  const r = await fetch(`${API_URL}/readings`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  if (!r.ok) throw new Error(`HTTP ${r.status}`);
  return r.json();
}
```

> ⚠️ **`localhost` on a phone means the phone**, not your laptop. Use your machine's LAN IP (`192.168.x.x`) or a tunnel. Same trap as containers in [[Docker deep dive]].

> **`EXPO_PUBLIC_*` variables are bundled into the app** — anyone can read them. Never put a secret there; that's what your backend is for ([[Actix Web]], [[FastAPI reference]]).

## Building and shipping

```bash
npm install -g eas-cli
eas login
eas build:configure

eas build --platform android --profile preview     # installable APK
eas build --platform ios --profile production
eas submit --platform ios                          # to the App Store

eas update --branch production                     # OTA - JS changes, no store review
```

> **`eas update` ships JavaScript changes over the air**, bypassing store review. Native changes (a new native module) still need a full build.

## Common mistakes

| Mistake | Fix |
|---|---|
| Bare string outside `<Text>` | Wrap it |
| `npm install expo-*` | `npx expo install` |
| `localhost` for the API | LAN IP or tunnel |
| `<ScrollView>` for a long list | `<FlatList>` |
| Secrets in `EXPO_PUBLIC_*` | Keep them server-side |
| Tokens in AsyncStorage | `expo-secure-store` |
| Metro cache weirdness | `npx expo start --clear` |
| Native module missing in Expo Go | Needs a dev build (`eas build --profile development`) |

## Related

[[Primary Native App Stack]] · [[CSS for React Native]] · [[TypeScript]] · [[Tailwindcss]] · [[NextJs TypeScript]] · [[Actix Web]] · [[Flexbox]]
