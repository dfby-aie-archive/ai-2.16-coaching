# Lesson 2.16: Building a Sample App I - Dogstagram

## Overview

- **Duration:** ~2 hours (hands-on lab). Per-section timings are noted throughout as a pacing guide - they assume learners copy code from this file rather than typing it from scratch.
- **Prerequisites:** Lesson 2.15 - UI Composition and Layout Management in Mobile Applications

## Learning Objectives

By the end of this lesson, you will be able to:

1. **Build** a complete React Native app from scratch using core components, styling, and flex layout
2. **Extract** reusable components (`Button`, `LoadingOverlay`) and a shared color token file
3. **Apply** `FlatList`, `Alert`, `Pressable`, `expo-linear-gradient`, and custom fonts to solve real UI problems

## Introduction

In this session you will build Dogstagram - a dog image gallery app that fetches random dog photos from a public API, displays them in a scrollable list, and lets you add and clear photos. There is no new theory today; the goal is fluency. Every concept you use here - core components, `StyleSheet`, flexbox, `useState`, `useEffect`, `useRef`, and component decomposition - was introduced in 2.14 and 2.15. Working through a complete app from scratch is how those pieces connect into a mental model you can apply to any mobile project.

We will build this step by step, in the same order problems actually show up: something will work, then break, then we fix it, and each fix introduces the next concept. That is closer to real development than being handed a finished pattern.

Lesson 2.17 will add navigation so Dogstagram can grow into a multi-screen experience.

---

## Setup (5 minutes)

Create the project and start the dev server:

```bash
npx create-expo-app --template blank dogstagram
cd dogstagram
npx expo start
```

> **Select SDK 57** to match the Expo Go app on your phone or simulator.

Start the iOS or Android simulator (or open Expo Go on your phone) and confirm the default app is running before continuing. We will install additional packages as we need them, so you can see exactly why each one is there.

---

## Part 1: App Shell (15 minutes)

### The container and header

Open `App.js` and replace the entire file with a container and a header:

```jsx
// App.js
import { StyleSheet, Text, View } from "react-native";

export default function App() {
  return (
    <View style={styles.container}>
      <Text style={styles.appHeader}>🐶 Dogstagram</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
  },
  appHeader: {
    fontSize: 24,
    fontWeight: "700",
    marginBottom: 20,
    textAlign: "center",
  },
});
```

Check that the header appears on screen.

### State for the dog list

We need a state to keep the list of dog images, and a loading state for while a request is in flight. Add this inside `App`, above the `return`:

```jsx
// App.js
const [dogs, setDogs] = useState([]);
const [isLoading, setIsLoading] = useState(false);
```

We used `useState` but have not imported it yet. Rather than waiting to hit a crash on the simulator, let's catch this the way you would on a real project: with a linter. Set one up now:

```bash
npx expo lint
```

> If this is the first time linting has run in this project, the CLI will offer to install and create an ESLint config - accept the default, and it will proceed to lint the project right after.

The lint output should flag `useState` as undefined - the same mistake that would otherwise show up as a red error screen at runtime, caught here as a single line of output before you ever press run. This is why it is good practice to set up a linter early in a project and keep it running as you go.

Fix the actual problem by adding the import:

```jsx
// App.js
import { useState } from "react";
```

Run `npx expo lint` again to confirm it is clean, then check the simulator.

### Fetching a dog

Add the dog API URL above the component, and a `getDog` function inside it:

```jsx
// App.js
const API_URL = "https://dog.ceo/api/breeds/image/random";

// inside App:
const getDog = async () => {
  try {
    setIsLoading(true);
    const response = await fetch(API_URL);
    if (!response.ok) {
      throw new Error(`Request failed with status ${response.status}`);
    }
    const data = await response.json();
    setDogs((prevDogs) => [...prevDogs, data.message]);
  } catch (error) {
    console.error(error);
  } finally {
    setIsLoading(false);
  }
};
```

### Wiring up a button

Add a button to trigger `getDog`. We will use React Native's built-in `Button` component for now:

