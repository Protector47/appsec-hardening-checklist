# Android App Security Checklist

Android-specific controls, aligned with [OWASP MASVS v2](https://mas.owasp.org/MASVS/) and the [MASTG](https://mas.owasp.org/MASTG/) Android tests. Use alongside [mobile-common.md](mobile-common.md). Verify against your own debug and release builds.

## 1. Build configuration

- [ ] **`targetSdkVersion` current**: Meet or exceed Google Play's current target API requirement.
  - **Why it matters:** Newer targets opt you into platform security defaults (scoped storage, exported component rules, cleartext restrictions).

- [ ] **`minSdkVersion` reviewed**: Drop versions without security patches where your user base allows. Each version you support may need fallbacks for weaker defaults.

- [ ] **`android:debuggable` false in release**: Gradle release build type does not set `debuggable true`.
  - **How to verify:** `aapt dump badging app-release.apk` or check the merged manifest in Android Studio.

- [ ] **R8 / ProGuard enabled for release**: `minifyEnabled true` and `shrinkResources true`. Upload mapping files to your crash reporter.

- [ ] **Release signing with Play App Signing**: Upload key separate from the app signing key. Keys stored outside the repo.

- [ ] **APK Signature Scheme v2 or later**.

- [ ] **No secrets in `BuildConfig`, `strings.xml`, or `local.properties`**: These end up readable in the APK.

- [ ] **Merged manifest reviewed**: Libraries can add permissions, components, and providers. Check the merged manifest, not just your own.

## 2. Data storage

- [ ] **Keys in Android Keystore**: Generate keys with `KeyGenParameterSpec`. Prefer StrongBox (`setIsStrongBoxBacked(true)`) where available, with fallback to TEE.

- [ ] **Encrypted storage for tokens**: Encrypt with a Keystore key using AES-GCM. Jetpack Security `EncryptedSharedPreferences` is deprecated, so use Tink or your own Keystore-backed wrapper for new code.

- [ ] **Keys bound to user authentication where appropriate**: `setUserAuthenticationRequired(true)` with a timeout or per-use `BiometricPrompt.CryptoObject`.

- [ ] **Internal storage for app files**: Use `Context.getFilesDir()`. No sensitive files on external or shared storage.

- [ ] **No `MODE_WORLD_READABLE` or `MODE_WORLD_WRITEABLE`**.

- [ ] **Backups controlled**: `android:allowBackup="false"` or explicit rules in `android:dataExtractionRules` (Android 12+) and `android:fullBackupContent` that exclude tokens, keys, and sensitive databases.
  - **Why it matters:** Backups and device-to-device transfers can copy app data to other devices.
  - **How to verify:** Review the rules XML. Test backup and restore on a test device and confirm tokens are not restored.

- [ ] **`FLAG_SECURE` on sensitive screens**: Blocks screenshots, screen recording, and the recent apps thumbnail.

- [ ] **Sensitive inputs**: `inputType="textPassword"` and `importantForAutofill` set correctly. `textNoSuggestions` on sensitive fields.

- [ ] **Clipboard**: Mark copied sensitive content with `ClipDescription.EXTRA_IS_SENSITIVE` (Android 13+).

## 3. Network security

- [ ] **Network Security Config defined**: `res/xml/network_security_config.xml` referenced in the manifest.

- [ ] **Cleartext disabled**: `cleartextTrafficPermitted="false"` in base config. No `android:usesCleartextTraffic="true"`.

- [ ] **No user CA trust in release**: Release builds trust only system CAs. Add user CAs only in a `debug-overrides` block.

- [ ] **Pinning (if chosen) in Network Security Config**: `<pin-set>` with a backup pin and an `expiration` date. Or OkHttp `CertificatePinner`.

- [ ] **No custom `TrustManager` or `HostnameVerifier` that accepts everything**.
  - **How to verify:** Grep for `X509TrustManager`, `checkServerTrusted`, `ALLOW_ALL_HOSTNAME_VERIFIER`, `setHostnameVerifier`.

- [ ] **Security provider updated**: Call `ProviderInstaller.installIfNeeded()` (Google Play services) on older devices.

## 4. Exported components and IPC

- [ ] **`android:exported` explicit on every component**: Required on Android 12+. Set `false` unless another app must reach it.
  - **How to verify:** Review the merged manifest for all `activity`, `service`, `receiver`, and `provider` entries.

- [ ] **Exported components protected by permissions**: Use `signature`-level custom permissions for components shared between your own apps.

- [ ] **Content providers locked down**: `exported="false"` unless needed. If exported, set read and write permissions. Use `grantUriPermissions` with specific paths. Use parameterized queries in provider code.

- [ ] **FileProvider paths narrow**: `file_paths.xml` exposes only specific directories, never root (`path="."` on the whole files dir) or external storage root.

- [ ] **Intent data validated**: Treat extras and data URIs in received intents as untrusted.

- [ ] **No implicit intents for sensitive data**: Use explicit intents (set the component or package) when sending data inside your app.

- [ ] **PendingIntents immutable**: `FLAG_IMMUTABLE` unless mutability is required, and then with an explicit intent.

- [ ] **Broadcasts scoped**: Register receivers with `RECEIVER_NOT_EXPORTED` (Android 13+) for internal broadcasts. Send sensitive broadcasts with a permission or use in-process mechanisms.

- [ ] **Task hijacking mitigations**: Avoid unnecessary `taskAffinity` and `launchMode="singleTask"` on sensitive activities.

## 5. Deep links and App Links

- [ ] **App Links verified**: `android:autoVerify="true"` with a valid `/.well-known/assetlinks.json` on your domain.
  - **How to verify:** `adb shell pm get-app-links <package>` shows `verified`.

- [ ] **Custom schemes not used for auth callbacks**: Use verified App Links or the AppAuth redirect handling.

- [ ] **Deep link handlers validate everything**: Host, path, and parameters checked. No loading arbitrary URLs in a WebView from a deep link.

- [ ] **No sensitive actions straight from a link**: Require the user to confirm and be authenticated.

## 6. WebView

- [ ] **JavaScript off unless needed**: `setJavaScriptEnabled(false)` by default.

- [ ] **File access off**: `setAllowFileAccess(false)`, `setAllowContentAccess(false)`. Never enable `setAllowFileAccessFromFileURLs` or `setAllowUniversalAccessFromFileURLs`.

- [ ] **Load only allowlisted HTTPS origins**: Check URLs in `shouldOverrideUrlLoading`. Open other links in the browser.

- [ ] **JavaScript interfaces restricted**: Prefer `WebViewCompat.addWebMessageListener` with an origin allowlist over `addJavascriptInterface`. Expose minimal methods.

- [ ] **Local content via `WebViewAssetLoader`**: Serve bundled content over `https://appassets.androidplatform.net` instead of `file://`.

- [ ] **SSL errors not ignored**: `onReceivedSslError` calls `handler.cancel()`.

- [ ] **Safe Browsing on**: Default enabled. Do not disable it.

- [ ] **WebView data cleared on logout**: Cookies, cache, and storage.

## 7. Authentication and biometrics

- [ ] **`BiometricPrompt` with `CryptoObject`**: Biometric success unlocks a Keystore key used for a real crypto operation.

- [ ] **Strong biometrics for sensitive operations**: `BIOMETRIC_STRONG` authenticator class.

- [ ] **Keys invalidated on biometric enrollment change**: `setInvalidatedByBiometricEnrollment(true)`.

- [ ] **Credential Manager for passkeys and passwords**: Use the Jetpack Credential Manager API for sign-in.

## 8. Integrity and resilience

- [ ] **Play Integrity API**: Request integrity tokens for sensitive actions. Decode and verify on your server. Act on `appIntegrity`, `deviceIntegrity`, and `accountDetails` verdicts according to your risk policy.
  - **Why it matters:** Lets the server tell a genuine install on a genuine device from a modified app or emulator.

- [ ] **Root detection as a risk signal**: Report to the server. Restrict high-risk features rather than blocking the whole app, unless your threat model requires it.

- [ ] **Signature check at runtime**: Compare the signing certificate digest to the expected value and report mismatches.

- [ ] **Native code hardened**: Stack protection, PIE, and RELRO (defaults in current NDK). Keep the NDK current.

## 9. Permissions and privacy

- [ ] **Minimal permissions**: Remove unused permissions from the merged manifest (`tools:node="remove"` for library-added ones you do not need).

- [ ] **Runtime permission requests in context**.

- [ ] **Photo picker instead of storage permissions** where possible.

- [ ] **Approximate location** unless precise is required.

- [ ] **Data safety form matches behavior**.

## 10. Release verification

- [ ] **Static analysis on the release APK/AAB**: Android Lint security checks, plus MobSF or similar run against your own build.

- [ ] **Manifest review on every release**: Diff the merged manifest against the previous release.

- [ ] **Secrets scan on the APK**: Unzip and scan resources, assets, and decompiled code.

- [ ] **Dependency scan**: Gradle dependencies and Android SDKs.
