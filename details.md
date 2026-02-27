# Olauncher - Developer Guide

Welcome to the Olauncher codebase! This guide is tailored for developers (especially those new to Android/Java/Kotlin) who want to fork, run, build, and add features to this open-source minimal launcher.

## 1. Project Overview

**Olauncher** is a highly minimal, text-based, ad-free Android launcher. It replaces the default home screen with a clean interface featuring just a clock, date, and up to 8 pinned apps, alongside an app drawer accessed via swipe.

**Key Technologies:**
- **Language**: Kotlin (100% Kotlin codebase).
- **Build System**: Gradle.
- **Minimum Android Version**: Android 7.0 (API 24).
- **Architecture**: Single Activity Architecture using **Android Jetpack components** (Navigation component, ViewModel, ViewBinding, WorkManager).

---

## 2. Directory Structure: Where to Find What

The core of the application lives in `app/src/main/`.

### Source Code (`app/src/main/java/app/olauncher/`)
- **`MainActivity.kt`**: The single entry point of the app. It holds the `NavHostFragment` which switches between different screens. It also handles edge-case lifecycle events and intent resets.
- **`MainViewModel.kt`**: The central brain. It interacts with the Android `LauncherApps` API to load installed applications, pin them, and handle launching. It also sets up background workers.
- **`ui/` (User Interface)**
  - `HomeFragment.kt`: The main home screen. Displays the clock, date, and pinned apps. Handles all the swipe gestures (left, right, up, down).
  - `AppDrawerFragment.kt` / `AppDrawerAdapter.kt`: The screen you see when you swipe up. Shows the full list of installed apps and handles the search functionality.
  - `SettingsFragment.kt`: The settings menu, accessed by long-pressing the home screen.
- **`data/` (Data & Storage)**
  - `Prefs.kt`: A wrapper around `SharedPreferences`. This is where all user settings (e.g., pinned apps, theme, alignments) are saved locally.
  - `Constants.kt`: Static constant values, flags, and hardcoded variables used across the app.
  - `AppModel.kt`: The data class representing an installed application or shortcut.
- **`helper/` (Utilities)**
  - Contains various utility functions for theming, permissions, and device queries.
  - `WallpaperWorker.kt`: A WorkManager task that runs in the background to handle the daily plain wallpaper updates.

### Resources (`app/src/main/res/`)
- **`layout/`**: XML files defining the UI. E.g., `fragment_home.xml` (Home Screen), `fragment_app_drawer.xml` (App List).
- **`values/`**: Contains `strings.xml` (all text/translations), `colors.xml`, `styles.xml`, etc.
- **`navigation/nav_graph.xml`**: Defines how the app navigates between the Home, App Drawer, and Settings screens.

---

## 3. How to Run and Build the App

As an Android project, the best way to develop is using **Android Studio**, but you can also use the command line via Gradle wrapper.

### Prerequisites
- JDK 17 (Java Development Kit).
- Android SDK (Installed via Android Studio).

### Option A: Using Android Studio (Recommended)
1. Open Android Studio.
2. Select **File > Open** and point it to the root `Olauncher` directory.
3. Wait for Gradle to sync (it will download the necessary Android dependencies).
4. Connect your Android device via USB (with **USB Debugging** enabled) or start an Emulator.
5. Click the green **Play** button (Run 'app') in the top toolbar to build, install, and launch the app.

### Option B: Using Command Line (Gradle Wrapper)
To build a **Debug APK** (used for testing):
```bash
# On Windows
gradlew assembleDebug

# On Mac/Linux
./gradlew assembleDebug
```
The APK will be generated at: `app/build/outputs/apk/debug/app-debug.apk`.

To build a **Release Bundle (.aab)** (for Play Store):
```bash
./gradlew bundleRelease
```

---

## 4. How to Install the App

If you built the app via command line, you can install it on a connected device using ADB (Android Debug Bridge):

```bash
adb install app/build/outputs/apk/debug/app-debug.apk
```
*Note: Since it's a launcher, once installed, press your phone's Home button. Android will ask you which app to use as your Home app. Select "Olauncher".*

---

## 5. How to Add Features

Here is a step-by-step mental model for adding a new feature.

### Scenario: Adding a new user setting (e.g., "Hide Clock")

