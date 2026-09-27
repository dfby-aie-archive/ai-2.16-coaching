# Optional Exercise: The CRM, on Mobile

> This is an optional exercise, not part of the standard 2.16 session plan. Use it as an alternative to Dogstagram, as a fast-finisher extension after Dogstagram, or as independent practice outside class time.

## Overview

- **Duration:** ~1 hour
- **Prerequisites:** Lesson 2.14 - Introduction to Cross-Platform Mobile Application Development, Lesson 2.15 - UI Composition and Layout Management in Mobile Applications, Lesson 2.11 - Form Handling, Validation, and Deployment (for the MockAPI.io `customers` endpoint)

## Learning Objectives

By the end of this exercise, you will be able to:

1. **Fetch** data from an existing REST API inside a React Native app, reusing an endpoint built earlier in the course
2. **Render** a list of records with `FlatList`, including a `CustomerCard` component ported from the web CRM
3. **Perform** create and delete operations from a mobile UI, using a form and a confirmation `Alert`

## Introduction

Every lesson from 2.1 to 2.11 built the same web CRM. Lessons 2.14 and 2.15 introduced React Native's core components and layout system, but so far you have only used them on throwaway UI. This exercise connects the two: you will build a small mobile app that talks to the exact same MockAPI.io `customers` endpoint your web CRM already uses, and displays that data with native components instead of HTML.

There is no navigation library here and no routing. Everything happens on a single screen, with conditional rendering standing in for the "screens" a router would otherwise manage; adding a second screen for customer detail is left as a bonus challenge, ahead of the real navigation library you will meet in Lesson 2.17.

By the end you will have a `CustomerCard` component, a scrollable, virtualised customer list, a form to add a customer, and a swipe-free delete flow guarded by a confirmation dialog, the same set of CRUD operations the web CRM performs, expressed with `View`, `Text`, `TextInput`, `FlatList`, and `Alert` instead of `div`, `p`, `input`, `.map()`, and `window.confirm`.

---

## Setup (5 minutes)

Create the project and start the dev server:

```bash
npx create-expo-app --template blank mobile-crm
cd mobile-crm
npx expo start
```

> **Select SDK 57**, matching the Expo Go app on your phone or simulator, the same requirement as in the main Dogstagram lesson.

You will reuse the MockAPI.io project you created in Lesson 2.11. If you no longer have the base URL handy, it is in `src/App.jsx` of your web CRM project, on the line that defines `API_BASE`.

Create a `constants.js` file at the project root to hold it:

```js
// constants.js
export const API_BASE = "https://YOUR-PROJECT-ID.mockapi.io/api/v1";
```

> **This exercise skips `SafeAreaView` and `KeyboardAvoidingView` to keep the focus on data fetching and CRUD.** In a real app, both matter: `SafeAreaView` keeps content clear of the notch and status bar, the same problem solved in the "Fix the Notch" activity in the main Dogstagram lesson, and `KeyboardAvoidingView` stops the on-screen keyboard from covering the `TextInput`s in Part 2, particularly on smaller phones. Adding both is bonus challenge 6 below; do not skip them in an app you intend to actually use.

---

## Part 1: Fetching and Displaying Customers (20 minutes)

### State and the initial fetch

Open `App.js` and set up state for the customer list and a loading flag, then fetch on mount with `useEffect`:

```jsx
// App.js
import { useEffect, useState } from "react";
import { StyleSheet, Text, View, FlatList, ActivityIndicator } from "react-native";
import { API_BASE } from "./constants";

export default function App() {
  const [customers, setCustomers] = useState([]);
  const [isLoading, setIsLoading] = useState(true);

  const fetchCustomers = async () => {
    try {
      setIsLoading(true);
      const response = await fetch(`${API_BASE}/customers`);
      if (!response.ok) {
        throw new Error(`Request failed with status ${response.status}`);
      }
      const data = await response.json();
      setCustomers(data);
    } catch (error) {
      console.error(error);
    } finally {
      setIsLoading(false);
    }
  };

  useEffect(() => {
    fetchCustomers();
  }, []);

  return (
    <View style={styles.container}>
      <Text style={styles.header}>Customers</Text>
      {isLoading && <ActivityIndicator size="large" />}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    marginTop: 60,
    paddingHorizontal: 16,
  },
  header: {
    fontSize: 24,
    fontWeight: "700",
    marginBottom: 16,
  },
});
```

This is the same `fetch`-inside-`useEffect` pattern you used for the web CRM's `CustomerContext`; only the components rendering the result are different.

Run the app and confirm the spinner briefly appears while the request is in flight.

### Porting CustomerCard

The web CRM's `CustomerCard.jsx` renders a customer's name, email, phone, and status inside a styled `div`. React Native has no `div` or CSS module, but the same idea, a small presentational component that takes a `customer` prop, carries over directly.

Create a `components/` folder and add `CustomerCard.js`:

