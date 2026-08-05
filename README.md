# GFMC SDK

Embeds the minicinema hub — streaming, entitlements and Google Play top-ups —
inside your Android app as a self-contained mini-program. You hand it a
session JWT and a `Context`; it takes over the screen from there.

The hub runs in its own `:gfmc` process, so a crash, OOM or WebView failure
inside it never takes down your app.

This repo is the distribution point. It carries **no source code** — only
published artifacts. Everything a host app needs is on this page.

**Current release: `1.2.2`** (SDK 2.3.2)

---

## Requirements

| Item | Minimum | Notes |
|---|---|---|
| `minSdk` | **24** (Android 7.0) | Lower fails the manifest merge |
| `compileSdk` | **36** | Enforced by the artifact's AAR metadata |
| `targetSdk` | 36 recommended | Play's target-API policy |
| Java / JVM target | **11** | `sourceCompatibility` / `targetCompatibility` |
| AGP / Gradle | 8.x | Gradle module metadata is required |
| Language | Kotlin **or** Java | Both are first-class — see below |
| Android System WebView | **Chromium 111+** | Runtime requirement, see below |
| Google Play Store | installed | Billing, and the WebView update prompt |

`INTERNET`, `ACCESS_NETWORK_STATE` and `com.android.vending.BILLING` are
declared in the SDK's own manifest and merge into your app automatically —
don't add them.

### About the WebView 111 floor

This is a *runtime* requirement, and **the Android version does not imply
it**. Android System WebView ships as an independently updatable APK, so a
recent OS guarantees nothing: a vivo V2111 running Android 12 was measured
carrying Chrome 150 alongside WebView pinned at **91**.

It matters because the hub's stylesheet is Tailwind v4 output — `@layer`
blocks throughout, colours built with `color-mix()`. An older Chromium
doesn't fail loudly; per CSS error handling it silently discards at-rules it
can't parse, so the page loads, hydrates and runs JavaScript while rendering
almost entirely unstyled. On that vivo device, 42 KB of the 54 KB stylesheet
was dropped, the whole `@layer utilities` block included.

The SDK detects this itself. Below Chromium 111 it blocks the load and shows
a full-screen prompt deep-linking to the *actual* WebView provider's Play
listing — which is not always `com.google.android.webview` — with an escape
hatch for users who want to continue anyway. Your app also receives
`onError(GfmcSDKError.WEBVIEW_OUTDATED, …)`. You don't need to check anything
yourself.

---

## Install

No token, no account, no invitation. The `gh-pages` branch of this repo *is*
a Maven repository — a Maven repository is just a directory tree — so Gradle
reads it over plain HTTPS.

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

Keep `google()` and `mavenCentral()`. Only the SDK comes from the URL above;
Google Play Billing (`com.android.billingclient:billing-ktx`) resolves from
`google()` automatically, because it's declared in the artifact's Gradle
module metadata. **Don't declare Billing yourself.**

