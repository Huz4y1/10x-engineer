TWRNC lets you use Tailwind-like utility classes in **React Native**.

**Install**

- Run: `npm install twrnc`
- Import: `import tw from "twrnc";`

# Layout fundamentals (the only ones you usually need)

## 1) `flex-1` (use this on screens)

Use `flex-1` on your top-level screen container so it fills the available space.

```
<View style={tw`flex-1`}>
	...
</View>
```

---

## 2) Flex direction (how children stack)

### Default (most common)

React Native defaults to **column** layout, so you usually do **not** need to set `flex-col`.

```
flex-col  // not needed in React Native most of the time
```

### Horizontal layout

Use `flex-row` when you want children next to each other.

```
flex-row
```

---

## 3) `justify-*` (main axis alignment)

In a column layout, `justify-*` moves content **vertically**.

|Class|Effect|
|---|---|
|`justify-start`|Top|
|`justify-center`|Middle|
|`justify-end`|Bottom|
|`justify-between`|Pushes first item to top and last item to bottom|

---

## 4) `items-*` (cross axis alignment)

In a column layout, `items-*` moves content **horizontally**.

|Class|Effect|
|---|---|
|`items-start`|Left|
|`items-center`|Center|
|`items-end`|Right|

---

## 5) `px-*` and `py-*` (safe spacing)

Padding prevents your UI from touching the screen edges.

- `px-6` keeps left and right spacing safe.
- `py-10` adds breathing space vertically.

---

## 6) `gap-*` (space between children)

Instead of adding margins between items, use `gap-*` on the parent.

```
gap-4
```

---

# The responsive pattern you should default to

## Onboarding screen layout (best practice)

Goal: keep content at the top and the primary button at the bottom, without hard-coded margins.

```
<View style={tw`flex-1 bg-white px-6 py-10 justify-between items-center`}>

	{/* TOP CONTENT */}
	<View style={tw`items-center`}>
		<Text style={tw`text-2xl font-bold`}>Welcome</Text>
	</View>

	{/* BOTTOM BUTTON */}
	<NextButton href="/onBoardingScreens/screen2" />

</View>
```

## Why this works (no absolute positioning needed)

- `flex-1` makes the screen fill any device.
- `justify-between` automatically pushes the bottom section down.
- `px-6` and `py-10` keep consistent spacing on all screens.
- `items-center` keeps alignment consistent.

---

# What not to use for layout

Avoid these for positioning key UI elements:

```
mt-110  // hard-coded spacing
-mt-10  // hard-coded negative spacing
absolute // unless you are doing overlays
fixed positioning
```

**Why:** it breaks on small phones, large tablets, and when font sizes change.

---

# Mental model (simple and reliable)

Think of screens like a container with zones:

```
[ TOP CONTENT ]
[   SPACE     ]
[ BOTTOM CTA  ]
```

- `justify-between` creates the “top and bottom” effect.
- Padding creates breathing room.

---

# A reusable “screen template” component

This structure scales well across many screens.

```
<View style={tw`flex-1 bg-white px-6 py-10 justify-between`}>
	<View>
		{children}
	</View>

	<View>
		<NextButton />
	</View>
</View>
```

---

# Quick summary (memorise)

- `flex-1` for full-screen containers
- `justify-between` for top + bottom layouts
- `items-center` for horizontal centering
- `px-6` and `py-10` for safe spacing
- Prefer `gap-*` over lots of margins
- Avoid absolute positioning for layout

---

# Custom colours (hex) in twrnc

## 1) Use hex colours directly

```
style={tw`bg-[#1E90FF]`}
```

Example button:

```
<Pressable style={tw`bg-[#ff4d4d] p-4 rounded-xl`}>
	<Text style={tw`text-white`}>Button</Text>
</Pressable>
```

---

## 2) Works for text and borders too

Text:

```
<Text style={tw`text-[#1E90FF]`}>Hello</Text>
```

Border:

```
style={tw`border border-[#00ff88]`}
```

---

## 3) Common use cases

Brand colour button:

```
<Pressable style={tw`bg-[#4F46E5] p-4 rounded-xl`}>
```

Soft background:

```
<View style={tw`bg-[#F5F7FF] flex-1`}>
```

Accent text:

```
<Text style={tw`text-[#FFB703]`}>
```

---

## 4) Best practice for repeated colours

If you use the same colours often, create a constants object.

```
const colors = {
	primary: "#4F46E5",
	background: "#F5F7FF",
	danger: "#FF4D4D",
};
```

Then use it in styles:

```
style= backgroundColor: colors.primary 
```

---

## 5) Hex vs Tailwind colour names

|Use case|Recommendation|
|---|---|
|Quick styling|`bg-blue-500`|
|Exact branding|`bg-[#hex]`|
|Design system|Use constants (like `colors.primary`)|

---

# Images in React Native (Expo)

Images are easy once you remember:

1. how to load them
2. how to size them
3. how to position them with flex

---

## 1) Basic image usage

```
import { Image } from "react-native";
```

Local image:

```
<Image
	source={require("../assets/images/onboarding1.png")}
	style= width: 200, height: 200 
/>
```

---

## 2) Image from a URL

```
<Image
	source= uri: "https://picsum.photos/300" 
	style= width: 300, height: 300 
/>
```

---

## 3) The most important rule

**Images must have width and height** or they will not show.

---

## 4) Styling images with `twrnc`

```
<Image
	source={require("../assets/images/onboarding1.png")}
	style={tw`w-48 h-48 rounded-xl`}
/>
```

---

## 5) Control how the image fits (`resizeMode`)

```
<Image
	source={require("../assets/images/onboarding1.png")}
	style={tw`w-full h-64`}
	resizeMode="contain"
/>
```

|Mode|What it does|
|---|---|
|`cover`|Fills space and may crop|
|`contain`|Fits the whole image (no cropping)|
|`stretch`|Distorts the image (avoid)|
|`center`|Keeps original size, centered|

---

## 6) Positioning images (the right way)

Center an image inside a section:

```
<View style={tw`items-center`}>
	<Image
		source={require("../assets/images/onboarding1.png")}
		style={tw`w-48 h-48`}
	/>
</View>
```

Center in the middle of the screen:

```
<View style={tw`flex-1 justify-center items-center`}>
	<Image
		source={require("../assets/images/onboarding1.png")}
		style={tw`w-48 h-48`}
	/>
</View>
```

---

## 7) Full onboarding example (text + image + button)

```
<View style={tw`flex-1 bg-white px-6 py-10 justify-between`}>

	{/* TOP TEXT */}
	<View style={tw`items-center`}>
		<Text style={tw`text-2xl font-bold`}>Welcome</Text>
	</View>

	{/* IMAGE */}
	<View style={tw`items-center`}>
		<Image
			source={require("../assets/images/onboarding1.png")}
			style={tw`w-64 h-64`}
			resizeMode="contain"
		/>
	</View>

	{/* BUTTON */}
	<NextButton href="/onBoardingScreens/screen2" />

</View>
```

---

## 8) Responsive images

If you want the image to adapt to screen width, avoid fixed sizes.

```
style={tw`w-full h-64`}
```

Or calculate size dynamically:

```
const { width } = Dimensions.get("window");
const size = Math.min(width * 0.8, 320);

<Image style= width: size, height: size  />
```

---

## 9) Full-width banner

```
<Image
	source={require("../assets/images/banner.png")}
	style={tw`w-full h-48`}
	resizeMode="cover"
/>
```

---

## 10) Rounded avatar

```
<Image
	source={require("../assets/images/profile.png")}
	style={tw`w-32 h-32 rounded-full`}
/>
```