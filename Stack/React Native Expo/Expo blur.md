Install

```bash
npx expo install expo-blur
```

example of a glass card component:

```tsx
import { BlurView } from "expo-blur";
import { Text, StyleSheet, View } from "react-native";

type Props = {
  title: string;
};

export default function GlassCard({ title }: Props) {
  return (
    <BlurView intensity={50} style={styles.card}>
      <Text style={styles.text}>{title}</Text>
    </BlurView>
  );
}

const styles = StyleSheet.create({
  card: {
    padding: 20,
    borderRadius: 16,
    overflow: "hidden", // IMPORTANT for blur clipping
    backgroundColor: "rgba(255,255,255,0.1)", // fallback glass look
  },
  text: {
    fontSize: 18,
    fontWeight: "600",
    color: "white",
  },
});
```

  

Using that component on a screen:

```tsx
import { View } from "react-native";
import GlassCard from "../components/GlassCard";

export default function HomeScreen() {
  return (
    <View style={{ flex: 1, justifyContent: "center", padding: 20 }}>
      <GlassCard title="Profile" />
      <GlassCard title="Settings" />
      <GlassCard title="Messages" />
    </View>
  );
}
```