```jsx
// App.js
import { StyleSheet, Text, View, Button } from "react-native";

// inside the returned JSX, below the header:
<Button title="Get Dog" onPress={getDog} />
```

`Button` takes a `title` prop and an `onPress` event handler - notice this is `onPress`, not the web's `onClick`.

Now add a second button to clear the list:

```jsx
// App.js
<Button title="Get Dog" onPress={getDog} />
<Button title="Clear" onPress={() => setDogs([])} />
```

Run the app. Notice two things about how the buttons are laid out:

- They stack in a column, not a row. This is because the default `flexDirection` for a `View` is `"column"`.
- They stretch to fill the width. This is because `alignItems` defaults to `"stretch"` on the cross-axis. Try adding `alignItems: "center"` to `container` to see the difference, then remove it again - we want the buttons side by side, not centered individually.

### Buttons side by side

Wrap the buttons in their own `View` and give it `flexDirection: "row"`:

```jsx
// App.js
<View style={styles.container}>
  <Text style={styles.appHeader}>🐶 Dogstagram</Text>
  <View style={styles.buttonsContainer}>
    <Button title="Get Dog" onPress={getDog} />
    <Button title="Clear" onPress={() => setDogs([])} />
  </View>
</View>
```

```jsx
// App.js
buttonsContainer: {
  flexDirection: "row",
  gap: 15,
  marginBottom: 20,
  justifyContent: "center",
},
```

Also add some breathing room at the top of the screen so the header is not flush against the status bar:

```jsx
// App.js
container: {
  flex: 1,
  backgroundColor: "#fff",
  marginTop: 60,
},
```

> This `marginTop: 60` is a temporary fix. We will replace it with something more reliable once we deal with the notch later in this lesson.

---

## Part 2: Displaying and Scrolling Images (12 minutes)

### Rendering the list

Display the dog images by mapping over the array, the same way you would in React web. Use the array index as the key for now - this API does not give us a unique ID for each image:

```jsx
// App.js
import { Image } from "react-native";

// inside container, below buttonsContainer:
<View style={styles.dogsContainer}>
  {dogs.map((dogUrl, index) => (
    <Image key={index} source={{ uri: dogUrl }} style={styles.dogImage} />
  ))}
</View>
```

```jsx
// App.js
dogsContainer: {
  flex: 1,
  alignItems: "center",
},
dogImage: {
  width: 300,
  height: 300,
  borderRadius: 10,
  marginBottom: 20,
},
```

Test it: tap "Get Dog" a few times, then "Clear".

### The scrolling problem

Once you have loaded four or five dogs, the images at the bottom are off-screen and you cannot reach them. On the web, content overflow becomes scrollable automatically. React Native does not do this for you - you have to explicitly mark content as scrollable, using the `ScrollView` component.

> **`ScrollView` needs a bounded height to work.** Make sure all of its parent views have a bounded height (here, `flex: 1` on `container` provides that).

```jsx
// App.js
import { ScrollView } from "react-native";

<View style={styles.dogsContainer}>
  <ScrollView showsVerticalScrollIndicator={false}>
    {dogs.map((dogUrl, index) => (
      <Image key={index} source={{ uri: dogUrl }} style={styles.dogImage} />
    ))}
  </ScrollView>
</View>
```

Test it again - all the loaded dogs should now be reachable by scrolling.

### Showing a loading spinner

Right now there is no feedback while a request is in flight - tapping "Get Dog" just does nothing visible until the image appears. Add an `ActivityIndicator`, React Native's built-in spinner, driven by the `isLoading` state we already set in `getDog`:

```jsx
// App.js
import { ActivityIndicator } from "react-native";

<View style={styles.dogsContainer}>
  <ScrollView showsVerticalScrollIndicator={false}>
    {dogs.map((dogUrl, index) => (
      <Image key={index} source={{ uri: dogUrl }} style={styles.dogImage} />
    ))}
  </ScrollView>
  {isLoading && <ActivityIndicator size="large" color="#333" />}
</View>
```

Test it - tap "Get Dog" and you should briefly see the spinner while the request is in flight.

### A nicer loading overlay

Right now the spinner pushes into the layout below the images. Let's make it float on top of the content instead:

```jsx
// App.js
{isLoading && (
  <View style={styles.loadingOverlay}>
    <ActivityIndicator size="large" color="#333" />
  </View>
)}
```

```jsx
// App.js
loadingOverlay: {
  ...StyleSheet.absoluteFill,
  justifyContent: "center",
  alignItems: "center",
},
```

`StyleSheet.absoluteFill` expands to `{ position: "absolute", top: 0, right: 0, bottom: 0, left: 0 }` - a convenient shortcut for anything that should cover its parent entirely, such as overlays or backgrounds. We are leaving `backgroundColor` off here so the overlay blends into whatever is behind it, whether that is a plain background now or the gradient we add in Part 3.

---

## Activity: Fix the Notch (10 minutes)

Our `marginTop: 60` hack for avoiding the status bar and notch is fragile - it does not adapt to different devices, and it does not account for the home indicator at the bottom of the screen either. You already met the proper solution for this in Lesson 2.15.

**Task:** Replace `marginTop: 60` with a solution that automatically adapts to each device's safe area, on all four edges.

**Hints:**

1. Install `react-native-safe-area-context`: `npx expo install react-native-safe-area-context`.
2. You need two components from that package, used together: one as a top-level provider, one wrapping your actual content.
3. The provider goes around everything; the second component takes over the role `container` was playing.
4. Once applied, `marginTop: 60` can be removed from `styles.container`.

<details>
<summary>Reference solution</summary>

```jsx
// App.js
import { SafeAreaProvider, SafeAreaView } from "react-native-safe-area-context";
```

```jsx
// App.js
return (
  <SafeAreaProvider>
    <SafeAreaView style={{ flex: 1 }}>
      <View style={styles.container}>
        {/* ...existing content... */}
      </View>
    </SafeAreaView>
  </SafeAreaProvider>
);
```

Remove `marginTop: 60` from `styles.container` - `SafeAreaView` now handles the inset automatically.

</details>

---

## Part 3: Gradient Background (6 minutes)

Previously, in the React web version of this app, the background used a CSS gradient:

```css
body {
  background-image: linear-gradient(to right, #e5dbff, #f3f0ff);
}
```

React Native does not have CSS, so there is no `linear-gradient` style property to reach for. To get the same effect we need a library:

```bash
npx expo install expo-linear-gradient
```

Wrap the app with `LinearGradient`, reusing the same two colors from the web version:

```jsx
// App.js
import { LinearGradient } from "expo-linear-gradient";

return (
  <SafeAreaProvider>
    <LinearGradient colors={["#e5dbff", "#f3f0ff"]} style={{ flex: 1 }}>
      <SafeAreaView style={{ flex: 1 }}>
        <View style={styles.container}>
          {/* ...existing content... */}
        </View>
      </SafeAreaView>
    </LinearGradient>
  </SafeAreaProvider>
);
```

Remove `backgroundColor: "#fff"` from `styles.container` if it is still there - it would cover the gradient.

Tap "Get Dog" and check the loading overlay - since it has no `backgroundColor`, it blends into the gradient automatically with no further changes needed.

---

## Part 4: Scrolling to the Newest Image and Unique Keys (10 minutes)

### Scroll to end on load

When a new dog is added it appears at the bottom, off-screen. Let's scroll there automatically. Add a ref:

```jsx
// App.js
import { useRef } from "react";

// inside App:
const scrollViewRef = useRef(null);
```

`ScrollView` exposes an `onContentSizeChange` prop that fires whenever its content changes height - exactly when a new image is added:

```jsx
// App.js
<ScrollView
  ref={scrollViewRef}
  onContentSizeChange={() => {
    scrollViewRef.current.scrollToEnd({ animated: true });
  }}
  showsVerticalScrollIndicator={false}
>
```

To keep this readable, pull the callback into its own function:

```jsx
// App.js
const scrollToEnd = () => {
  scrollViewRef.current.scrollToEnd({ animated: true });
};
```

```jsx
// App.js
<ScrollView ref={scrollViewRef} onContentSizeChange={scrollToEnd} showsVerticalScrollIndicator={false}>
```

