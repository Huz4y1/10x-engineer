Install these packages

```tsx
npx expo install @react-navigation/native
```

```tsx
npx expo install react-native-screens react-native-safe-area-context
```

```tsx
npx expo install @react-navigation/native-stack
```

If you want Instagram style bottom menu then install this one too

```tsx
npx expo install @react-navigation/bottom-tabs
```

Or if you want a side bar menu then install this one

```tsx
npx expo install @react-navigation/drawer
```

  

Your App.tsx file should look like this

```tsx
import { NavigationContainer } from "@react-navigation/native";
import { createNativeStackNavigator } from "@react-navigation/native-stack";

import HomeScreen from "./screens/HomeScreen";
import DetailsScreen from "./screens/DetailsScreen";

const Stack = createNativeStackNavigator();

export default function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Details" component={DetailsScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

  

Here is a simple clean folder structure for you app

```bash
my-app/
│
├── App.tsx
├── package.json
│
├── screens/
│   ├── HomeScreen.tsx
│   ├── DetailsScreen.tsx
│   ├── ProfileScreen.tsx
│
├── components/
│   ├── Button.tsx
│   ├── Header.tsx
│
├── services/   
│   ├── api.ts
│
├── assets/
```

  

making a button on a screen which takes you to another screen:

```tsx
import { View, Text, Pressable, StyleSheet } from "react-native";

export default function HomeScreen({ navigation }: any) {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Home Screen</Text>

      <Pressable
        style={styles.button}
        onPress={() => navigation.navigate("Details")}
      >
        <Text style={styles.buttonText}>Go to Details</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: "center",
    alignItems: "center",
  },
  title: {
    fontSize: 24,
    marginBottom: 20,
  },
  button: {
    backgroundColor: "blue",
    padding: 12,
    borderRadius: 10,
  },
  buttonText: {
    color: "white",
    fontWeight: "bold",
  },
});
```

  

```tsx
import { View, Text } from "react-native";

export default function DetailsScreen() {
  return (
    <View style={{ flex: 1, justifyContent: "center", alignItems: "center" }}>
      <Text>Details Screen</Text>
    </View>
  );
}
```