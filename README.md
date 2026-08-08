# GFMC SDK

Embeds the minicinema hub — streaming, entitlements and Google Play top-ups —
inside your Android app as a self-contained mini-program. You hand it a
session JWT and a `Context`; it takes over the screen from there.

The hub runs in its own `:gfmc` process, so a crash, OOM or WebView failure
inside it never takes down your app.

This repo is the distribution point. It carries **no source code** — only
published artifacts. Everything a host app needs is on this page.

**Current release: `1.2.7`** (SDK 2.3.7)

> ### ⚠️ Use `GfmcSDKEnv.SANDBOX` on 1.2.7
>
> In this release the `PRODUCTION` host is **not reachable yet** — opening the
> hub on it fails with `GfmcSDKError.NETWORK_ERROR`.
>
> `GfmcSDK.init(context)` **defaults to `PRODUCTION`**, so the one-line quick
> start below hits this unless you pass an environment. Pass `SANDBOX`
> explicitly:
>
> ```kotlin
> GfmcSDK.init(context, GfmcSDKEnv.SANDBOX)
> ```
>
> `SANDBOX` and `DEV` are live and serve the same hub. Switch to `PRODUCTION`
> once BDN confirms it is up — no other code change is needed. Staying on
> `1.2.6` is also fine: every environment was reachable on that release.

---

## Requirements

| Item | Minimum | Notes |
|---|---|---|
| `minSdk` | **24** (Android 7.0) | Lower fails the manifest merge |
| `compileSdk` | **36** | Enforced by the artifact's AAR metadata |
| `targetSdk` | 36 recommended | Play's target-API policy |
| Java / JVM target | **11** | `sourceCompatibility` / `targetCompatibility` |
| AGP / Gradle | 8.x | Gradle module metadata is required |
| Language | Kotlin **or** Java | Both call the SDK identically |
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
    implementation("com.sltr.gfmc:gfmc-sdk:1.2.7")
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

The artifacts are also published to the GitHub Packages registry. Use this
only if your network blocks `raw.githubusercontent.com`.

> **The registry is an incomplete mirror.** It is published best-effort and
> has gaps — `1.2.4` and `1.2.5` were never uploaded to it, for instance.
> `gh-pages` is the source of truth for what has actually shipped, so check
> [`maven-metadata.xml`](https://raw.githubusercontent.com/BDN-ID/gfmc-sdk/gh-pages/com/sltr/gfmc/gfmc-sdk/maven-metadata.xml)
> rather than the registry's package list, and prefer the `gh-pages`
> repository unless you specifically cannot reach it.

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

### Environment

`init(context)` runs against **production**. Pass an environment explicitly to
change that:

```kotlin
// Kotlin
GfmcSDK.init(context)                       // production (default) — see the warning below
GfmcSDK.init(context, GfmcSDKEnv.SANDBOX)   // sandbox
```

```java
// Java
GfmcSDK.init(this);
GfmcSDK.init(this, GfmcSDKEnv.SANDBOX);
```

> **On 1.2.7, pass `SANDBOX`.** The default `PRODUCTION` host is not reachable
> yet — every hub open on it returns `NETWORK_ERROR`. See the warning at the
> top of this page.

A third argument turns on logging — worth having while you integrate. It
writes to logcat under the tag `GfmcSDK`:

```kotlin
GfmcSDK.init(context, GfmcSDKEnv.SANDBOX, true)
```

> The environment selects the hub host. Up to and including `1.2.6` every
> value resolved to the same host, so the choice only recorded intent. **From
> `1.2.7` the split is real** — `PRODUCTION` and `SANDBOX`/`DEV` are separate
> hosts, and only the latter is currently reachable.

Test purchases are a separate matter, and not something the SDK controls.
Google decides that from the Play Console: add the tester's account under
**Setup → License testing**, and install the app from a Play track. Purchases
made by those accounts aren't charged.

---


## Versioning

The artifact version and the SDK version are separate sequences. The SDK
version is what `GfmcSDK.version` reports at runtime.

| Artifact | SDK | |
|---|---|---|
| `1.2.7` | 2.3.7 | Hub's first render is authenticated — the session JWT is seeded into the WebView cookie jar before the first navigation, cutting ~2.6s of boot that was spent fetching a token the host already held. Immersive hub + `SET_FULLSCREEN` bridge action. **`PRODUCTION` host not reachable yet — use `SANDBOX`** |
| `1.2.6` | 2.3.6 | Capsule "more" menu: Share item removed, all native UI defaults to English, new `GET_APP_VERSION` bridge action, redesigned About dialog |
| `1.2.5` | 2.3.5 | Recovers from WebView renderer-process crashes (`onRenderProcessGone`) instead of a permanent blank/gray screen |
| `1.2.4` | 2.3.4 | Fixed `isMiniProgramProcess()`'s API-level guard, which silently disabled `init()`/warmup on every Android 9–12 cold start |
| `1.2.3` | 2.3.3 | `GfmcSDKEnv.STAGING` renamed to `SANDBOX` |
| `1.2.2` | 2.3.2 | |
| `1.2.1` | 2.3.1 | |
| `1.2.0` | 2.3.0 | |

Release notes are distributed by the BDN team with each version.
