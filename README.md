# Motto OTT SDK for Android

The Kotlin SDK for building OTT applications on the Motto Content Delivery API: Android and
Android TV (minSdk 26, one APK serves both), Jetpack Compose, coroutines and `StateFlow`,
compiled. It is the engine beneath the application — the platform bootstrap, page
sessions, sources and their refresh, filters, auth, playback, checkout, navigation — and the
application owns the look. This repository documents its releases: each tag is a release,
`VERSION` names it, and `api/` is the public API map. The modules themselves are served
from a Maven repository (see Installation).

| Module | Purpose |
|---|---|
| `ott-core` | The engine: platform bootstrap, page sessions, sources and their refresh, filters, auth and the token lifecycle, playback resolution, concurrency, monetization reads, analytics, one resolver per page component. It carries the generated Content Delivery API types. |
| `ott-player` | Media3 ExoPlayer with Widevine, the normalized refusal taxonomy, the headless `PlayerSession`, the Compose `PlayerView`, Google IMA ads, Mux Data analytics, Google Cast. |
| `ott-page-components` | The page arrangement, `PageComponentStack` (the Android TV focus rules), the per-component interfaces, and the two player page components. |
| `ott-navigation` | Navigation: `TabNavigator` for an application that owns its tabs, `StackNavigator` for a section inside a host's own shell, one `open` for every target, the Back-key rules. |
| `ott-auth` | The Compose auth forms and copy, OIDC through Custom Tabs, Google sign-in through Credential Manager, TV pairing. |
| `ott-checkout` | Provider-pluggable checkout: the registry, the paywall and offer surfaces, the platform's hosted checkout (a Custom Tab on a phone, a QR code on a television). |

## Installation

The modules are in a public Maven repository, read without credentials. Declare it, and Mux's,
beside `google()` and `mavenCentral()`, and depend on the modules your application uses, all
at the same version:

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven("https://muxinc.jfrog.io/artifactory/default-maven-release-local")
        maven("https://storage.googleapis.com/motto-ott-sdk-releases/android")
    }
}

// app/build.gradle.kts
dependencies {
    implementation("com.mottostreaming.ott:ott-core:<version>")
    implementation("com.mottostreaming.ott:ott-player:<version>")
    implementation("com.mottostreaming.ott:ott-page-components:<version>")
    implementation("com.mottostreaming.ott:ott-navigation:<version>")
    implementation("com.mottostreaming.ott:ott-auth:<version>")
    implementation("com.mottostreaming.ott:ott-checkout:<version>")
}
```

Then:

```kotlin
val client = MottoOTTClient(MottoOTTClientConfig(applicationContext, "<your public key>", "https://your-app.example", "android"))
val platform = client.bootstrap()
val page = client.pages.open("/")
```

`ott-core` carries the generated Content Delivery API types (`com.mottostreaming:cda-proto`,
served from the same repository) as part of its API, so an application names any message
under its generated name (`build.buf.gen.motto.cda.cms.banner.v3.Banner`) without adding it.
Every module but `ott-core` brings Google IMA and Mux Data, and `ott-player` Google Cast, from
their vendors' repositories; `ott-core` alone brings none of them. Mux reports under the
platform's `integrations.mux.env_key` while its analytics feature is on, and measures nothing
on a platform without one.

## What the application declares

The SDK reports a missing declaration through `MottoOTTDiagnostics.onHostMisconfiguration`.

- **Core library desugaring**, which Google IMA requires of every APK that carries it (any
  module but `ott-core`):

  ```kotlin
  android { compileOptions { isCoreLibraryDesugaringEnabled = true } }
  dependencies { coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.1.4") }
  ```

- **The Google Cast options provider**, for casting from a phone (a television never starts a
  sender). `CastButton` needs a `FragmentActivity` host, because the route picker opens as a
  dialog fragment.

  ```xml
  <meta-data
      android:name="com.google.android.gms.cast.framework.OPTIONS_PROVIDER_CLASS_NAME"
      android:value="com.mottostreaming.ott.player.cast.GoogleCastOptionsProvider" />
  ```

- **Picture in Picture** on the activity that hosts the player:

  ```xml
  <activity
      android:supportsPictureInPicture="true"
      android:configChanges="screenSize|smallestScreenSize|screenLayout|orientation" />
  ```

- **The OIDC return** (`ott-auth`), by default the platform's callback page as a verified App
  Link, forwarded to `OidcSignIn.handleRedirect(intent.data)` from `onCreate` and
  `onNewIntent` (a `singleTop` or `singleTask` activity keeps it in the same one). The
  platform serves `/.well-known/assetlinks.json` naming the application's package and signing
  certificate.

  ```xml
  <intent-filter android:autoVerify="true">
      <action android:name="android.intent.action.VIEW" />
      <category android:name="android.intent.category.DEFAULT" />
      <category android:name="android.intent.category.BROWSABLE" />
      <data android:scheme="https" android:host="<platform host>" android:path="/auth/oidc-callback" />
  </intent-filter>
  ```

- **The return from the web checkout** (`ott-checkout`): `PurchaseRequest.returnUrl`, which
  defaults to the client's `appBaseUrl`, as an App Link or a custom scheme the application
  declares; the activity passes the landing's query to `Checkout.completeReturn` from
  `onNewIntent`.

## Versions

Each release is a tag, `<major>.<minor>.<patch>`, and the version rules are semantic: a minor
release adds, a major one may break. Every module of a release has the release's version.
`api/` holds each module's public API as the Kotlin binary compatibility validator writes it:
the map of what an application compiles against, and, compared between two tags, what a
release changed.
