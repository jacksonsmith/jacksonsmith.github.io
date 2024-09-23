---
layout: post
title: End-to-End Testing in React Native with Maestro A Comprehensive Guide Android
---

Here's the updated article with the additional section:

---

# End-to-End Testing in React Native with Maestro: A Comprehensive Guide Part 2

In the first part of this guide, we explored how to set up and execute end-to-end (E2E) tests in React Native using Maestro, focusing primarily on iOS. Following the positive feedback and requests from readers, this second part will cover how to set up and run E2E tests for Android using Maestro. We'll walk through the setup process, including the necessary npm scripts for building, installing, and running the tests on Android.

If you're interested in learning about the iOS version, please refer to the first part of this guide [here](https://medium.com/@3jacksonsmith/end-to-end-testing-in-react-native-with-maestro-a-comprehensive-guide-c644bbb71ed8).

## Apologies for the Delay

First of all, I want to apologize for the delay in bringing you this second part. I found a small job in between setting up cribs for my son, who will be arriving soon, and dealing with COVID-19, which I contracted just a couple of days ago. But now that I'm feeling better, I've returned to writing. I hope to share more of what I've been studying soon as well.

## Why Android?

While iOS testing is crucial, Android dominates a significant portion of the mobile market, especially in regions like Latin America, Asia, and parts of Europe. Ensuring that your React Native app works flawlessly on Android is just as important as on iOS, if not more so, depending on your target audience. With Maestro, running E2E tests on Android is streamlined and efficient.

## Setting Up Your Android E2E Testing Environment

Before diving into the test scripts and workflows, let's ensure your environment is ready for Android E2E testing.

### 1. Setting Up Android Dependencies

Ensure you have the necessary dependencies installed:

- **Android Studio**: For managing Android SDKs and emulators.
- **Android SDK**: Ensure you have the SDKs for the Android versions you intend to test.
- **Java Development Kit (JDK)**: Required for building and running Android apps.
- **Maestro**: Installed globally, as we did in the first part.

Install Maestro using:

```bash
curl -Ls "https://get.maestro.mobile.dev" | bash
```

### 2. Configuring npm Scripts for Android

To make your E2E testing workflow seamless, we'll add a few npm scripts to your `package.json`. These scripts handle the build, installation, and execution of tests on Android devices or emulators.

```json
{
  "scripts": {
    "e2e:android:build": "cd android && ./gradlew assembleDebug",
    "e2e:android:install": "adb install android/app/build/outputs/apk/debug/app-debug.apk",
    "e2e:android:run": "maestro test ./e2e/android/android-flow.yaml"
  }
}
```

**Here's a breakdown of what each script does:**

- **`e2e:android:build`**: Builds the Android app in debug mode, preparing it for testing.
- **`e2e:android:install`**: Installs the debug APK onto a connected Android device or emulator.
- **`e2e:android:run`**: Executes the E2E tests using Maestro on Android.

## Writing the Maestro Workflow for Android

With the environment set up and the npm scripts in place, let's create the Maestro workflow for Android. Below is a sample workflow:

```yaml
appId: com.awesomeproject
---
steps:
  - launchApp

  - tapOn:
      text: "Step One"

  - assertVisible:
      text: "Edit App.tsx to change this screen and then come back to see your edits."

  - assertVisible:
      text: "Debug" # Verifying the text within DebugInstructions is visible

  - assertVisible:
      text: "Read the docs to discover what to do next:"

  - scrollUntilVisible:
      element:
        text: "Follow us on Twitter"
```

## Running Your Android E2E Tests

Once everything is set up, you can easily run your Android E2E tests with the following commands:

### 1. Build the Android App:

```bash
npm run e2e:android:build
```

This will compile your Android app in debug mode.

### 2. Install the APK on a Device or Emulator:

```bash
npm run e2e:android:install
```

Ensure your Android device or emulator is connected and recognized by `adb` (Android Debug Bridge).

### 3. Execute the E2E Tests:

```bash
npm run e2e:android:run
```

This command runs the Maestro test workflow you defined for Android.

## Conclusion

With these steps, you can now confidently run end-to-end tests on your Android application using Maestro. This not only helps in catching bugs early but also ensures a consistent user experience across both major mobile platforms. In the next part of this guide, we might explore advanced workflows and integrating these tests into a CI/CD pipeline.

Stay tuned, and happy testing!

---

This article expands upon the previous guide by introducing Android into your E2E testing strategy, fulfilling the requests of our dedicated readers. Your feedback is invaluable—if there's anything else you'd like to see covered, let me know in the comments below!
