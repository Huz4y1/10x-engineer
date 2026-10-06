---
tags: [css, language, web]
---

# CSS

Styling. Language index: [[Languages]] · Markup: [[HTML]] · Utility framework: [[Tailwindcss]]

**Deeper notes:** [[Flexbox]] · [[Grid]] · [[Responsive design]] · [[CSS libraries]]

---
  

- Colors
- Fonts
- Sizes
- Spacing
- Borders
- Layout
- Animations
- Positioning

  

Any h1 in HTML that you label becomes blue and and p becomes green

```css
h1 {
    color: blue;
}

p {
    color: green;
}
```

  

Using classes, better than using tags. The dot means that its a class. Classes can be used for many elements in HTML

```css
.warning {
    color: red;
}

.success {
    color: green;
}
```

```html
<p class="warning">Danger!</p>

<p class="success">Success!</p>
```

  

IDs are used for 1 specific element

```css
#main-title {
    color: purple;
}
```

```html
<h1 id="main-title">Welcome</h1>
```

  

styling

Background color

```css
.box {
    background-color: yellow;
}
```

Width and Height

```css
.box {
    width: 40%;
    height: 100vh;
}
```

borders

```css
.box {
    border: 2px solid black;
}

can have different styles too
border: 2px solid black;
border: 2px dashed red;
border: 2px dotted blue;
```

Rounded corners

```css
.box {
    border-radius: 10px;
}
```

Padding = the space inside something like a box

```css
.box {
    padding: 20px;
}
```

Margin = the space outside something like a box, creates gaps between elements

```css
.box {
    margin: 20px;
}
```

Font size

```css
font-size: 1rem;
```

Centering text

```css
h1 {
    text-align: center;
}
```

bold text

```css
h1 {
    font-weight: bold;
}
```

Hover effects

```css
button {
    background-color: blue;
    color: white;
}

button:hover {
    background-color: red;
}
```

  

layouts for containers

Flexbox this is to make items in a big blog be in a row

```css
.container {
    display: flex;
    display-direction: row;
}
```

making the layout in columns

```css
.container {
    display: flex;
    flex-direction: column;
}
```

making gaps between items in the container

```css
.container {
    display: flex;
    flex-direction: row;
    gap: 20px;
}
```

making items in the container move to the next line

```css
.container {
    display: flex;
    flex-direction: row;
    flex-wrap: wrap;
}
```

grid = creating an excel spreadsheet style layout

```css
.grid {
    display: grid;
    grid-template-columns: 1fr 1fr;   //each fr represents a column
}
```

  

Layering elements on top of eachother

suppose you have:

```html
<div class="box1"></div>
<div class="box2"></div>
```

the css for that is

```css
.box1 {
  width: 200px;
  height: 200px;
  background: red;
  position: absolute;
  top: 50px;
  left: 50px;
  z-index: 1;
}

.box2 {
  width: 200px;
  height: 200px;
  background: blue;
  position: absolute;
  top: 100px;
  left: 100px;
  z-index: 2;
}
```

the thing that makes it layer on top is the z-index and only works if position is defined

  

Making a card had a translucent blurred effect

```css
.card {
  width: 300px;
  height: 200px;

  background: rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(12px);

  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.3);
}
```