### Why the index key is not good enough

We have been using the array index as the `key` for each image. That works, but it is fragile - if items are ever removed from the middle of the list, indices shift and React can misidentify which item changed. We also have a more specific problem: this dog API sometimes returns the same URL twice in a row, so we cannot use the URL as a key either.

We need a real unique ID generated on our end, independent of what the API sends back. Expo provides this:

```bash
npx expo install expo-crypto
```

```jsx
// App.js
import * as Crypto from "expo-crypto";
```

> There are other ways to generate a UUID in React Native - for example the `react-native-uuid` package (`npm i react-native-uuid`, then `import uuid from "react-native-uuid"` and `uuid.v4()`). `expo-crypto` is used here because it needs no extra native setup in an Expo project.

Update the dog list to store an object with an `id` and `url`, instead of a bare URL string:

```jsx
// App.js
const getDog = async () => {
  try {
    setIsLoading(true);
    const response = await fetch(API_URL);
    if (!response.ok) {
      throw new Error(`Request failed with status ${response.status}`);
    }
    const data = await response.json();
    setDogs((prevDogs) => [
      ...prevDogs,
      { id: Crypto.randomUUID(), url: data.message },
    ]);
  } catch (error) {
    console.error(error);
  } finally {
    setIsLoading(false);
  }
};
```

And update the render to match:

```jsx
// App.js
{dogs.map((dog) => (
  <Image key={dog.id} source={{ uri: dog.url }} style={styles.dogImage} />
))}
```

---

## Part 5: From ScrollView to FlatList (12 minutes)

So far this works, but there is a hidden cost. `ScrollView` renders **all** of its children at once, whether or not they are currently visible on screen. For a short, fixed list that is fine. But Dogstagram's list only grows - imagine a learner tapping "Get Dog" fifty times. Creating fifty `Image` components and their underlying native views up front, most of which are never seen, wastes memory and slows rendering.

`FlatList` solves this by only rendering the items currently in (or near) the viewport, and discarding off-screen items as the user scrolls - a technique called virtualization.

### Migrating to FlatList

`FlatList` takes a `data` prop (the array) and a `renderItem` prop (a function describing how to render each item). `renderItem` receives an object; the item itself is under its `item` property:

```jsx
// App.js
import { FlatList } from "react-native";

<View style={styles.dogsContainer}>
  <FlatList
    data={dogs}
    keyExtractor={(dog) => dog.id}
    renderItem={({ item }) => (
      <Image source={{ uri: item.url }} style={styles.dogImage} />
    )}
  />
  {isLoading && (
    <View style={styles.loadingOverlay}>
      <ActivityIndicator size="large" color="#333" />
    </View>
  )}
</View>
```

For the key, `FlatList` uses a separate `keyExtractor` prop instead of a `key` prop on each item. It tells `FlatList` which field to use as the unique key.

> **Note:** if your items already have a `key` field, `keyExtractor` is not strictly required - `FlatList`'s default extractor checks `item.key`, then `item.id`, then falls back to the index. It is still good practice to pass it explicitly so the behavior is clear to anyone reading the code.

We lost scroll-to-end in the migration, but it still works the same way - just reconnect the ref and callback:

```jsx
// App.js
<FlatList
  ref={scrollViewRef}
  onContentSizeChange={scrollToEnd}
  data={dogs}
  keyExtractor={(dog) => dog.id}
  renderItem={({ item }) => (
    <Image source={{ uri: item.url }} style={styles.dogImage} />
  )}
/>
```

You can pull `renderItem` out into its own function to keep the JSX cleaner:

```jsx
// App.js
const renderDogItem = ({ item }) => (
  <Image source={{ uri: item.url }} style={styles.dogImage} />
);

// in the FlatList:
<FlatList
  ref={scrollViewRef}
  onContentSizeChange={scrollToEnd}
  data={dogs}
  keyExtractor={(dog) => dog.id}
  renderItem={renderDogItem}
/>
```

### Empty state and scroll indicator

`FlatList` gives us a dedicated prop for showing something when the list is empty, instead of writing `dogs.length === 0 && ...` ourselves:

