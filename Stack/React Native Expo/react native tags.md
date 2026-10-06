## `<View>`

Like a `<div>` (container)

```tsx
<View>
<Text>Hello</Text>
</View>
```

Used for:

- layout
- spacing
- grouping UI

  

## `<Text>`

Like `<p>` / `<span>`

```tsx
<Text>Hello world</Text>
```

Used for:

- ALL text on screen (you can’t use plain strings)

  

## `<ScrollView>`

Scrollable container

```tsx
<ScrollView>
<Text>Lots of content...</Text>
</ScrollView>
```

Used when content is longer than screen.

  

## `<FlatList>` IMPORTANT

Optimized list (better than ScrollView for big lists)

```tsx
<FlatList
data={[1,2,3]}
renderItem={({ item }) =><Text>{item}</Text>}
/>
```

Used for:

- feeds
- chats
- lists

  

# 2. Touch / buttons

## `<Pressable>` (modern standard)

```tsx
<PressableonPress={() =>console.log("pressed")}>
<Text>Click me</Text>
</Pressable>
```

---

## `<Button>` (basic, limited styling)

```tsx
<Buttontitle="Click me"onPress={() => {}}/>
```

## `<TouchableOpacity>` (older but still used)

```tsx
<TouchableOpacityonPress={() => {}}>
<Text>Tap me</Text>
</TouchableOpacity>
```

  

# 3. Media

## `<Image>`

```tsx
<Imagesource={{ uri:"https://..." }}/>
```

## `<Video>` (from Expo)

```tsx
import {Video }from"expo-av";
```

  

## `<TextInput>`

```tsx
<TextInput
placeholder="Enter name"
onChangeText={(text) =>console.log(text)}
/>
```

Used for:

- forms
- login
- search bars

  

# Layout helpers

## `<SafeAreaView>`

Avoids notch / status bar overlap

```tsx
<SafeAreaView>
<Text>Safe content</Text>
</SafeAreaView>
```