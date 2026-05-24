# Lesson 2.16: Building a Sample App I - Dogstagram

## Overview

- **Duration:** ~2 hours (hands-on lab)
- **Prerequisites:** Lesson 2.15 - UI Composition and Layout Management in Mobile Applications

## Learning Objectives

By the end of this lesson, you will be able to:

1. **Build** a complete React Native app from scratch using core components, styling, and flex layout
2. **Extract** reusable components (`Button`, `LoadingOverlay`) and a shared color token file
3. **Apply** `FlatList`, `Alert`, `Pressable`, and `expo-linear-gradient` to solve real UI problems

## Introduction

In this session you will build Dogstagram - a dog image gallery app that fetches random dog photos from a public API, displays them in a scrollable list, and lets you add and clear photos. There is no new theory today; the goal is fluency. Every concept you use here - core components, `StyleSheet`, flexbox, `useState`, `useEffect`, `useRef`, and component decomposition - was introduced in 2.14 and 2.15. Working through a complete app from scratch is how those pieces connect into a mental model you can apply to any mobile project.

Lesson 2.17 will add navigation so Dogstagram can grow into a multi-screen experience.

---

## Setup

Create the project and install dependencies:

```bash
npx create-expo-app --template blank dogstagram
cd dogstagram
npm i axios
npx expo install expo-linear-gradient expo-crypto react-native-safe-area-context
npx expo start
```

Start the iOS or Android simulator (or open Expo Go on your phone) and confirm the default app is running before continuing.

---

## Part 1: App Shell and Data Fetching

### Creating the app container

Open `App.js` and replace the entire file with the following shell. This sets up the safe area wrapper, the header, and the two state variables the app needs.

```jsx
import { useState, useRef } from "react";
import { View, Text, FlatList, Image, StyleSheet, Alert } from "react-native";
import { SafeAreaProvider, SafeAreaView } from "react-native-safe-area-context";
import * as Crypto from "expo-crypto";
import axios from "axios";

const API_URL = "https://dog.ceo/api/breeds/image/random";

export default function App() {
  const [dogs, setDogs] = useState([]);
  const [isLoading, setIsLoading] = useState(false);

  return (
    <SafeAreaProvider>
      <SafeAreaView style={styles.container}>
        <Text style={styles.appHeader}>🐶 Dogstagram</Text>
      </SafeAreaView>
    </SafeAreaProvider>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: "center",
    paddingHorizontal: 16,
  },
  appHeader: {
    fontSize: 24,
    fontWeight: "700",
    textAlign: "center",
    marginBottom: 20,
  },
});
```

Check that the header appears on screen and the app does not crash.

> **Why `SafeAreaProvider` and `SafeAreaView`?** Modern phones have a notch, Dynamic Island, or home indicator that can overlap content. `SafeAreaProvider` detects device-specific insets and `SafeAreaView` applies them as padding automatically - much more reliable than a fixed `marginTop`.

### Adding the fetch function

Add the `getDog` function inside the `App` component, above the `return`:

```jsx
const getDog = async () => {
  try {
    setIsLoading(true);
    const response = await axios.get(API_URL);
    setDogs((prevDogs) => [
      ...prevDogs,
      { id: Crypto.randomUUID(), url: response.data.message },
    ]);
  } catch (error) {
    console.error(error);
  } finally {
    setIsLoading(false);
  }
};
```

> **Why `Crypto.randomUUID()` instead of the URL as a key?** The dog API occasionally returns the same URL for consecutive requests. Using the URL as a list key would produce duplicate key warnings and cause React to misidentify which item changed. A UUID generated per call is always unique regardless of what the API returns.

Press "Get Dog" in the terminal - nothing visible will happen yet, but add a temporary `<Text>{dogs.length} dogs</Text>` line after the header to confirm state is updating. Remove it once confirmed.

---

## Part 2: Displaying Images with FlatList

### Why FlatList instead of ScrollView

`ScrollView` renders all its children into the component tree at once. For a list that grows without bound - like Dogstagram, where each button press adds another image - this means every image is mounted and kept in memory whether or not it is on screen. `FlatList` only renders items currently visible in the viewport, discarding off-screen items from memory as the user scrolls.

For a fixed, short list `ScrollView` is fine. For a list that can grow large, use `FlatList`.

### Adding FlatList

Add the `FlatList` and a buttons area inside the `SafeAreaView`, below the header:

```jsx
<View style={styles.buttonsContainer}>
  {/* Buttons will go here in Part 3 */}
</View>

<FlatList
  data={dogs}
  keyExtractor={(dog) => dog.id}
  renderItem={({ item }) => (
    <Image source={{ uri: item.url }} style={styles.dogImage} />
  )}
  showsVerticalScrollIndicator={false}
  ListEmptyComponent={
    <Text style={styles.emptyText}>No dogs yet. Tap Get Dog!</Text>
  }
/>
```

