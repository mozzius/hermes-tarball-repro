# hermes-tarball-repro

Since `facebook/hermes` moved to `meta/hermes` (2026-10-28), Android builds that compile React Native from source fail at `configureBuildForHermes`.

## What happens

- `ReactAndroid/hermes-engine/build.gradle.kts` downloads `https://github.com/facebook/hermes/tarball/<tag>`. That URL now redirects (301) to `meta/hermes`.
- GitHub names a tarball's top-level folder `<owner>-<repo>-<sha>`. So it is now `meta-hermes-<sha>/` instead of `facebook-hermes-<sha>/`:

  ```
  $ curl -sL https://github.com/facebook/hermes/tarball/hermes-v260318099.0.2 | tar -tz | head -1
  meta-hermes-4188b63/
  ```

- `unzipHermes` only flattens entries that start with `facebook-hermes-`. So the Hermes source lands one folder too deep, in `sdks/hermes/meta-hermes-<sha>/`, and CMake can't find it.

## Reproduce

This is the [community reproducer template](https://github.com/react-native-community/reproducer-react-native) (React Native 0.88.1) with one change: `ReproducerApp/android/settings.gradle` builds React Native from source, which apps that patch React Native's native code do (for example via Expo's `buildReactNativeFromSource`).

```sh
cd ReproducerApp
yarn
cd android
./gradlew :app:assembleDebug -PreactNativeArchitectures=arm64-v8a
```

It fails within about 30 seconds:

```
> Task :react-native:packages:react-native:ReactAndroid:hermes-engine:configureBuildForHermes FAILED
CMake Error: The source directory ".../ReproducerApp/node_modules/react-native/sdks/hermes" does not appear to contain CMakeLists.txt.
```

Use a fresh `node_modules`. A checkout that already has a flattened `sdks/hermes` from before the move can keep building.

## Fix

The [`fixed`](../../tree/fixed) branch applies the proposed change with patch-package. It strips the tarball's single top-level folder whatever its name, like the folly, fmt and fast_float download tasks in `ReactAndroid/build.gradle.kts` already do:

```diff
       from(tarTree(downloadHermesDest)) {
         eachFile {
-          // We flatten the unzip as the tarball contains a `facebook-hermes-<SHA>`
-          // folder at the top level.
-          if (this.path.startsWith("facebook-hermes-")) {
-            this.path = this.path.substringAfter("/")
-          }
+          // We flatten the unzip as the tarball contains a single
+          // `<owner>-hermes-<SHA>` folder at the top level. Strip it regardless
+          // of the owner, which changes when the repository moves (it went
+          // from `facebook-hermes-` to `meta-hermes-`).
+          this.path = this.path.substringAfter("/")
         }
       }
```

## Filed as

- React Native: [react/react-native#59587](https://github.com/react/react-native/issues/59587)
- Fix: [react/react-native#59588](https://github.com/react/react-native/pull/59588) (the `fixed` branch of this repo applies its `unzipHermes` change)

## Environment

Reproduced on macOS 27 (arm64) with JDK 17 and React Native 0.88.1. The failing step is the Gradle download-and-extract logic, which doesn't depend on the host OS.
