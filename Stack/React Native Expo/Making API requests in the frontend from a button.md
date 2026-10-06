  

```tsx
import { useState } from "react";
import { View, Text, Pressable } from "react-native";

export default function App() {
  const [data, setData] = useState<string>("");

  async function handlePress() {
    try {
      const res = await fetch("http://192.168.0.20:3000/api/test");
      const text = await res.text();
      setData(text);
    } catch (err) {
      console.log(err);
    }
  }

  return (
    <View style={{ padding: 20 }}>
      <Pressable
        onPress={handlePress}
        style={{
          backgroundColor: "blue",
          padding: 12,
          borderRadius: 10,
        }}
      >
        <Text style={{ color: "white" }}>Call API</Text>
      </Pressable>

      <Text style={{ marginTop: 20 }}>{data}</Text>
    </View>
  );
}
```