```jsx
// components/CustomerCard.js
import { View, Text, StyleSheet } from "react-native";

export default function CustomerCard({ customer }) {
  return (
    <View style={styles.card}>
      <Text style={styles.name}>
        {customer.firstName} {customer.lastName}
      </Text>
      <Text style={styles.email}>{customer.email}</Text>
      <Text>Phone: {customer.phone || "N/A"}</Text>
      <Text>Status: {customer.status}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  card: {
    borderWidth: 1,
    borderColor: "#ddd",
    borderRadius: 8,
    padding: 16,
    marginBottom: 12,
    backgroundColor: "#fff",
  },
  name: {
    fontSize: 17,
    fontWeight: "700",
    marginBottom: 4,
  },
  email: {
    color: "#555",
    marginBottom: 4,
  },
});
```

> The web version reads `customer.contactNo` and `customer.jobTitle`. Those fields belonged to an earlier draft of the mock data. By the time you set up MockAPI.io in Lesson 2.11, the real schema settled on `firstName`, `lastName`, `email`, `phone`, and `status`, so `CustomerCard` here reads those instead.

### Rendering the list

Replace the loading-only render with a `FlatList`, the same virtualised list component from the Dogstagram lesson:

```jsx
// App.js
import CustomerCard from "./components/CustomerCard";

// replacing the previous return:
return (
  <View style={styles.container}>
    <Text style={styles.header}>Customers</Text>
    {isLoading ? (
      <ActivityIndicator size="large" />
    ) : (
      <FlatList
        data={customers}
        keyExtractor={(customer) => customer.id}
        renderItem={({ item }) => <CustomerCard customer={item} />}
        ListEmptyComponent={<Text>No customers yet.</Text>}
      />
    )}
  </View>
);
```

**Browser check:** the app should show a spinner briefly, then a scrollable list of `CustomerCard`s pulled from the same MockAPI.io project the web CRM uses. If the list is empty, add a customer or two through the web app first, then reload.

---

## Part 2: Adding a Customer (20 minutes)

### A minimal form

Real form validation with Yup, as covered in Lesson 2.11, is out of scope here; this form only checks that the required fields are not blank. Add form state and two `TextInput`s above the list:

```jsx
// App.js
import { TextInput, Button } from "react-native";

// inside App, alongside the other state:
const [firstName, setFirstName] = useState("");
const [lastName, setLastName] = useState("");
const [email, setEmail] = useState("");
```

```jsx
// App.js
// inside container, below the header, above the FlatList:
<View style={styles.form}>
  <TextInput
    style={styles.input}
    placeholder="First name"
    value={firstName}
    onChangeText={setFirstName}
  />
  <TextInput
    style={styles.input}
    placeholder="Last name"
    value={lastName}
    onChangeText={setLastName}
  />
  <TextInput
    style={styles.input}
    placeholder="Email"
    value={email}
    onChangeText={setEmail}
    autoCapitalize="none"
    keyboardType="email-address"
  />
  <Button title="Add Customer" onPress={handleAddCustomer} />
</View>
```

```jsx
// App.js
form: {
  marginBottom: 20,
  gap: 8,
},
input: {
  borderWidth: 1,
  borderColor: "#ccc",
  borderRadius: 6,
  paddingHorizontal: 12,
  paddingVertical: 8,
},
```

`TextInput` is React Native's equivalent of an HTML `<input>`. Instead of an `onChange` event with a `target.value`, its `onChangeText` prop hands you the new string directly, which is why `setFirstName` can be passed straight in as the handler.

### Posting the new customer

Add `handleAddCustomer`, following the same `POST` pattern as `addCustomer` in the web CRM's `CustomerContext.jsx`:

```jsx
// App.js
import { Alert } from "react-native";

const handleAddCustomer = async () => {
  if (!firstName.trim() || !lastName.trim() || !email.trim()) {
    Alert.alert("Missing Information", "First name, last name, and email are all required.");
    return;
  }

  try {
    const response = await fetch(`${API_BASE}/customers`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        firstName,
        lastName,
        email,
        phone: "",
        status: "active",
        company: "",
        notes: "",
        tags: [],
      }),
    });
    if (!response.ok) {
      throw new Error(`Request failed with status ${response.status}`);
    }
    const newCustomer = await response.json();
    setCustomers((prevCustomers) => [...prevCustomers, newCustomer]);
    setFirstName("");
    setLastName("");
    setEmail("");
  } catch (error) {
    console.error(error);
    Alert.alert("Error", "Could not add customer. Please try again.");
  }
};
```

Just like `AddInteractionForm` in Lesson 2.11, the server's response already contains the saved record, server-assigned `id` included, so the new customer is appended straight into state instead of triggering a full refetch.

**Browser check:** fill in the three fields and tap "Add Customer." The new card should appear at the bottom of the list immediately, and the inputs should clear. Leave a field blank and tap the button; you should see the missing-information alert instead of a network request.

---

## Part 3: Deleting a Customer (15 minutes)

### Confirming before delete

Deleting is permanent, so confirm first with `Alert`, the same pattern used for "Clear" in Dogstagram. Add a `handleDeleteCustomer` function:

