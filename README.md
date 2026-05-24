# 2.16 Building a Sample App I - Dogstagram

## Lesson Overview

A coaching session that consolidates the React Native core components, styling, and flex layout concepts taught in Lessons 2.14 and 2.15. Learners build Dogstagram - a dog image gallery app - from scratch in a single session. The emphasis is on applying familiar concepts fluently rather than introducing new theory.

## Dependencies

- [Lesson](./lesson.md)

## Lesson Objectives

- Build a complete single-screen React Native app using `FlatList`, `Pressable`, `Alert`, `SafeAreaView`, and `ActivityIndicator`
- Extract reusable components (`Button`, `LoadingOverlay`) and a shared color token file to organise the codebase
- Apply `expo-linear-gradient` and `expo-crypto` as real-world Expo library integrations

## Lesson Plan

| Duration | What | How or Why |
| --------- | ---------------------------------- | ------------------------------------------------------------------------------ |
| 30 min | Lecture and app demo | Recap 2.14 and 2.15; show finished Dogstagram; explain FlatList vs ScrollView, component decomposition, and what comes next in 2.17 |
| 5 min | Setup | Scaffold Expo project, install axios, expo-linear-gradient, expo-crypto, react-native-safe-area-context |
| 25 min | Part 1: App shell and data fetching | SafeAreaView wrapper, header, state, getDog async function with expo-crypto UUID |
| 25 min | Part 2: FlatList | Render images, empty state, scroll-to-end with useRef |
| 10 min | Activity 1 | Wire up buttons; add Alert confirmation before clearing |
| 20 min | Part 3: Custom Button component | Pressable with pressed-state feedback, platform shadows, children prop |
| 15 min | Part 4: LoadingOverlay and color tokens | StyleSheet.absoluteFillObject, ActivityIndicator, styles/colors.js |
| 10 min | Activity 2 | Gradient background with expo-linear-gradient |
| 10 min | Wrap up and Q&A | Recap, bonus challenges for fast finishers |
| **Total** | | **~150 min - allows a buffer for questions and pacing** |
