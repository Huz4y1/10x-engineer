Here is how CSS looks in React Native

```tsx
import { View, Text, StyleSheet } from 'react-native';

export default function Card() {
  return (
    <View style={styles.card}>
      <Text style={styles.title}>Racing Cage</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  card: {
    backgroundColor: '#1a1a2e',
    padding: 16,
    borderRadius: 12,
  },
  title: {
    color: 'white',
    fontSize: 18,
    fontWeight: 'bold',
  },
});
```

  

# Spacing and Padding

```tsx
{
  padding: 16,              // all sides
  paddingHorizontal: 16,    // left + right (RN shorthand, no CSS equivalent)
  paddingVertical: 8,       // top + bottom
  paddingTop: 12,
  margin: 'auto',           // centering trick works
  gap: 12,                  // spacing between flex children — use this, not margins
}
```

# Flexbox — the whole layout system

Everything is flex. Default `flexDirection` is `column`. The relative sizing you want lives almost entirely here.

```tsx
{
	flex: 1,                     
  flexDirection: 'row',        // or 'column' (default), 'row-reverse'
  justifyContent: 'space-between', // main axis: flex-start | center | flex-end | space-around | space-evenly
  alignItems: 'center',        // cross axis: stretch (default) | flex-start | center | flex-end
  flexWrap: 'wrap',
  gap: 8,
}
```

# Relative sizing for hieght and width

```tsx
{
  width: '100%',      // percentage strings
  height: '50%',
  flex: 1,            // fill remaining space (best for most layouts)
  aspectRatio: 16/9,  // set one dimension, derive the other — great for images/cards
  minWidth: '25%',
  maxWidth: '80%',
}
```

# Images and Sizing

```tsx
import { Image } from 'react-native';

<Image
  source={{ uri: 'https://...' }}   // remote
  style={{ width: '100%', aspectRatio: 16 / 9 }}  // relative width, ratio-locked height
  resizeMode="cover"
/>
```

`resizeMode` controls how the image fills its box (this is RN's `object-fit`):

- `cover` — fill box, crop overflow (most common)
- `contain` — fit whole image inside, may letterbox
- `stretch` — distort to fill
- `center` — no scaling, centered

# For local images you `require` them and RN knows their intrinsic size:

```tsx
<Image source={require('./car.png')}style={{ width:'100%', aspectRatio:1}}/>
```

```tsx
import { Image, StyleSheet } from 'react-native';

// ...in your component:
<Image source={require('./car.png')} style={styles.car} />

// ...at the bottom of the file:
const styles = StyleSheet.create({
  car: {
    width: '100%',
    aspectRatio: 1,
  },
});
```

# Positioning and Placement

```tsx
{
  position: 'absolute',   // or 'relative' (default)
  top: 0,
  left: 0,
  right: 0,               // stretch to both edges without a width
  bottom: 0,
}
```

# Borders and Radius

```tsx
{
  borderWidth: 1,
  borderColor: '#334',
  borderRadius: 12,
  borderTopLeftRadius: 8,   // per-corner
  borderStyle: 'solid',     // dashed | dotted (limited support)
}
```

# Text Styling

```tsx
{
  color: 'white',
  fontSize: 16,
  fontWeight: 'bold',    // '100'–'900' | 'normal' | 'bold'
  lineHeight: 24,
  letterSpacing: 0.5,
  textAlign: 'center',   // left | right | center | justify
  textTransform: 'uppercase',
  fontStyle: 'italic',
}
```

# Backgrounds, Shadows and Opacity

```tsx
{
  backgroundColor: 'rgba(26,26,46,0.9)',   // rgba/hex/named all work
  opacity: 0.8,
  overflow: 'hidden',   // needed to clip children to borderRadius

  // iOS shadow
  shadowColor: '#000',
  shadowOffset: { width: 0, height: 2 },
  shadowOpacity: 0.25,
  shadowRadius: 8,
  // Android shadow (separate property!)
  elevation: 4,
}
```

# Example of positioning things on top of each other like Z-index

```tsx
<View style={styles.container}>
  <Image source={require('./car.png')} style={styles.image} />

  {/* This sits ON TOP of the image, pinned to corners */}
  <View style={styles.overlay}>
    <Text style={styles.label}>Racing Cage</Text>
  </View>
</View>

const styles = StyleSheet.create({
  container: {
    position: 'relative',   // anchor for the absolute child (default anyway)
    width: '100%',
    aspectRatio: 16 / 9,
  },
  image: {
    width: '100%',
    height: '100%',
  },
  overlay: {
    position: 'absolute',
    left: 0,
    right: 0,
    bottom: 0,              // pin to bottom edge, full width
    padding: 12,
    backgroundColor: 'rgba(0,0,0,0.5)',
  },
  label: { color: 'white', fontWeight: 'bold' },
});
```

# Another example of a component using RN CSS system

```tsx
import { View, Text, Image, StyleSheet } from 'react-native';

export default function CarCard() {
  return (
    <View style={styles.screen}>
      <View style={styles.card}>
        <View style={styles.imageWrap}>
          <Image
            source={{ uri: 'https://picsum.photos/800/450' }}
            style={styles.image}
            resizeMode="cover"
          />
          <View style={styles.badge}>
            <Text style={styles.badgeText}>NEW</Text>
          </View>
        </View>

        <View style={styles.body}>
          <Text style={styles.title}>Racing Cage</Text>
          <View style={styles.statsRow}>
            <Text style={styles.stat}>Speed 92</Text>
            <Text style={styles.stat}>Grip 78</Text>
          </View>
        </View>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  screen: {
    flex: 1,                       // fill the whole screen
    backgroundColor: '#0f0f1e',
    padding: 16,
    justifyContent: 'center',
  },
  card: {
    width: '100%',                 // relative, not a pixel width
    backgroundColor: '#1a1a2e',
    borderRadius: 16,
    overflow: 'hidden',            // clips image corners to radius
  },
  imageWrap: {
    width: '100%',
    aspectRatio: 16 / 9,           // height derived from width + ratio
    position: 'relative',
  },
  image: {
    width: '100%',
    height: '100%',
  },
  badge: {
    position: 'absolute',          // overlay, no fixed layout slot
    top: 12,
    right: 12,
    backgroundColor: '#e94560',
    paddingHorizontal: 10,
    paddingVertical: 4,
    borderRadius: 999,
  },
  badgeText: { color: 'white', fontWeight: 'bold', fontSize: 12 },
  body: { padding: 16, gap: 8 },
  title: { color: 'white', fontSize: 20, fontWeight: 'bold' },
  statsRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
  },
  stat: { color: '#a0a0c0', flex: 1 },   // each stat takes equal share
});
```