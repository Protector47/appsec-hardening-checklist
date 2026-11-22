# Mobile App Security Checklist (Cross-Platform)

Controls that apply to both Android and iOS, including React Native, Flutter, and other cross-platform apps. Organized by [OWASP MASVS v2](https://mas.owasp.org/MASVS/) control groups. Platform specifics are in [mobile-android.md](mobile-android.md) and [mobile-ios.md](mobile-ios.md). Verification steps follow the [OWASP MASTG](https://mas.owasp.org/MASTG/) and should be run against your own builds.

Starting point: treat the mobile app as a public client. Anything shipped in the binary can be extracted. Security decisions belong on the server.

## MASVS-STORAGE: Data storage

- [ ] **Store as little as possible on the device**: Cache only what offline use needs. Clear caches on logout.

- [ ] **Secrets and tokens in platform secure storage**: Android Keystore-backed encryption, iOS Keychain. Not in plain preferences, files, or SQLite.
  - **Cross-platform:** Use maintained wrappers (`flutter_secure_storage`, `react-native-keychain`, `expo-secure-store`) and confirm they use the hardware-backed store on both platforms.

- [ ] **No sensitive data in logs**: Strip debug logging from release builds. No tokens, PII, or request bodies in `Log`, `NSLog`, `console.log`, or `print`.
  - **How to verify:** Run the release build with device logs open (`adb logcat`, Xcode console) and go through login and key flows.

- [ ] **No sensitive data in screenshots and app switcher**: Mask sensitive screens when the app goes to background.

- [ ] **Keyboard caching off for sensitive fields**: Mark password and card fields as secure text. Disable autocorrect and suggestions on sensitive inputs.

- [ ] **Clipboard use limited**: Do not copy secrets to the clipboard. If needed (for example OTP or recovery codes), expire the clipboard content.

- [ ] **Sensitive data excluded from backups**: See platform checklists.

- [ ] **Local databases encrypted when they hold sensitive data**: SQLCipher or platform encryption with keys in secure storage.

- [ ] **Data wiped on logout and account deletion**: Tokens, caches, databases, files, and WebView storage.

## MASVS-CRYPTO: Cryptography

- [ ] **Use platform crypto APIs**: Keystore and CryptoKit / CommonCrypto. No custom algorithms.

- [ ] **No hardcoded keys**: Keys generated on device in secure hardware or delivered by the server per user. A key in the binary is public.
  - **How to verify:** Run a secret scanner on the built APK/IPA and on the JS bundle or Dart snapshot.

- [ ] **Modern algorithms**: AES-GCM, ChaCha20-Poly1305, ECDSA P-256, Ed25519, SHA-256 or better.

- [ ] **Secure random**: `SecureRandom`, `SecRandomCopyBytes`, or the framework's CSPRNG.

## MASVS-AUTH: Authentication and authorization

- [ ] **Authentication enforced on the server**: The app never decides on its own whether a user is logged in or entitled. Every API call is authorized server-side.

- [ ] **OAuth 2.0 with PKCE through the system browser**: Use AppAuth or the platform auth session (Custom Tabs, `ASWebAuthenticationSession`). No embedded WebView login for third-party IdPs.
  - **Why it matters:** Embedded WebViews let the app see the user's IdP password and break SSO protections.

- [ ] **No client secrets in the app**: Mobile apps are public clients.

- [ ] **Short-lived access tokens, rotating refresh tokens**: Refresh tokens stored in secure storage, revoked on logout.

- [ ] **Biometrics unlock a key, not a boolean**: Biometric success should release a Keystore or Keychain key that is needed to decrypt the refresh token. A biometric check that only returns true or false can be bypassed on a compromised device.

- [ ] **Re-auth for sensitive actions**: Biometric or PIN prompt before payments, changing credentials, or viewing highly sensitive data.

- [ ] **Session ends server-side on logout**.

## MASVS-NETWORK: Network communication

- [ ] **TLS only**: No cleartext HTTP to any host. Platform configs enforce it (see platform checklists).

- [ ] **Default certificate validation intact**: No custom trust managers that accept all certificates, no hostname verification overrides, no `badCertificateCallback` returning true in Flutter.
  - **How to verify:** Grep source and dependencies. Test against a host with an invalid certificate in a lab and confirm the connection fails.

- [ ] **Certificate pinning decision documented**: Pinning raises the bar for interception but adds outage risk if keys rotate.
  - **Secure default:** Pin to the public key (SPKI) of your intermediate or leaf, with at least one backup pin. Have a remote config or app update path to change pins. Many apps skip pinning and rely on TLS plus server-side controls. Either choice is fine if it is deliberate.

- [ ] **Third-party SDK traffic reviewed**: Know which hosts your SDKs contact and what data they send.

- [ ] **No sensitive data in URLs**: Tokens and PII in headers or body.

## MASVS-PLATFORM: Platform interaction

- [ ] **Deep links validated**: Treat every deep link parameter as untrusted input. Deep links cannot trigger sensitive actions without confirmation and auth.

- [ ] **Verified app links**: Android App Links and iOS Universal Links for auth callbacks and sensitive routes, instead of custom URL schemes.
  - **Why it matters:** Any app can register the same custom scheme.

- [ ] **WebView hardened**: JavaScript off unless needed. Load only your own HTTPS content. No file access. JS bridges exposed only to your own origins.

- [ ] **IPC limited**: Only expose components or extensions that must be reachable by other apps, with permission checks.

- [ ] **Permissions minimal**: Request runtime permissions only when needed and explain why.

## MASVS-CODE: Code quality

- [ ] **Minimum OS version supported by the vendor**: Drop OS versions that no longer get security updates where your user base allows.

- [ ] **Dependencies scanned**: Gradle, CocoaPods, SPM, npm, and pub dependencies in SCA. SDKs updated regularly.

- [ ] **Input from all sources validated**: Server responses, deep links, QR codes, files, clipboard, and IPC.

- [ ] **Forced update path**: Server can require a minimum app version so you can retire vulnerable releases.

- [ ] **Debug features stripped**: Debug menus, test endpoints, and staging toggles removed from release builds, not just hidden.

- [ ] **Crash reports scrubbed**: No PII or tokens in crash reporter breadcrumbs.

## MASVS-RESILIENCE: Reverse engineering and tampering

These controls slow down analysis. They do not replace server-side security. Decide based on risk (banking, DRM, games, high-fraud apps get more).

- [ ] **Threat model decides resilience level**: Write down what you protect (fraud, IP, cheating) and how much effort is worth it.

- [ ] **Code obfuscation for release builds**: R8 / ProGuard on Android. Symbol stripping on iOS. Obfuscation flags for Flutter (`--obfuscate --split-debug-info`). Hermes bytecode for React Native.

- [ ] **Device integrity attestation on the server**: Play Integrity API and App Attest / DeviceCheck. Verify attestation tokens on the server, not in the app.
  - **Why it matters:** Server-side verification is much harder to fake than in-app checks.

- [ ] **Root and jailbreak detection as a signal, not a gate**: Detection can be bypassed. Use it to raise risk scores or limit high-risk features, and report it to the server. Blocking all rooted devices also blocks legitimate power users.

- [ ] **Tamper and repackaging checks**: Verify signing certificate at runtime and report mismatches to the server.

- [ ] **Anti-debugging for high-risk apps only**: Adds complexity and support burden. Use only if the threat model calls for it.

## MASVS-PRIVACY: Privacy

- [ ] **Data minimization**: Collect only what the feature needs.

- [ ] **SDK inventory**: Each analytics and ad SDK has an owner, a purpose, and a data list.

- [ ] **Store privacy labels accurate**: Google Play Data safety and Apple privacy nutrition labels match actual behavior, including SDKs.

- [ ] **User controls**: Users can see, export, and delete their data. Consent is collected before tracking where required.

- [ ] **Identifiers limited**: Avoid persistent hardware identifiers. Use app-scoped IDs.