Add the new styles to `StyleSheet.create`:

```jsx
buttonsContainer: {
  flexDirection: "row",
  gap: 15,
  justifyContent: "center",
  marginBottom: 20,
},
dogImage: {
  width: 300,
  height: 300,
  borderRadius: 10,
  marginBottom: 20,
},
emptyText: {
  textAlign: "center",
  color: "#888",
  marginTop: 40,
},
```

> **Common mistake:** `FlatList` requires a bounded height to scroll correctly. Because `SafeAreaView` has `flex: 1`, the `FlatList` inherits a bounded container. If you placed `FlatList` inside a `View` without a flex constraint, it would collapse to zero height and show nothing.

### Scrolling to the newest image

When a new dog is added, it appears at the bottom of the list. Add a `ref` to automatically scroll there:

```jsx
const flatListRef = useRef(null);
```

Add `ref` and `onContentSizeChange` to the `FlatList`:

```jsx
<FlatList
  ref={flatListRef}
  onContentSizeChange={() =>
    flatListRef.current?.scrollToEnd({ animated: true })
  }
  data={dogs}
  keyExtractor={(dog) => dog.id}
  renderItem={({ item }) => (
    <Image source={{ uri: item.url }} style={styles.dogImage} />
  )}
  showsVerticalScrollIndicator={false}
  ListEmptyComponent={
    <Text style={styles.emptyText}>No dogs yet. Tap Get Dog!</Text>
  }
/>
```

`onContentSizeChange` fires whenever the rendered content changes height - including when a new item is added - so the scroll runs immediately after the image appears.

---

## Activity 1: Add an Alert Before Clearing

Right now there are no buttons wired up. Add two placeholder buttons using React Native's built-in `Button` component (the custom one comes in Part 3) and update the clear action to show a confirmation alert.

**Task:** Wire up a "Get Dog" button and a "Clear" button. The "Clear" button must show a confirmation `Alert` before resetting the list. If the list is already empty, show a different message instead of the confirmation.

**Hints:**

1. Import `Button` and `Alert` from `"react-native"`.
2. Place two `<Button>` elements inside the `buttonsContainer` view.
3. `Alert.alert(title, message, buttonsArray)` - the `buttonsArray` contains objects with `text` and `onPress` properties.
4. Check `dogs.length === 0` before showing the confirmation.

<details>
<summary>Reference solution</summary>

Add to your imports:

```jsx
import { View, Text, FlatList, Image, StyleSheet, Alert, Button } from "react-native";
```

Add the handler function inside `App`, after `getDog`:

```jsx
const handleClearDogs = () => {
  if (dogs.length === 0) {
    Alert.alert("Clear Dogs", "No dogs to clear.", [{ text: "OK" }]);
    return;
  }
  Alert.alert(
    "Clear Dogs",
    "Are you sure you want to clear all dogs?",
    [
      { text: "Cancel" },
      { text: "OK", onPress: () => setDogs([]) },
    ]
  );
};
```

Update the buttons container in your JSX:

```jsx
<View style={styles.buttonsContainer}>
  <Button title="Get Dog" onPress={getDog} />
  <Button title="Clear" onPress={handleClearDogs} />
</View>
```

</details>

Test that pressing "Get Dog" loads an image, pressing "Clear" shows the confirmation, and pressing "OK" on the alert removes all images.

---

## Part 3: Custom Button Component

The built-in `Button` looks different on iOS and Android and offers limited styling control. A custom component using `Pressable` gives you consistent appearance on both platforms and lets you add pressed-state feedback.

### Creating the component

Create a `components/` folder and add `Button.js`:

```jsx
import { Pressable, Text, StyleSheet } from "react-native";

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

export default Button;
```

Two things to note here:

**The `style` prop as a function.** `Pressable` passes an object with a `pressed` boolean into the `style` prop when you provide a function instead of a plain object. The function returns an array of style objects - React Native merges them in order. When `pressed` is `true`, `styles.buttonPressed` is included; when `false`, the `&&` short-circuits to `false`, which React Native ignores.

**Platform shadows.** `elevation` adds a shadow on Android. The `shadowColor`, `shadowOffset`, `shadowOpacity`, and `shadowRadius` properties add a shadow on iOS. Both are needed for consistent depth across platforms.

### Using the custom Button

In `App.js`, remove the `Button` import from `"react-native"` and replace it with your component:

```jsx
import Button from "./components/Button";
```

Update the buttons container - pass the label as children, not a `title` prop:

```jsx
<View style={styles.buttonsContainer}>
  <Button onPress={getDog}>Get Dog</Button>
  <Button onPress={handleClearDogs}>Clear</Button>
</View>
```

Verify both buttons still work. Press each one and observe the opacity feedback on press.

---

