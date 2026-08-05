# GFMC SDK — distribution

Maven distribution point for the GFMC SDK (`com.sltr.gfmc:gfmc-sdk`). This
repo holds **no source code** — only the published packages, listed under
**Packages** in the sidebar. Source is maintained privately by the BDN team.

No invitation, no token, no account. The `gh-pages` branch of this repo *is*
a Maven repository — a Maven repository is just a directory tree — so Gradle
can read it directly over plain HTTPS.

## Consuming the SDK

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://raw.githubusercontent.com/BDN-ID/gfmc-sdk/gh-pages/") }
    }
}
```

```kotlin
// app/build.gradle.kts
dependencies {
    implementation("com.sltr.gfmc:gfmc-sdk:1.2.2")
}
```

Keep `google()` and `mavenCentral()` in the list. Only the SDK comes from the
URL above; Google Play Billing
(`com.android.billingclient:billing-ktx`) resolves from `google()`
automatically because it's declared in this artifact's Gradle module
metadata — don't add it yourself.

## Kotlin and Java

Both languages call the SDK the same way. Java needs no casts, no wrappers,
and no `INSTANCE`.

```kotlin
// Kotlin
GfmcSDK.init(context)
GfmcSDK.setTokenRefresher { result -> result.emit(auth.freshJwt()) }
GfmcSDK.setSkuListener { sku -> Log.d("Sku", sku) }
GfmcSDK.open(context, jwt)
GfmcSDK.openWithTokenProvider(context, jwt) { auth.freshJwt() }
```

```java
// Java
GfmcSDK.init(this);
GfmcSDK.setTokenRefresher(result -> result.emit(auth.freshJwt()));
GfmcSDK.setSkuListener(sku -> Log.d("Sku", sku));
GfmcSDK.open(this, jwt);
GfmcSDK.openWithTokenProvider(this, jwt, () -> auth.freshJwt());
```

`GfmcSDKListener` has seven methods, but every one defaults to a no-op — a
Java host overrides only what it needs.

Two notes if you're upgrading rather than starting fresh:

- **On 1.2.0**, `GfmcSDK.init(this)` doesn't compile from Java at all —
  *"Non-static method 'init(android.content.Context)' cannot be referenced
  from a static context"*. `GfmcSDK` is a Kotlin `object`, whose members Java
  sees on a hidden `INSTANCE` field. `GfmcSDK.INSTANCE.init(this)` works
  around it and keeps working on every version.
- **Before 1.2.2**, the synchronous refresh form was an `open()` overload.
  It's now `openWithTokenProvider(context, jwt, provider)` — as a third
  `open()` overload it collided with `open(context, jwt, onSku)`, and neither
  compiler can pick between two single-abstract-method parameters given a
  bare lambda. `open(context, jwt)` and `open(context, jwt, onSku)` did not
  change.

Published versions are listed in
[`maven-metadata.xml`](https://raw.githubusercontent.com/BDN-ID/gfmc-sdk/gh-pages/com/sltr/gfmc/gfmc-sdk/maven-metadata.xml).

## If your network blocks raw.githubusercontent.com

The same artifacts are also published to the GitHub Packages registry. That
route needs a token — GitHub's Maven registry refuses anonymous requests even
for public packages, which is a registry-wide rule rather than a restriction
we set. Anonymous pulls exist only on the `ghcr.io` Docker registry.

```kotlin
maven {
    url = uri("https://maven.pkg.github.com/BDN-ID/gfmc-sdk")
    credentials {
        username = providers.gradleProperty("gpr.user").orNull
        password = providers.gradleProperty("gpr.key").orNull
    }
}
```

Create a **Personal access token (classic)** at
<https://github.com/settings/tokens> with the **`read:packages`** scope, then
put it in `~/.gradle/gradle.properties`, outside your project:

```properties
gpr.user=<your github username>
gpr.key=<your token>
```

Never inline the token in `settings.gradle.kts` — that file is committed. Any
GitHub account works; membership in BDN-ID is not required.

## Host app requirements

| Item | Minimum |
|---|---|
| `minSdk` | 24 (Android 7.0) |
| `compileSdk` | 36 |
| Java / JVM target | 11 |
| AGP / Gradle | 8.x |
| Android System WebView | Chromium **111+** |
| Google Play Store | installed |

The WebView floor is a *runtime* requirement and the Android version does not
imply it — WebView ships as a separately updatable APK, and devices have been
measured running WebView 91 alongside Chrome 150. Below Chromium 111 the SDK
blocks the load itself and shows an update prompt rather than rendering an
unstyled page.

`INTERNET`, `ACCESS_NETWORK_STATE` and `com.android.vending.BILLING` come
from the SDK's own manifest and merge into your app automatically.

## Integration

See the reference host app,
[`BDN-ID/jessica-native-dummy-host`](https://github.com/BDN-ID/jessica-native-dummy-host),
for complete working wiring — `GfmcSDK.init` / `setTokenRefresher` /
`setSkuListener` / `open`, plus the purchase and JWT-refresh flows.

## Troubleshooting

| Symptom | Cause |
|---|---|
| `Could not find com.sltr.gfmc:gfmc-sdk:x.y.z` | Trailing slash missing from the repository URL, that version was never published, or your network blocks `raw.githubusercontent.com` |
| `Could not resolve com.android.billingclient:billing-ktx` | `google()` is missing from your repositories — only the SDK itself comes from the BDN URL |
| `NoClassDefFoundError: BillingClient` at runtime | You're consuming a raw `.aar` instead of the Maven artifact; an `.aar` carries no dependency metadata, so Billing is never resolved. Use the coordinate above |
| A newly announced version isn't found | `raw.githubusercontent.com` caches branch content for a few minutes. Retry shortly |
| `401 Unauthorized` | Only applies to the GitHub Packages route: token missing, expired, or lacking `read:packages` |