1. **Add the Preference Storage:**
   Go to `app/src/main/java/app/olauncher/data/Prefs.kt` and add a new boolean property to store the setting.
   ```kotlin
   var hideClock: Boolean
       get() = prefs.getBoolean("hide_clock", false)
       set(value) = prefs.edit().putBoolean("hide_clock", value).apply()
   ```

2. **Update the UI Layout:**
   Go to `app/src/main/res/layout/fragment_settings.xml`. Add a new switch or button for your setting. ViewBinding will automatically generate references to this view.

3. **Wire up the Settings Logic:**
   Go to `app/src/main/java/app/olauncher/ui/SettingsFragment.kt`. Set the initial state of your new switch and add an `setOnCheckedChangeListener` to save the new value into `Prefs`.
   ```kotlin
   binding.switchHideClock.isChecked = prefs.hideClock
   binding.switchHideClock.setOnCheckedChangeListener { _, isChecked ->
       prefs.hideClock = isChecked
       // Optionally tell the ViewModel to update the Home screen immediately
       viewModel.refreshHome(false) 
   }
   ```

4. **Implement the Feature Logic:**
   Go to `app/src/main/java/app/olauncher/ui/HomeFragment.kt`. Find where the clock is populated (e.g., in `populateDateTime()`). Check your new preference and update the UI accordingly.
   ```kotlin
   if (prefs.hideClock) {
       binding.clock.visibility = View.GONE
   } else {
       binding.clock.visibility = View.VISIBLE
   }
   ```

### Scenario: Modifying App Launching Behavior
If you want to change how apps are fetched or launched, you'll work inside `MainViewModel.kt`. Look for the `launchApp()` function. It uses the `LauncherApps` API to fetch intents and start activities.

### Scenario: Modifying Gestures
If you want to add a custom swipe gesture (e.g., diagonal swipe), check `getSwipeGestureListener()` in `HomeFragment.kt`. It utilizes the custom `OnSwipeTouchListener.kt` found in the `listener/` package.

---

## Tips for Kotlin & Android Beginners
1. **ViewBinding**: You'll see `binding.clock.setOnClickListener...`. Instead of old-school `findViewById`, Android generates a `FragmentHomeBinding` class that contains direct references to all IDs in the corresponding XML file.
2. **Context**: In Android, accessing system services (like battery or packages) requires a `Context`. In an Activity, the Activity itself is the context (`this`). In a Fragment, use `requireContext()`.
3. **Coroutines**: You'll see `viewModelScope.launch { }` in `MainViewModel`. This is Kotlin's way of doing background threads (like fetching the app list without freezing the UI).
4. **LiveData/State**: The ViewModel holds `MutableLiveData`. Fragments `observe()` these variables. When the ViewModel updates the LiveData, the Fragment automatically reacts. This keeps the logic separated from the UI.

---

## 6. CI/CD (GitHub Actions)

We have set up two GitHub Actions workflows to automate your build and release process.

### 1. Automatic Debug Build
- **File**: `.github/workflows/build-debug.yml`
- **Trigger**: Runs automatically whenever you push code to the `main` branch.
- **Action**: It builds the debug APK and uploads it as a build artifact. You can download this APK from the "Actions" tab in your GitHub repository to test your changes.

### 2. Manual Release to Play Store
- **File**: `.github/workflows/release-app.yml`
- **Trigger**: Manual. You must go to the "Actions" tab, select "Release to Play Store", and click "Run workflow".
- **Action**: It builds a release bundle (.aab), signs it, and uploads it to the Google Play Console (Draft status).

### Required Secrets for Release
To make the release workflow work, you need to add the following **Secrets** in your GitHub Repository settings (**Settings > Secrets and variables > Actions**):

| Secret Name | Description |
|---|---|
| `SIGNING_KEY_STORE_BASE64` | Your Keystore file encoded in Base64. <br> Run this command to generate it: `base64 -i path/to/your.keystore -o keystore_base64.txt` (on Mac/Linux) or use an online tool (be careful with private keys!). Copy the content of the text file. |
| `SIGNING_KEY_ALIAS` | The alias you used when creating your keystore. |
| `SIGNING_KEY_PASSWORD` | The password for the key alias. |
| `SIGNING_STORE_PASSWORD` | The password for the keystore file. |
| `PLAY_SERVICE_ACCOUNT_JSON` | The entire content of your Google Play Service Account JSON key file. You get this from the Google Cloud Console linked to your Play Console. |

**Important Note**: The release workflow is set to upload as a **Draft**. This allows you to review the release in the Play Console before rolling it out to users.