```jsx
// App.js
<FlatList
  ref={scrollViewRef}
  onContentSizeChange={scrollToEnd}
  data={dogs}
  keyExtractor={(dog) => dog.id}
  renderItem={renderDogItem}
  ListEmptyComponent={<Text>🫤 No dogs yet!</Text>}
  showsVerticalScrollIndicator={false}
/>
```

> For advanced use cases, `FlatList` also exposes props to tune how much off-screen content is kept in memory. We will not need that here, but it is worth knowing it exists for lists with hundreds or thousands of items.

---

## Part 6: Confirming Before Clearing (6 minutes)

It would be easy to accidentally tap "Clear" and lose every photo. Let's confirm first, using React Native's `Alert`.

`Alert.alert(title, message, buttons)` takes a title, a message, and an array of button objects. Each button object looks like `{ text: "button text", onPress: () => {}, style: "default" }`.

```jsx
// App.js
import { Alert } from "react-native";

// inside App, replacing the inline () => setDogs([]):
const handleClearDogs = () => {
  if (dogs.length === 0) {
    Alert.alert("Clear Dogs", "No dogs to clear", [{ text: "OK" }]);
    return;
  }

  Alert.alert("Clear Dogs", "Are you sure you want to clear all dogs?", [
    { text: "Cancel" },
    { text: "OK", onPress: () => setDogs([]) },
  ]);
};
```

```jsx
// App.js
<Button title="Get Dog" onPress={getDog} />
<Button title="Clear" onPress={handleClearDogs} />
```

`style` on a button object is optional and only affects iOS: use `"cancel"` for dismissing, `"destructive"` for dangerous or irreversible actions, and `"default"` for everything else.

Test that pressing "Get Dog" loads an image, pressing "Clear" shows the confirmation, and pressing "OK" removes all images. Also confirm that clearing an already-empty list shows the different message.

---

## Part 7: Custom Button Component (18 minutes)

The native `Button` looks different on iOS and Android, and offers very little styling control. Let's build our own.

### Basic structure

A button is generally a container with text inside it. Create a `components/` folder and add `Button.js`:

```jsx
// components/Button.js
import { Text } from "react-native";

function Button() {
  return <Text>My Button</Text>;
}

export default Button;
```

### Accepting props