```jsx
// App.js
const handleDeleteCustomer = (customer) => {
  Alert.alert(
    "Delete Customer",
    `Are you sure you want to delete ${customer.firstName} ${customer.lastName}?`,
    [
      { text: "Cancel", style: "cancel" },
      {
        text: "Delete",
        style: "destructive",
        onPress: () => deleteCustomer(customer.id),
      },
    ]
  );
};

const deleteCustomer = async (customerId) => {
  try {
    const response = await fetch(`${API_BASE}/customers/${customerId}`, {
      method: "DELETE",
    });
    if (!response.ok) {
      throw new Error(`Request failed with status ${response.status}`);
    }
    setCustomers((prevCustomers) =>
      prevCustomers.filter((customer) => customer.id !== customerId)
    );
  } catch (error) {
    console.error(error);
    Alert.alert("Error", "Could not delete customer. Please try again.");
  }
};
```

`style: "destructive"` renders the delete option in red on iOS, the same semantic use of `style` you saw in the Dogstagram "Clear" confirmation.

### Wiring the delete action into CustomerCard

`CustomerCard` needs a way to trigger deletion without owning the confirmation logic itself. Pass `onDelete` down as a prop, and wrap the card in a `Pressable` for a delete button:

```jsx
// components/CustomerCard.js
import { View, Text, Pressable, StyleSheet } from "react-native";

export default function CustomerCard({ customer, onDelete }) {
  return (
    <View style={styles.card}>
      <Text style={styles.name}>
        {customer.firstName} {customer.lastName}
      </Text>
      <Text style={styles.email}>{customer.email}</Text>
      <Text>Phone: {customer.phone || "N/A"}</Text>
      <Text>Status: {customer.status}</Text>
      <Pressable onPress={onDelete} style={styles.deleteButton}>
        <Text style={styles.deleteText}>Delete</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  card: {
    borderWidth: 1,
    borderColor: "#ddd",
    borderRadius: 8,
    padding: 16,
    marginBottom: 12,
    backgroundColor: "#fff",
  },
  name: {
    fontSize: 17,
    fontWeight: "700",
    marginBottom: 4,
  },
  email: {
    color: "#555",
    marginBottom: 4,
  },
  deleteButton: {
    marginTop: 8,
    alignSelf: "flex-start",
  },
  deleteText: {
    color: "#e03131",
    fontWeight: "600",
  },
});
```

Pass the handler in from `App.js`:

```jsx
// App.js
<FlatList
  data={customers}
  keyExtractor={(customer) => customer.id}
  renderItem={({ item }) => (
    <CustomerCard
      customer={item}
      onDelete={() => handleDeleteCustomer(item)}
    />
  )}
  ListEmptyComponent={<Text>No customers yet.</Text>}
/>
```

Notice `onDelete` is wrapped in an inline arrow function inside `renderItem`, rather than passed as `handleDeleteCustomer` directly. `handleDeleteCustomer` needs to know which customer to delete, `item` here, so the wrapping function is what supplies that argument at render time.

**Browser check:** tap "Delete" on any card. A confirmation dialog should appear naming that specific customer. Tap "Cancel" and confirm nothing happens; tap "Delete" and confirm the card disappears from the list.

---

## Bonus Challenges

1. Add a pull-to-refresh gesture to the `FlatList` using its `refreshing` and `onRefresh` props, so dragging down re-fetches the customer list.
2. Add a second screen, without a navigation library, using conditional rendering: a boolean state like `showAddForm` that toggles between the list view and a full-screen add form.
3. Add a status filter above the list, two `Pressable` "chips" for "Active" and "Inactive" that filter `customers` before passing them to `FlatList`.
4. Style the "Status" text in `CustomerCard` conditionally, green for `"active"`, gray for `"inactive"`, using the pattern from Lesson 2.15 for conditional styles.
5. Replace the plain-text delete button with a confirmation-free swipe-to-delete gesture, using a community library such as `react-native-gesture-handler`'s `Swipeable`. Compare the tradeoffs against the `Alert`-based confirmation you built here.
6. Add `SafeAreaProvider`/`SafeAreaView` around the app, following the same pattern as the "Fix the Notch" activity in the Dogstagram lesson, then wrap the form in a `KeyboardAvoidingView` (`behavior="padding"` on iOS, `"height"` on Android) so the keyboard does not cover the `TextInput`s when adding a customer on a small screen.

---

## Summary

- The same MockAPI.io endpoint powers both the web CRM and this mobile app; only the rendering layer changes, from HTML and CSS to React Native's core components.
- `FlatList` plus a small presentational component (`CustomerCard`) is the mobile equivalent of `.map()` over a `div` list on the web.
- `TextInput`'s `onChangeText` prop replaces the web's `onChange` plus `event.target.value`.
- `Alert.alert` with a `"destructive"` style button is React Native's equivalent of a `window.confirm` before an irreversible action.
- Passing a handler like `onDelete` down as a prop, wrapped in an inline arrow function inside `renderItem`, is how a child component triggers a parent's logic while the parent retains the actual state update.

---

## Additional Resources

- [TextInput - React Native docs](https://reactnative.dev/docs/textinput)
- [FlatList - React Native docs](https://reactnative.dev/docs/flatlist)
- [Alert - React Native docs](https://reactnative.dev/docs/alert)
- [Pressable - React Native docs](https://reactnative.dev/docs/pressable)