## Part 4: Loading Overlay and Color Tokens

### Centralising colors

Right now `"#6741d9"` appears in `Button.js`. As the app grows, this value will appear in multiple components and updating it means editing every file. A dedicated color file solves this.

Create `styles/colors.js`:

```js
export const Colors = {
  PRIMARY: "#6741d9",
  PRIMARY_LIGHT_1: "#f3f0ff",
  PRIMARY_LIGHT_2: "#e5dbff",
};
```

Update `components/Button.js` to import and use it:

```jsx
import { Colors } from "../styles/colors";

// inside StyleSheet.create:
buttonContainer: {
  backgroundColor: Colors.PRIMARY,
  // ... rest unchanged
},
```

### Creating LoadingOverlay

Create `components/LoadingOverlay.js`:

```jsx
import { View, ActivityIndicator, StyleSheet } from "react-native";
import { Colors } from "../styles/colors";

function LoadingOverlay() {
  return (
    <View style={styles.loadingOverlay}>
      <ActivityIndicator size="large" color={Colors.PRIMARY} />
    </View>
  );
}

const styles = StyleSheet.create({
  loadingOverlay: {
    ...StyleSheet.absoluteFillObject,
    justifyContent: "center",
    alignItems: "center",
    backgroundColor: "rgba(255,255,255,0.7)",
  },
});

export default LoadingOverlay;
```

> **`StyleSheet.absoluteFillObject`** expands to `{ position: "absolute", top: 0, right: 0, bottom: 0, left: 0 }`. It makes the overlay fill its parent entirely regardless of the parent's size. The spread `...` merges it into the style object alongside the other properties.

### Using LoadingOverlay in App

Import the component and render it conditionally inside `SafeAreaView`:

```jsx
import LoadingOverlay from "./components/LoadingOverlay";

// Inside SafeAreaView, after FlatList:
{isLoading && <LoadingOverlay />}
```

Press "Get Dog" and observe the spinner appearing over the content while the image loads.

---

## Activity 2: Add a Gradient Background

The app currently has a plain white background. Replace it with a soft gradient using `expo-linear-gradient`.

**Task:** Wrap the `SafeAreaProvider` in a `LinearGradient` component that uses `Colors.PRIMARY_LIGHT_2` and `Colors.PRIMARY_LIGHT_1` (light purple tones) and spans the full screen.

**Hints:**

1. Import `LinearGradient` from `"expo-linear-gradient"`.
2. `LinearGradient` takes a `colors` prop (an array of color strings) and a `style` prop.
3. Give the `LinearGradient` `style={{ flex: 1 }}` so it fills the screen.
4. The `colors` array goes from the starting color to the ending color of the gradient.
5. Remove `backgroundColor` from `styles.container` if you set one - it will cover the gradient.

<details>
<summary>Reference solution</summary>

Add to your imports in `App.js`:

```jsx
import { LinearGradient } from "expo-linear-gradient";
import { Colors } from "./styles/colors";
```

Wrap the top-level JSX:

```jsx
return (
  <LinearGradient
    colors={[Colors.PRIMARY_LIGHT_2, Colors.PRIMARY_LIGHT_1]}
    style={{ flex: 1 }}
  >
    <SafeAreaProvider>
      <SafeAreaView style={styles.container}>
        {/* ... */}
      </SafeAreaView>
    </SafeAreaProvider>
  </LinearGradient>
);
```

Remove `backgroundColor` from `styles.container` if present.

</details>

---

## Final Code Reference

At this point your project structure should look like this:

```
dogstagram/
├── App.js
├── components/
│   ├── Button.js
│   └── LoadingOverlay.js
└── styles/
    └── colors.js
```

**`App.js`** - final version:

```jsx
import { useState, useRef } from "react";
import { View, Text, FlatList, Image, StyleSheet, Alert } from "react-native";
import { SafeAreaProvider, SafeAreaView } from "react-native-safe-area-context";
import { LinearGradient } from "expo-linear-gradient";
import * as Crypto from "expo-crypto";
import axios from "axios";

import Button from "./components/Button";
import LoadingOverlay from "./components/LoadingOverlay";
import { Colors } from "./styles/colors";

const API_URL = "https://dog.ceo/api/breeds/image/random";

export default function App() {
  const [dogs, setDogs] = useState([]);
  const [isLoading, setIsLoading] = useState(false);
  const flatListRef = useRef(null);

  const getDog = async () => {
    try {
      setIsLoading(true);
      const response = await axios.get(API_URL);
      setDogs((prevDogs) => [
        ...prevDogs,
        { id: Crypto.randomUUID(), url: response.data.message },
      ]);
    } catch (error) {
      console.error(error);
    } finally {
      setIsLoading(false);
    }
  };

  const handleClearDogs = () => {
    if (dogs.length === 0) {
      Alert.alert("Clear Dogs", "No dogs to clear.", [{ text: "OK" }]);
      return;
    }
    Alert.alert(
      "Clear Dogs",
      "Are you sure you want to clear all dogs?",
      [
        { text: "Cancel" },
        { text: "OK", onPress: () => setDogs([]) },
      ]
    );
  };

  return (
    <LinearGradient
      colors={[Colors.PRIMARY_LIGHT_2, Colors.PRIMARY_LIGHT_1]}
      style={{ flex: 1 }}
    >
      <SafeAreaProvider>
        <SafeAreaView style={styles.container}>
          <Text style={styles.appHeader}>🐶 Dogstagram</Text>
          <View style={styles.buttonsContainer}>
            <Button onPress={getDog}>Get Dog</Button>
            <Button onPress={handleClearDogs}>Clear</Button>
          </View>
          <FlatList
            ref={flatListRef}
            onContentSizeChange={() =>
              flatListRef.current?.scrollToEnd({ animated: true })
            }
            data={dogs}
            keyExtractor={(dog) => dog.id}
            renderItem={({ item }) => (
              <Image source={{ uri: item.url }} style={styles.dogImage} />
            )}
            showsVerticalScrollIndicator={false}
            ListEmptyComponent={
              <Text style={styles.emptyText}>No dogs yet. Tap Get Dog!</Text>
            }
          />
          {isLoading && <LoadingOverlay />}
        </SafeAreaView>
      </SafeAreaProvider>
    </LinearGradient>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: "center",
    paddingHorizontal: 16,
  },
  appHeader: {
    fontSize: 24,
    fontWeight: "700",
    textAlign: "center",
    marginBottom: 20,
  },
  buttonsContainer: {
    flexDirection: "row",
    gap: 15,
    justifyContent: "center",
    marginBottom: 20,
  },
  dogImage: {
    width: 300,
    height: 300,
    borderRadius: 10,
    marginBottom: 20,
  },
  emptyText: {
    textAlign: "center",
    color: "#888",
    marginTop: 40,
  },
});
```

**`components/Button.js`:**

```jsx
import { Pressable, Text, StyleSheet } from "react-native";
import { Colors } from "../styles/colors";

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

const styles = StyleSheet.create({
  buttonContainer: {
    backgroundColor: Colors.PRIMARY,
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

export default Button;
```

**`components/LoadingOverlay.js`:**

```jsx
import { View, ActivityIndicator, StyleSheet } from "react-native";
import { Colors } from "../styles/colors";

function LoadingOverlay() {
  return (
    <View style={styles.loadingOverlay}>
      <ActivityIndicator size="large" color={Colors.PRIMARY} />
    </View>
  );
}

const styles = StyleSheet.create({
  loadingOverlay: {
    ...StyleSheet.absoluteFillObject,
    justifyContent: "center",
    alignItems: "center",
    backgroundColor: "rgba(255,255,255,0.7)",
  },
});

export default LoadingOverlay;
```

**`styles/colors.js`:**

```js
export const Colors = {
  PRIMARY: "#6741d9",
  PRIMARY_LIGHT_1: "#f3f0ff",
  PRIMARY_LIGHT_2: "#e5dbff",
};
```

---

## Bonus Challenges

1. Add a counter below the header that displays how many dog photos have been loaded, for example "3 dogs loaded". Update it live as images are added and cleared.
2. Use `Dimensions.get("window").width` from `"react-native"` to make each dog image span the full screen width minus horizontal padding, rather than a fixed 300.
3. Add an `onLongPress` handler to each `Image` (wrap it in a `Pressable`) that removes just that one dog from the list, with an `Alert` confirmation.
4. Extract the `FlatList` item renderer into a separate `DogCard` component that also shows the image index as a small badge in the corner.

---

## Summary

- **`FlatList`** virtualises a list so only visible items are rendered, making it suitable for lists that grow without bound.
- **`Pressable`** with a function-style `style` prop enables pressed-state feedback that works consistently on iOS and Android.
- **Component extraction** - `Button` and `LoadingOverlay` - keeps `App.js` focused on layout and data flow rather than styling details.
- **Color tokens** in `styles/colors.js` centralise the palette so changes propagate everywhere automatically.
- **`StyleSheet.absoluteFillObject`** is the idiomatic way to make an overlay fill its parent container.

---

## Additional Resources

- [FlatList - React Native docs](https://reactnative.dev/docs/flatlist)
- [Pressable - React Native docs](https://reactnative.dev/docs/pressable)
- [Alert - React Native docs](https://reactnative.dev/docs/alert)
- [expo-linear-gradient - Expo docs](https://docs.expo.dev/versions/latest/sdk/linear-gradient/)
- [expo-crypto - Expo docs](https://docs.expo.dev/versions/latest/sdk/crypto/)
