# 08. Mobile App (React Native + Expo)

**Project**: `C:\smart-eye-app`

## Overview

A React Native app built with Expo that reads detection results from Firebase  
in real-time and displays them on iPhone.

## Stack

| Tool | Purpose |
|------|---------|
| React Native | Cross-platform mobile framework |
| Expo (SDK 54) | Toolchain — enables testing via Expo Go without Mac |
| Firebase JS SDK | Realtime Database listener |
| Expo Go | iPhone app for development testing (no build needed) |

> Use SDK 54, not the latest. Expo Go on App Store may lag behind latest SDK.

## Setup

```bash
npx create-expo-app smart-eye-app   # select "For learning with Expo Go (SDK 54)"
cd smart-eye-app
npm install firebase
npx expo install expo-notifications
```

On Windows, use `npm.cmd` instead of `npm` if PowerShell shows "open with" dialog:
```bash
npm.cmd install firebase
```

## Firebase Config File (never commit)

```ts
// firebase.config.ts  ← listed in .gitignore
import { initializeApp } from "firebase/app";
import { getDatabase } from "firebase/database";

const firebaseConfig = {
  apiKey: "...",
  authDomain: "...",
  databaseURL: "https://<project>-default-rtdb.firebaseio.com",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "...",
};

const app = initializeApp(firebaseConfig);
export const database = getDatabase(app);
```

Get config values from: Firebase Console → Settings → General → Your apps → `</>` (Web).

## Home Screen (app/(tabs)/index.tsx)

```tsx
useEffect(() => {
  const detectionsRef = query(
    ref(database, "detections"),
    orderByKey(),
    limitToLast(1)           // only the latest detection
  );

  const unsubscribe = onValue(detectionsRef, (snapshot) => {
    if (snapshot.exists()) {
      const data = snapshot.val();
      const lastKey = Object.keys(data)[0];
      setLatest(data[lastKey]);   // triggers re-render
    }
  });

  return () => unsubscribe();    // cleanup on unmount
}, []);
```

`onValue` fires immediately with current data, then again on every change → real-time updates.

## Running the App

```bash
npx expo start       # shows QR code
# Scan with iPhone camera → opens in Expo Go
# If connection fails: npx expo start --tunnel
```

## App Display States

```
No data yet:               Waiting for data from ESP-EYE...

detected: false            🔍 Nothing Detected
                           Label: none  Confidence: 0.0%

detected: true             📦 Object Detected!
                           Label: box  Confidence: 87.5%
```

## Full Pipeline

```
ESP32 camera
  → FOMO inference (96×96, ~620ms)
    → firebase_send_detection() via HTTPS POST
      → Firebase Realtime Database
        → onValue() listener in React Native app
          → iPhone screen updates in real-time
```

## Current Limitation

The cloned Parcel Detection model does not reliably detect objects in a home environment  
because it was trained on someone else's data. To fix this, train a new model on Edge Impulse  
using images taken in your actual deployment location.
