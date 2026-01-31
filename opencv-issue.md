```markdown
# BioCampEntry – Gradle / AGP / Kotlin / OpenCV Build Issue Postmortem

## Overview

This document records a complete analysis of the Gradle sync and build failures encountered while configuring an Android project (`BioCampEntry`) that integrates:

- Android Gradle Plugin (AGP)
- Kotlin 2.x
- KSP
- Hilt
- Jetpack Compose
- OpenCV Android SDK (as a module)

The issues were primarily caused by **version incompatibilities across AGP patch releases, Kotlin 2.x, KSP, and Hilt**, compounded by **OpenCV’s legacy Gradle and Kotlin code**.

---
## Helpful threads
- https://github.com/opencv/opencv/issues/24663
- https://docs.gradle.org/current/userguide/compatibility.html#android

## Environment

- **OS**: Windows
- **Gradle Wrapper**: `gradle-8.13-bin.zip`
- **Android Studio**: Hedgehog / Iguana-era (AGP 8.11.x compatible)
- **Build System**: Version Catalog (`libs.versions.toml`)
- **Language**: Kotlin 2.2.21
- **Third-party SDK**: OpenCV (imported as Android module)

---

## Initial Symptoms

### 1. AGP Internal API Failure

```

Unable to load class 'com.android.build.api.artifact.ScopedArtifact$POST_COMPILATION_CLASSES'

```

This error occurs **before compilation**, during Gradle configuration, indicating a **binary incompatibility between AGP and a plugin** that relies on internal AGP APIs.

---

### 2. KSP Plugin Not Found

```

Plugin [id: 'com.google.devtools.ksp', version: '2.2.21-2.0.2'] was not found

```

Cause:
- The artifact `2.2.21-2.0.2` **does not exist**
- KSP versions must exactly match a published Kotlin-compatible build

---

### 3. OpenCV Gradle Warnings (Groovy DSL)

Multiple warnings like:

```

Properties should be assigned using the 'propName = value' syntax
This is scheduled to be removed in Gradle 10.0

```

Cause:
- OpenCV’s `build.gradle` uses **deprecated Groovy DSL syntax**
- Gradle 8.x emits warnings; Gradle 10 will fail hard

These warnings are **non-fatal** for now.

---

### 4. Kotlin 2.x Unsigned Types Warnings (OpenCV)

```

This declaration needs opt-in.
@kotlin.ExperimentalUnsignedTypes

````

Cause:
- OpenCV Kotlin bindings use `UByte`, `UInt`, etc.
- Kotlin 2.x enforces stricter opt-in requirements
- These are **compiler warnings**, not errors

---

## Root Cause Analysis

### Primary Failure Cause

**AGP 8.11.2 + Hilt + KSP caused a binary mismatch**

- AGP `8.11.2` introduced internal API changes
- Hilt and/or KSP were still compiled against **older AGP internals**
- Result: `ScopedArtifact$POST_COMPILATION_CLASSES` not found at runtime

This is a **known class of failure** when upgrading AGP patch versions too aggressively.

---

### Secondary Contributing Factors

1. **Incorrect KSP version**
   - `2.2.21-2.0.2` was invalid
2. **OpenCV not Kotlin-2-ready**
   - Emits unsigned type warnings
3. **Gradle cache/daemon instability**
   - Exacerbates plugin resolution errors on Windows

---

## Resolution Steps (What Fixed It)

### 1. Downgraded AGP Patch Version

```toml
agp = "8.11.0"
````

Reason:

* `8.11.0` is stable and widely tested
* Avoids breaking internal API changes in `8.11.1+ / 8.11.2`

---

### 2. Corrected KSP Version

```toml
ksp = "2.2.21-2.0.4"
```

Reason:

* Matches Kotlin `2.2.21`
* Actually exists in Maven repositories

---

### 3. Aligned Hilt Version

```toml
hilt = "2.58"
```

Reason:

* Known compatible with AGP 8.11.0
* Newer Hilt versions track AGP internals closely and may break

---

### 4. Accepted OpenCV Warnings (Non-blocking)

* Groovy DSL deprecations → safe until Gradle 10
* Kotlin unsigned opt-in warnings → harmless unless `-Werror` is enabled

---

## Final Working Version Catalog

```toml
[versions]
agp = "8.11.0"
kotlin = "2.2.21"
ksp = "2.2.21-2.0.4"
hilt = "2.58"
composeBom = "2024.09.00"

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
google-devtools-ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
hilt-android-gradle = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

---

## Compatibility Matrix

| Component                   | Version Used        | Compatible With        | Notes                                                   |
| --------------------------- | ------------------- | ---------------------- | ------------------------------------------------------- |
| Android Gradle Plugin (AGP) | 8.11.0              | Gradle 8.13            | Stable; newer 8.11.x patch releases broke internal APIs |
| Gradle Wrapper              | 8.13                | AGP 8.11.0             | Required minimum for AGP 8.11.x                         |
| Kotlin                      | 2.2.21              | AGP 8.11.0             | Kotlin 2.x requires stricter opt-in handling            |
| Kotlin Compose Plugin       | 2.2.21              | Compose BOM 2024.09.00 | Must match Kotlin version exactly                       |
| KSP                         | 2.2.21-2.0.4        | Kotlin 2.2.21          | Earlier `2.0.2` artifact did not exist                  |
| Hilt                        | 2.58                | AGP 8.11.0             | Uses internal AGP APIs; sensitive to AGP patch changes  |
| Jetpack Compose BOM         | 2024.09.00          | Kotlin 2.2.21          | Verified stable with Kotlin 2.x                         |
| AndroidX Core KTX           | 1.17.0              | AGP 8.11.0             | No compatibility issues observed                        |
| Lifecycle Runtime KTX       | 2.10.0              | Kotlin 2.2.21          | Compatible with Compose + Kotlin 2                      |
| Navigation Runtime          | 2.9.6               | AGP 8.11.0             | Works with Compose Navigation                           |
| OpenCV Android SDK          | 4.x (module import) | Kotlin 2.2.21          | Requires unsigned type opt-in                           |
| Operating System            | Windows             | Gradle Daemon          | File-locking can cause transient cache issues           |

---

## Key Takeaways

* **Do not blindly upgrade AGP patch versions**
* **KSP must exactly match Kotlin**
* **Hilt is highly AGP-internal-API sensitive**
* **OpenCV is not Kotlin 2–native yet**
* When builds “magically” start working, it is almost always **version realignment**, not cache luck

---

## Status

✅ Project builds
✅ Gradle sync successful
⚠ OpenCV warnings acknowledged
🟢 Configuration considered stable

```
```
