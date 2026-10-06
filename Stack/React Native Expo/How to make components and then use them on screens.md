example of a card component

```tsx
import { View, Text, StyleSheet } from "react-native";

type Props = {
  title: string;
  subtitle?: string;
};

export default function Card({ title, subtitle }: Props) {
  return (
    <View style={styles.card}>
      <Text style={styles.title}>{title}</Text>
      {subtitle && <Text style={styles.subtitle}>{subtitle}</Text>}
    </View>
  );
}

const styles = StyleSheet.create({
  card: {
    backgroundColor: "white",
    padding: 16,
    borderRadius: 12,
    marginBottom: 12,

    // shadow (iOS)
    shadowColor: "#000",
    shadowOpacity: 0.1,
    shadowRadius: 6,
    shadowOffset: { width: 0, height: 2 },

    // shadow (Android)
    elevation: 3,
  },

  title: {
    fontSize: 18,
    fontWeight: "bold",
  },

  subtitle: {
    marginTop: 4,
    color: "gray",
  },
});
```

  

using it on a screen:

```tsx
import { View } from "react-native";
import Card from "../components/Card";

export default function HomeScreen() {
  return (
    <View style={{ flex: 1, padding: 20, backgroundColor: "#f2f2f2" }}>
      <Card title="Profile" subtitle="View your account info" />
      <Card title="Settings" subtitle="Manage preferences" />
      <Card title="Messages" subtitle="Check your chats" />
    </View>
  );
}
```