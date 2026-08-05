# GFMC SDK — distribution

Maven distribution point for the GFMC SDK (`com.sltr.gfmc:gfmc-sdk`). This
repo holds **no source code** — only the published packages, listed under
**Packages** in the sidebar. Source is maintained privately by the BDN team.

The package is public: no invitation, no approval, no account setup on our
side. You do still need a GitHub token, because GitHub Packages refuses
anonymous Maven requests even for public packages — see
[Credentials](#credentials) below.

## Consuming the SDK

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven {
            url = uri("https://maven.pkg.github.com/BDN-ID/gfmc-sdk")
            credentials {
                username = providers.gradleProperty("gpr.user").orNull
                    ?: System.getenv("GITHUB_ACTOR")
                password = providers.gradleProperty("gpr.key").orNull
                    ?: System.getenv("GITHUB_TOKEN")
            }
        }
    }
}
```

```kotlin
// app/build.gradle.kts
dependencies {
    implementation("com.sltr.gfmc:gfmc-sdk:1.2.0")
}
```

Google Play Billing (`com.android.billingclient:billing-ktx`) is declared in
the artifact's Gradle module metadata and resolves automatically — don't add
it yourself.

## Credentials

GitHub Packages requires authentication even for public packages — this is a
registry-wide rule, not a restriction we set. Anonymous pulls exist only on
the `ghcr.io` Docker registry.

Create a **Personal access token (classic)** at
<https://github.com/settings/tokens> with the **`read:packages`** scope —
that's the only one needed — then put it outside your project, in
`~/.gradle/gradle.properties`:

```properties
gpr.user=<your github username>
gpr.key=<your token>
```

Never inline the token in `settings.gradle.kts` — that file is committed.
Any GitHub account works; membership in BDN-ID is not required.

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
| `401 Unauthorized` | Token missing, expired, or lacking `read:packages` |
| `Could not find com.sltr.gfmc:gfmc-sdk:x.y.z` | Credentials absent — Gradle silently skips a repo it can't authenticate against — or that version was never published |
| `NoClassDefFoundError: BillingClient` at runtime | You're consuming a raw `.aar` instead of the Maven artifact; the `.aar` carries no dependency metadata. Use the coordinate above |