To make it reusable, it needs to accept `children` (the button's content) and `onPress` (the handler). Wrap the text in `Pressable`, React Native's tap-handling primitive:

```jsx
// components/Button.js
import { Text, Pressable } from "react-native";

function Button({ children, onPress }) {
  return (
    <Pressable onPress={onPress}>
      <Text>{children}</Text>
    </Pressable>
  );
}

export default Button;
```

Replace the native buttons in `App.js`:

```jsx
// App.js
import Button from "./components/Button";

<View style={styles.buttonsContainer}>
  <Button onPress={getDog}>Get Dog</Button>
  <Button onPress={handleClearDogs}>Clear</Button>
</View>
```

Test it - both buttons should still work, just unstyled.

### Styling the button

Two things need styling: the container and the text.

```jsx
// components/Button.js
import { Text, Pressable, StyleSheet } from "react-native";

function Button({ children, onPress }) {
  return (
    <Pressable onPress={onPress} style={styles.buttonContainer}>
      <Text style={styles.buttonText}>{children}</Text>
    </Pressable>
  );
}

const styles = StyleSheet.create({
  buttonContainer: {
    backgroundColor: "#6741d9",
    paddingVertical: 8,
    paddingHorizontal: 16,
    borderRadius: 5,
    alignItems: "center",
    justifyContent: "center",
  },
  buttonText: {
    color: "#fff",
    fontWeight: "700",
    textTransform: "uppercase",
  },
});

export default Button;
```

### Adding a shadow

The usual CSS shadow properties do not exist in React Native. Shadows are platform-specific:

```jsx
// components/Button.js
buttonContainer: {
  backgroundColor: "#6741d9",
  paddingVertical: 8,
  paddingHorizontal: 16,
  borderRadius: 5,
  alignItems: "center",
  justifyContent: "center",

  // Shadow for Android - higher value means a bigger shadow
  elevation: 8,
  // Shadow for iOS
  shadowColor: "#000",
  // width: horizontal offset, height: vertical offset
  shadowOffset: { width: 0, height: 2 },
  shadowOpacity: 0.25,
  // 1 = sharp shadow, 10 = soft blurry shadow
  shadowRadius: 2,
},
```

> The iOS shadow only shows up if `backgroundColor` is not transparent. Android's `elevation` only works on `View` or `Pressable`, not on `Text`. Even with both set, the two platforms will not render an identical shadow - close enough is usually the practical goal here. If you want a pixel-perfect cross-platform shadow, the `react-native-shadow-2` package exists, but it adds a dependency, some performance overhead, and its own edge cases (like clipping), so reach for it only if the built-in shadow genuinely is not good enough.

### Feedback when pressed

When a button is tapped, users expect some visual feedback. `Pressable`'s `style` prop can take a function instead of a plain object; the function receives a `pressed` boolean:

```jsx
// components/Button.js
buttonPressed: {
  opacity: 0.6,
},
```

```jsx
// components/Button.js
style={({ pressed }) => [
  styles.buttonContainer,
  pressed && styles.buttonPressed,
]}
```

The array syntax lets React Native merge multiple style objects in order. When `pressed` is `true`, `buttonPressed` is included; when `false`, the `&&` short-circuits to `false`, which React Native simply ignores.

> **Android-only alternative:** `Pressable` supports an `android_ripple` prop for a native ripple effect, but it has no effect on iOS, so the opacity approach above is what gives consistent feedback on both platforms. If you want the ripple on Android and the opacity fade on iOS, you can combine them using `Platform.OS === "ios"` to conditionally apply `buttonPressed`.

The full component:

```jsx
// components/Button.js
import { StyleSheet, Text, Pressable } from "react-native";

function Button({ children, onPress }) {
  return (
    <Pressable
      onPress={onPress}
      style={({ pressed }) => [
        styles.buttonContainer,
        pressed && styles.buttonPressed,
      ]}
    >
      <Text style={styles.buttonText}>{children}</Text>
    </Pressable>
  );
}

export default Button;

const styles = StyleSheet.create({
  buttonContainer: {
    backgroundColor: "#6741d9",
    paddingVertical: 8,
    paddingHorizontal: 16,
    borderRadius: 5,
    alignItems: "center",
    justifyContent: "center",
    elevation: 8,
    shadowColor: "#000",
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.25,
    shadowRadius: 2,
  },
  buttonText: {
    color: "#fff",
    fontWeight: "700",
    textTransform: "uppercase",
  },
  buttonPressed: {
    opacity: 0.6,
  },
});
```

---

## Part 8: Centralising Colors (10 minutes)

The color `"#6741d9"` now appears in `Button.js`, and we are about to reuse it in the header too. If this app grows, hunting down every occurrence to change the theme becomes a real cost.

```jsx
// App.js
appHeader: {
  fontSize: 24,
  fontWeight: "700",
  marginBottom: 20,
  textAlign: "center",
  color: "#6741d9",
},
```

A codebase that stays manageable at a few hundred lines can become painful at a few thousand, precisely because of small repeated decisions like this one made without a shared source of truth. Let's fix it now, while it is cheap.

Create `styles/colors.js`:

```js
// styles/colors.js
export const Colors = {
  PRIMARY: "#6741d9",
  PRIMARY_LIGHT_1: "#f3f0ff",
  PRIMARY_LIGHT_2: "#e5dbff",
};
```

Update every place a raw color string is used - `Button.js`, the `appHeader` style, the `ActivityIndicator`, and the gradient:

```jsx
// components/Button.js
import { Colors } from "../styles/colors";

// inside StyleSheet.create:
buttonContainer: {
  backgroundColor: Colors.PRIMARY,
  // ...rest unchanged
},
```

```jsx
// App.js
import { Colors } from "./styles/colors";

appHeader: {
  fontSize: 24,
  fontWeight: "700",
  marginBottom: 20,
  textAlign: "center",
  color: Colors.PRIMARY,
},
```

```jsx
// App.js
<LinearGradient colors={[Colors.PRIMARY_LIGHT_2, Colors.PRIMARY_LIGHT_1]} style={{ flex: 1 }}>
```

### Extracting LoadingOverlay

While we are extracting shared pieces, pull the loading overlay into its own component too - `App.js` is getting crowded:

```jsx
// components/LoadingOverlay.js
import { View, ActivityIndicator, StyleSheet } from "react-native";
import { Colors } from "../styles/colors";

export default function LoadingOverlay() {
  return (
    <View style={styles.loadingOverlay}>
      <ActivityIndicator size="large" color={Colors.PRIMARY} />
    </View>
  );
}

const styles = StyleSheet.create({
  loadingOverlay: {
    ...StyleSheet.absoluteFill,
    justifyContent: "center",
    alignItems: "center",
  },
});
```

The spinner color is now `Colors.PRIMARY` instead of the earlier hardcoded gray, so the loading state matches the app's theme.

```jsx
// App.js
import LoadingOverlay from "./components/LoadingOverlay";

{isLoading && <LoadingOverlay />}
```

### Seeing the payoff

Every color in the app now flows from `styles/colors.js`. Try changing the theme from purple to orange:

```js
// styles/colors.js
export const Colors = {
  PRIMARY: "#e8590c",
  PRIMARY_LIGHT_1: "#fff4e6",
  PRIMARY_LIGHT_2: "#ffc078",
};
```

Reload the app - the header, buttons, gradient, and loading spinner should all update together, without touching `App.js`, `Button.js`, or `LoadingOverlay.js`. This is exactly the payoff of centralising colors: one change in one file instead of hunting through every component. Feel free to keep the orange theme or switch back to purple.

---

## Part 9: Custom Fonts (10 minutes)

Right now the app uses the system default font. A custom font goes a long way toward making an app feel designed rather than default.

Browse available fonts at the [Expo Google Fonts repository](https://github.com/expo/google-fonts), then install the one you want - this lesson uses Rubik:

```bash
npx expo install @expo-google-fonts/rubik expo-font
```

In `App.js`, use the `useFonts` hook to load the font weights you need. It returns a boolean telling you whether loading is complete - while it is `false`, render a loading state instead of the real UI:

```jsx
// App.js
import { Rubik_400Regular, Rubik_700Bold } from "@expo-google-fonts/rubik";
import { useFonts } from "expo-font";

export default function App() {
  const [fontsLoaded] = useFonts({
    Rubik_400Regular,
    Rubik_700Bold,
  });

  const [dogs, setDogs] = useState([]);
  // ...other state...

  if (!fontsLoaded) {
    return <ActivityIndicator size="large" color="#333" />;
  }

  // ...rest of the component
}
```

> **Hooks must run before any early return.** `useFonts` and every other hook needs to be called on every render, so the `if (!fontsLoaded)` check has to come after all your `useState`/`useRef` calls, never before.

Apply the fonts wherever `Text` is styled. Since `Rubik_700Bold` is already the bold variant of the font, drop `fontWeight` - it is redundant now that `fontFamily` points to a specific weighted file, and can even stop the correct font from being applied:

```jsx
// App.js
appHeader: {
  fontFamily: "Rubik_700Bold",
  fontSize: 24,
  marginBottom: 20,
  textAlign: "center",
  color: Colors.PRIMARY,
},
```

```jsx
// components/Button.js
buttonText: {
  fontFamily: "Rubik_700Bold",
  color: "#fff",
  textTransform: "uppercase",
},
```

Let's also add a line of body text so we can see the regular weight in use:

```jsx
// App.js
<Text style={styles.welcomeText}>👋 Welcome! Get a dog!</Text>
```

```jsx
// App.js
welcomeText: {
  fontFamily: "Rubik_400Regular",
  textAlign: "center",
  marginBottom: 10,
},
```

---

## Part 10: Background Image (6 minutes)

The plain gradient works, but a very subtle background pattern can add texture without competing with the dog photos.

A pattern image is already provided at `assets/images/wallpaper.jpg`. If you would rather use your own, search for something like "background pattern cute dog" on a site such as [Freepik](https://www.freepik.com/), download a light, seamless pattern, and place it in `assets/images/` instead.

`ImageBackground` renders an image behind its children. Wrap it around `SafeAreaView`, inside `LinearGradient`:

```jsx
// App.js
import { ImageBackground } from "react-native";

return (
  <SafeAreaProvider>
    <LinearGradient colors={[Colors.PRIMARY_LIGHT_2, Colors.PRIMARY_LIGHT_1]} style={{ flex: 1 }}>
      <ImageBackground
        source={require("./assets/images/wallpaper.jpg")}
        style={{ flex: 1 }}
        imageStyle={{ opacity: 0.3 }}
      >
        <SafeAreaView style={{ flex: 1 }}>
          {/* ...existing content... */}
        </SafeAreaView>
      </ImageBackground>
    </LinearGradient>
  </SafeAreaProvider>
);
```

`ImageBackground` takes a `source` prop for the image, a `style` prop for layout, and an `imageStyle` prop for styling the image itself. Keep `imageStyle={{ opacity: 0.03 }}` low - the pattern should be barely perceptible, not compete with the dog photos.

Reload the app and confirm the faint pattern is visible behind the gradient.

---

## Bonus Challenges

1. Add a counter below the header that displays how many dog photos have been loaded, for example "3 dogs loaded". Update it live as images are added and cleared.
2. Make each dog image span the full container width instead of a fixed 300, using `width: "100%"` and `aspectRatio: 1` on `dogImage` to keep it square.
3. Add an `onLongPress` handler to each `Image` (wrap it in a `Pressable`) that removes just that one dog from the list, with an `Alert` confirmation.
4. Extract the `FlatList` item renderer into a separate `DogCard` component that also shows the image index as a small badge in the corner.
5. Redo challenge 2 using `Dimensions.get("window").width` instead of `"100%"`, subtracting some horizontal padding to compute an explicit pixel width. Compare the two approaches - `Dimensions` gives you the actual pixel value to use in calculations, which matters when you need more than "fill the container" (for example, a non-square aspect ratio, or sizing something relative to the screen rather than its parent).

---

## Summary

- **`ScrollView` vs `FlatList`** - `ScrollView` renders all children immediately, which is fine for short fixed lists but wasteful for a list that grows without bound. `FlatList` virtualises rendering so only visible items stay in memory.
- **`SafeAreaProvider`/`SafeAreaView`** replace fragile fixed margins with insets that adapt to each device's notch, status bar, and home indicator.
- **`Pressable`** with a function-style `style` prop enables pressed-state feedback that works consistently on iOS and Android.
- **Component extraction** - `Button` and `LoadingOverlay` - keeps `App.js` focused on layout and data flow rather than styling details.
- **Color tokens** in `styles/colors.js` centralise the palette so changes propagate everywhere automatically.
- **`StyleSheet.absoluteFill`** is the idiomatic way to make an overlay fill its parent container.
- **`useFonts`** loads custom fonts asynchronously, returning a boolean you can use to hold back rendering until the fonts are ready.
- **`ImageBackground`** layers an image behind its children, useful for subtle background texture without an extra absolutely-positioned `Image`.

---

## Additional Resources

- [ScrollView - React Native docs](https://reactnative.dev/docs/scrollview)
- [FlatList - React Native docs](https://reactnative.dev/docs/flatlist)
- [Pressable - React Native docs](https://reactnative.dev/docs/pressable)
- [Alert - React Native docs](https://reactnative.dev/docs/alert)
- [expo-linear-gradient - Expo docs](https://docs.expo.dev/versions/latest/sdk/linear-gradient/)
- [expo-crypto - Expo docs](https://docs.expo.dev/versions/latest/sdk/crypto/)
- [expo-font and useFonts - Expo docs](https://docs.expo.dev/versions/latest/sdk/font/)
- [ImageBackground - React Native docs](https://reactnative.dev/docs/imagebackground)
- [Expo Google Fonts repository](https://github.com/expo/google-fonts)
