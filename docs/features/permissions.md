---
sidebar_position: 20
---

# Runtime Permissions

KMPStarterKit ships a small, app-level permissions API backed by [Calf](https://github.com/MohamedRejeb/Calf). It shows real system permission dialogs on Android and iOS, and acts as a granted no-op on desktop and web — so shared code never needs platform checks.

### Using a permission

Each common permission has a ready-made helper composable:

```kotlin
@Composable
fun MyScreen() {
    val cameraPermission = rememberCameraPermissionState { granted ->
        // called with the result after request()
    }

    when {
        cameraPermission.isGranted -> CameraPreview()
        cameraPermission.shouldShowRationale -> RationaleCard(
            onAllow = { cameraPermission.openSettings() },
        )
        else -> Button(onClick = { cameraPermission.request() }) {
            Text("Enable camera")
        }
    }
}
```

Available helpers (in `util/permissions/AppPermissionState.kt`):

| Helper | Permission | iOS Info.plist key |
|---|---|---|
| `rememberNotificationPermissionState()` | Notifications (Android 13+ / iOS) | — |
| `rememberCameraPermissionState()` | Camera | `NSCameraUsageDescription` |
| `rememberGalleryPermissionState()` | Photo gallery | `NSPhotoLibraryUsageDescription` |

These are the permissions the kit ships, and it depends only on their Calf modules
(`calf-permissions-core`, `-camera`, `-gallery`, `-notifications`).

:::warning Do not use the umbrella `calf-permissions` artifact
It links every permission API — location, bluetooth, contacts, calendar and the rest. App Store
review scans for those symbols and rejects the upload with **ITMS-90683: Missing purpose string in
Info.plist**, demanding a usage description for permissions your app never requests. Depend on one
module per permission you actually use.
:::

To use another permission, add its module in `MobileApp/gradle/libs.versions.toml` and
`shared/build.gradle.kts`, then call the generic helper with the matching Info.plist key:

```kotlin
implementation(libs.calf.permissions.location)          // build.gradle.kts

rememberAppPermissionState(Permission.FineLocation)     // + NSLocationWhenInUseUsageDescription
```

### Ask on screen entry

For permissions that should be requested as soon as a screen appears (e.g. notifications on the home screen):

```kotlin
RequestPermissionOnEntry(rememberNotificationPermissionState())
```

This skips the request if the permission is already granted, or if the user previously denied it (prompting again unasked is hostile UX — show a rationale UI with a button instead).

### Any other permission

Every other Calf permission works through the same wrapper:

```kotlin
val bluetooth = rememberAppPermissionState(Permission.Bluetooth)
```

Add the matching iOS usage-description key to `iosApp/iosApp/Info.plist`, and the Android `<uses-permission>` entry to `androidApp/src/main/AndroidManifest.xml` if the permission requires one.
