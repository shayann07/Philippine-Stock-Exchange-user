# Philippine-Stock-Exchange-user

A native Kotlin Android **client** for a Firebase-backed investment / MLM platform — `applicationId = com.pse.pse`. Despite the repository name, this is **not** a stock-exchange / brokerage / market-data app: users sign up with Firebase Auth, buy predefined "investment plans" with daily ROI + payout caps, deposit USDT-BEP20 via a Render-hosted backend (`psedeposit-main.onrender.com`), withdraw with a flat $4 fee, and earn referral / leadership / team-rank commissions through an MLM hierarchy. Pairs with the sibling `Philippine-Stock-Exchange-admin` repo on the same Firebase project (`philippine-stock-exchang-db`). Push goes out via the FCM v1 HTTP API called directly from the device. MVVM + Jetpack Navigation, ~88 Kotlin files, ~90 layouts.

> **Heads-up:** the previous README claimed real-time market data, watchlists, broker-API trading, two-factor auth, and an MIT licence — none of those exist in the code. The same Firebase service-account key as the admin app is inlined in source. See [Security Notice](#-security-notice--read-first).

## ⚠ Security Notice — Read first

Three serious credential issues are committed to git history, on a public repo:

1. **Firebase Admin service-account JSON inlined** at `app/src/main/java/com/pse/pse/fcm/AccessToken.kt:27-42`. Same key as the sibling admin app:
   - `client_email = REDACTED_CLIENT_EMAIL`
   - `private_key_id = REDACTED_KEY_ID`
   - Full RSA `private_key` in the `REDACTED_KEY_START` block

   The app loads the JSON via `GoogleCredentials.fromStream(...)` with the `firebase.messaging` scope so it can call FCM v1 directly from the device. Anyone with the APK can extract the key. **Disable in IAM, purge from git history, and move FCM sending to a Cloud Function** — and do it for *both* apps at once because they share the credential.

2. **`app/google-services.json` is tracked.** Project `philippine-stock-exchang-db`, project number `324672116485`, web API key `REDACTED_API_KEY`. The same project the admin app talks to — compromise on one side compromises both.

3. **The deposit endpoint is unauthenticated.** `app/src/main/java/com/pse/pse/fragments/DepositAmountFragment` posts JSON to `https://psedeposit-main.onrender.com/api/create-transaction` with no API key or signature; trust comes from CoinPayments-side configuration only.

Notes:
- `fcm/Fcm.kt` posts to `https://fcm.googleapis.com/v1/projects/philippine-stock-exchang-db/messages:send`, but the service account in `AccessToken.kt` is for project `philippine-stock-exchang-296cd`. Two **different** Firebase projects — the OAuth token does not match the FCM URL, so push from the device would 403.
- Release is **not minified** (`isMinifyEnabled = false`) — the production APK is fully reversible.
- There is **no `.gitignore`**. `local.properties`, `.gradle/`, `.idea/`, and `app/build/` are tracked.

## Status

- Working tree clean on `master`. Recent commits are all GitPulse contribution markers (`237352a`, `ba66b74`, `0a54e5d`, `11c6f72`, `7e0ec42`, …).
- Remote: `https://github.com/shayann07/Philippine-Stock-Exchange-user.git`.
- This README was rewritten from a code audit; the previous one's market-data / trading / 2FA / MIT claims and `<!-- gitpulse:contribution … -->` markers are removed.

## How it works

### Auth + onboarding

`SplashFragment` performs an auth check and routes to `SignInFragment` / `SignUpFragment` (Firebase Auth email/password). On sign-up, the app reserves a unique `MXG-`-style id via the `uidReservations` Firestore collection and validates the optional referrer code. `NewPasswordFragment` covers password reset.

### Single Activity + Navigation

`MainActivity` hosts a `NavHostFragment` with drawer + bottom navigation. 26 fragments under `fragments/`:

- **Dashboard** — `HomeFragment` (balance, quick actions).
- **Plans** — `PlanFragment` (templates from `plans` Firestore collection: dailyRoiPercent, totalPayoutCapPercent, directProfitPercent, min/max), `BuyPlanFragment` (debits `accounts.balance`, writes a `userPlans` row, credits the upline's "direct profit" wallet via `BuyPlanRepo`), `MyPlansFragment` (active / completed).
- **Wallet** — `DepositAmountFragment` (USDT-BEP20 via the Render endpoint, ZXing-rendered QR), `WithdrawAmountFragment` (writes a `withdraw` Firestore doc with a flat $4 fee for the admin to approve), `TransactionHistoryFragment` (reads the `transactions` ledger).
- **MLM hierarchy** — `LeadershipFragment`, `TeamLevelsFragment`, `TeamRankingFragment`, `LevelUsersFragment`, `SalaryIncomeFragment`.
- **Support** — `SupportFragment`, `NewSupportRequestFragment`, `ChatFragment`, `DetailChatFragment`, `SupportChatFragment`.
- **Profile** — `ProfileFragment`, `NewPasswordFragment`.

### MVVM + Repository

ViewModels mirror eight repositories: `AuthRepository`, `AccountRepository`, `BuyPlanRepo`, `TransactionRepository`, `PlanRepository`, `TeamRepository`, `LeadershipRepository`, `ChatRepository`. State is exposed as `LiveData` / `StateFlow`; coroutines via `viewModelScope`; cached user state in SharedPreferences.

### Push notifications

`fcm/NotificationService` extends `FirebaseMessagingService`. Sending happens **client-side** using the embedded service-account key in `AccessToken.kt` — anti-pattern; should live behind a Cloud Function.

### Auto-update

The APK download URL is pulled from Firebase **Remote Config**, downloaded with `DownloadManager`, and install completion is handled by `receivers/UpdateDownloadReceiver` (`exported="true"` to receive `ACTION_DOWNLOAD_COMPLETE`).

### What is NOT in the code despite the previous README

- No real-time market data, no stock prices, no trading volume, no watchlists.
- No broker API integration; users only buy predefined "investment plans".
- No two-factor authentication — only Firebase Auth email/password.
- No WebSocket feeds.

## Tech stack

- **Build:** version catalog at `gradle/libs.versions.toml`, Kotlin 2.0.21, Java 11, Google Services plugin, View Binding, BuildConfig. Release does **not** minify or shrink.
- **App config:** `applicationId = com.pse.pse`, `compileSdk = 36`, `minSdk = 24`, `targetSdk = 36`, `versionCode = 5`, `versionName = "5.0"`. `android:allowBackup="false"`.
- **Firebase:** firestore 25.1.4, auth 23.2.1, messaging 24.1.1, storage 20.1.0, remote-config 22.1.2, functions 22.0.0, analytics-ktx (BOM 32.3.0).
- **AndroidX / Jetpack:** Navigation 2.9.0, Lifecycle 2.6.1, Coroutines 1.7.3, Room 2.7.1 *(declared, unused)*.
- **Networking:** OkHttp 4.12.0, Volley 1.2.1, Gson 2.12.1, gRPC 1.73.0 *(declared, unused)*, `google-auth-library-oauth2-http` 1.2.0.
- **UI:** Material 1.12.0, Glide 4.16.0, Lottie 6.5.2, Picasso 2.71828, ZXing core 3.5.0, CircleImageView 3.1.0, Ultra-Pull-to-Refresh 1.0.11.
- **Permissions:** `INTERNET`, `ACCESS_NETWORK_STATE`, `REQUEST_INSTALL_PACKAGES`, `WRITE_EXTERNAL_STORAGE` (maxSdk 28), `READ_EXTERNAL_STORAGE` (maxSdk 32), `POST_NOTIFICATIONS`, `c2dm.RECEIVE`.

The repo does **not** use Hilt/Dagger, Compose, DataBinding, App Check, Crashlytics, WorkManager, or Retrofit. The CoinPayments Java library and gRPC are declared but unused.

## Project layout

```
Philippine-Stock-Exchange-user/
├── app/
│   ├── google-services.json                       ⚠ tracked (project philippine-stock-exchang-db)
│   ├── build.gradle.kts                           applicationId com.pse.pse, versionCode 5
│   └── src/main/
│       ├── AndroidManifest.xml
│       └── java/com/pse/pse/
│           ├── MainActivity.kt
│           ├── fragments/                         26 fragments (Splash, plans, deposit/withdraw,
│           │                                      MLM team views, support chat, profile)
│           ├── viewmodels/, repository/           MVVM (8 repos)
│           ├── fcm/
│           │   ├── AccessToken.kt                 ⚠ Firebase service-account JSON inlined (27-42)
│           │   ├── Fcm.kt                         FCM v1 send (project id mismatch)
│           │   └── NotificationService.kt
│           ├── receivers/UpdateDownloadReceiver.kt exported=true (DOWNLOAD_COMPLETE)
│           └── utils/, models/, adapters/
├── local.properties                               ⚠ tracked
├── .gradle/, .idea/, app/build/                   ⚠ tracked, no .gitignore
└── README.md
```

## Setup / run

1. **Rotate every credential listed in the [Security Notice](#-security-notice--read-first) first** — the embedded Firebase service-account key is shared with the admin app, so rotate at IAM and purge from history in *both* repos. Generate fresh `google-services.json` files for each app.
2. Add a real `.gitignore` (at minimum: `app/google-services.json`, `local.properties`, `.gradle/`, `.idea/`, `build/`, `*.jks`) and `git rm --cached` everything currently shadowed.
3. Open in Android Studio, sync, run on Android 7.0+.

## Honest limitations

- **README claims do not match the code.** No real-time market data, no watchlists, no broker-API trading, no 2FA, no MIT licence file.
- **Service account on the device.** `fcm/AccessToken.kt:27-42` lets every install mint privileged Google OAuth tokens. Move to a backend.
- **Mismatched FCM project id.** Service account is for `…-296cd`; `Fcm.kt` posts to `…-db`. Push from the device 403s today.
- **Deposit endpoint has no auth.** `https://psedeposit-main.onrender.com/api/create-transaction` accepts any caller; trust is implicit.
- **Release is not minified.** `isMinifyEnabled = false` — the production APK is fully reversible.
- **`UpdateDownloadReceiver` is exported.** Necessary to receive `DOWNLOAD_COMPLETE`, but verify it filters by download id before kicking off the install intent.
- **Room is on the classpath but unused.** No `@Entity`, no DAOs in this app — drop the dependency or wire it.
- **CoinPayments Java + gRPC are declared but unused.** Dead dependencies.
- **`AsyncTask`** in `AccessToken.kt:23` is deprecated since API 30. Combine the rotation work with a Coroutines port.
- **GitPulse marker leftovers in the previous README.** Removed in this rewrite.
- **No `LICENSE` file** at the repo root despite the previous README claiming MIT.
- **No tests** beyond `ExampleUnitTest.kt`.