Published versions are listed in
[`maven-metadata.xml`](https://raw.githubusercontent.com/BDN-ID/gfmc-sdk/gh-pages/com/sltr/gfmc/gfmc-sdk/maven-metadata.xml).

<details>
<summary>Alternative: GitHub Packages registry (needs a token)</summary>

The same artifacts are also published to the GitHub Packages registry. Use
this only if your network blocks `raw.githubusercontent.com`.

```kotlin
maven {
    url = uri("https://maven.pkg.github.com/BDN-ID/gfmc-sdk")
    credentials {
        username = providers.gradleProperty("gpr.user").orNull
        password = providers.gradleProperty("gpr.key").orNull
    }
}
```

That route needs a token even though this repo is public: GitHub's Maven
registry refuses anonymous requests regardless of package visibility. It's a
registry-wide rule, not a restriction we set — anonymous pulls exist only on
the `ghcr.io` Docker registry.

Create a **personal access token (classic)** at
<https://github.com/settings/tokens> with the `read:packages` scope, then put
it in `~/.gradle/gradle.properties`, outside your project:

```properties
gpr.user=<your github username>
gpr.key=<your token>
```

Never inline a token in `settings.gradle.kts` — that file gets committed. Any
GitHub account works; membership in BDN-ID is not required.

</details>

---

## Quick start

Four calls. `jwt` is a **minicinema session token issued by your backend** —
not your app's own auth access token.

```kotlin
// Kotlin
GfmcSDK.init(context)
GfmcSDK.setTokenRefresher { result -> result.emit(auth.freshMinicinemaJwt()) }
GfmcSDK.setSkuListener { sku -> Log.d("Gfmc", "user picked $sku") }
GfmcSDK.open(context, jwt)
```

```java
// Java
GfmcSDK.init(this);
GfmcSDK.setTokenRefresher(result -> result.emit(auth.freshMinicinemaJwt()));
GfmcSDK.setSkuListener(sku -> Log.d("Gfmc", "user picked " + sku));
GfmcSDK.open(this, jwt);
```

`init()` also prewarms the hub's WebView engine in the background, so the
later `open()` launches faster. Call it once at app start.

---

## API

| Call | Purpose |
|---|---|
| `init(context)` / `init(context, environment, enableLogging)` / `init(context, config)` | Configure and register bridges. Also prewarms the hub |
| `open(context, jwt)` | Launch the hub |
| `open(context, jwt, onSku)` | Launch, registering a SKU listener in one call |
| `openWithTokenProvider(context, jwt, provider)` | Launch with a *synchronous* refresh callback |
| `setTokenRefresher(refresher)` | How to fetch a fresh JWT when the current one expires |
| `setSkuListener(listener)` | Fires when the user picks a product, before purchase starts |
| `setListener(context, listener)` | Lifecycle, error and purchase callbacks |
| `prewarmMiniProgram(context)` | Re-warm the hub process after the system killed it |
| `version` | Structured SDK version (`GfmcSDKVersion`) |

`GfmcSDKConfig.Builder`: `setEnvironment`, `setLocale`, `setTheme`,
`enableLogging`, `setConnectionTimeout`.

### Callbacks

```kotlin
GfmcSDK.setListener(context, object : GfmcSDKListener {
    override fun onHubReady() {}
    override fun onHubClosed() {}
    override fun onShareRequested(url: String, title: String) {}
    override fun onError(code: GfmcSDKError, message: String) {}
    override fun onPurchaseCompleted(sku: String, transactionId: String) {}
    override fun onPurchaseFailed(reason: String) {}
})
```

`GfmcSDKError`: `AUTH_FAILED`, `NETWORK_ERROR`, `SESSION_EXPIRED`,
`SDK_NOT_INITIALIZED`, `WEBVIEW_UNAVAILABLE`, `WEBVIEW_OUTDATED`,
`BILLING_UNAVAILABLE`, `PURCHASE_FAILED`, `VERIFY_FAILED`. Raised today:
`WEBVIEW_UNAVAILABLE`, `WEBVIEW_OUTDATED`, `NETWORK_ERROR`, and
`SDK_NOT_INITIALIZED` (a generic uncaught-crash signal). The rest are
reserved.

### Refreshing the session token

The hub asks for a fresh JWT when its own expires. Two forms — pick one.

**Asynchronous** (preferred): deliver the token whenever it's ready, from a
coroutine, a Retrofit callback, a worker, anything.

```kotlin
GfmcSDK.setTokenRefresher { result ->
    auth.refreshAsync(
        onSuccess = { jwt -> result.emit(jwt) },
        onError   = { e -> result.fail(e.message ?: "refresh failed") },
    )
}
GfmcSDK.open(context, jwt)
```

**Synchronous**: return the token, or throw. It runs off the main thread, so
a blocking network call is fine here.

```kotlin
GfmcSDK.openWithTokenProvider(context, jwt) { auth.freshMinicinemaJwt() }
```
```java
GfmcSDK.openWithTokenProvider(this, jwt, () -> auth.freshMinicinemaJwt());
```

A refresh that throws is reported as a failed refresh — it won't crash the
hub. Refreshes time out after 15 seconds.

---

## Purchases and top-ups

The hub runs Google Play Billing itself. Your app doesn't implement any of
it; `setSkuListener` is informational only.

```
web requests PURCHASE
  → Play Billing dialog
  → success → settled with Google immediately
              (consumed for point packs, acknowledged for entitlements/subs)
            → onPurchaseCompleted(sku, transactionId)
  → failure → onPurchaseFailed(reason)
```

Native treats Play Billing success as final — it does **not** wait for a
host-backend confirmation. The web reports the top-up to its own backend
(`POST /topup`) using the payload native sends alongside the purchase reply:

```json
{
  "invoice_id": "...",
  "purchase_token": "...",
  "product_sku": "...",
  "platform_package_name": "...",
  "status": "success"
}
```

> ### `purchase_token` is what a backend must verify
>
> It's the only field Google will accept. The Play Developer API resolves a
> purchase by `packageName + productId + purchaseToken`; **no public endpoint
> takes an `invoice_id` or orderId**. Crediting points off the rest of the
> payload means trusting a claim made on the client.
>
> The field is empty for `failed` and `pending` outcomes, where Play hasn't
> issued a token yet.

This concerns whoever owns the `/topup` endpoint, not the Android host app —
but it's the single most common way an integration ends up quietly
insecure, so it's stated here too.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| `Could not find com.sltr.gfmc:gfmc-sdk:x.y.z` | Trailing slash missing from the repository URL, that version was never published, or your network blocks `raw.githubusercontent.com` |
| `Could not resolve com.android.billingclient:billing-ktx` | `google()` missing from your repositories — only the SDK comes from the BDN URL |
| `NoClassDefFoundError: BillingClient` at runtime | You're using a raw `.aar` instead of the Maven coordinate. An `.aar` carries no dependency metadata, so Billing is never resolved |
| `Manifest merger failed … minSdk` | Your `minSdk` is below 24 |
| `Dependency requires compileSdk 36 or later` | Raise `compileSdk` |
| *Non-static method … from a static context* (Java) | You're on 1.2.0. Upgrade, or call via `GfmcSDK.INSTANCE` |
| Hub opens unstyled / looks broken | The device's WebView is below Chromium 111. Expected on 1.2.0 and earlier; from 1.2.1 the SDK shows an update prompt instead |
| A newly announced version isn't found | `raw.githubusercontent.com` caches branch content for a few minutes. Retry shortly |
| `401 Unauthorized` | GitHub Packages route only: token missing, expired, or lacking `read:packages` |

Enable logging (`init(context, environment, enableLogging = true)`) and watch
logcat tag **`GfmcSDK`** — bridge traffic, billing responses and WebView
engine detection all report there.

---

## Versioning

The artifact version and the SDK version are separate sequences. The SDK
version is what `GfmcSDK.version` reports at runtime.

| Artifact | SDK |
|---|---|
| `1.2.2` | 2.3.2 |
| `1.2.1` | 2.3.1 |
| `1.2.0` | 2.3.0 |

Release notes are distributed by the BDN team with each version.
