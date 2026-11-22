# iOS App Security Checklist

iOS-specific controls, aligned with [OWASP MASVS v2](https://mas.owasp.org/MASVS/) and the [MASTG](https://mas.owasp.org/MASTG/) iOS tests. Use alongside [mobile-common.md](mobile-common.md). Verify against your own debug and release builds.

## 1. Build configuration

- [ ] **Deployment target reviewed**: Support iOS versions that still receive security updates where your user base allows.

- [ ] **Release builds strip debug symbols**: Upload dSYMs to your crash reporter instead of shipping them.

- [ ] **No `get-task-allow` in release entitlements**: Only development builds allow debugger attach.
  - **How to verify:** `codesign -d --entitlements - YourApp.app` on the release build.

- [ ] **Entitlements minimal**: Only the capabilities you use (App Groups, Keychain sharing, Associated Domains, push).

- [ ] **No secrets in `Info.plist`, xcconfig, or bundled files**: These are readable in the IPA.

- [ ] **Compiler protections on**: PIE, ARC, and stack canaries are on by default in current Xcode. Do not disable them.

- [ ] **Third-party SDKs have privacy manifests**: `PrivacyInfo.xcprivacy` present and required-reason APIs declared.

## 2. Keychain

- [ ] **Tokens and secrets in the Keychain**: Never in `UserDefaults`, plist files, or Core Data without encryption.

- [ ] **Accessibility class chosen deliberately**: `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` for most secrets. `AfterFirstUnlockThisDeviceOnly` only for items needed by background tasks.
  - **Why it matters:** `ThisDeviceOnly` items are not included in backups or migrated to new devices.

- [ ] **Avoid `kSecAttrAccessibleAlways`**: Deprecated and unsafe.

- [ ] **Access control for high-value items**: `SecAccessControlCreateWithFlags` with `.biometryCurrentSet` or `.userPresence` for items that need user presence.

- [ ] **Keychain cleared on first launch after reinstall**: Keychain items survive app deletion. Track first launch in `UserDefaults` and clear stale Keychain items.
  - **Why it matters:** Otherwise a reinstalled app can pick up the previous user's tokens.

- [ ] **Keychain access groups scoped**: Share items only between your own apps that need them.

## 3. Secure Enclave and cryptography

- [ ] **Private keys in the Secure Enclave where possible**: `kSecAttrTokenIDSecureEnclave` or CryptoKit `SecureEnclave.P256`. The key never leaves hardware.

- [ ] **CryptoKit for new code**: `AES.GCM`, `ChaChaPoly`, `P256`, `Curve25519`, `SHA256`.

- [ ] **No CommonCrypto ECB or static IVs**.

- [ ] **`SecRandomCopyBytes` or CryptoKit for randomness**.

## 4. Data protection

- [ ] **File protection class set**: `NSFileProtectionComplete` for sensitive files. Set the default in the Data Protection entitlement.
  - **Why it matters:** `Complete` makes files unreadable while the device is locked.

- [ ] **Core Data and SQLite protected**: Set `NSPersistentStoreFileProtectionKey` or the file attribute on the store.

- [ ] **Sensitive files excluded from backup**: `isExcludedFromBackup` on files that must not be in iCloud or local backups.

- [ ] **Caches controlled**: `URLCache` disabled or cleared for sensitive API responses. Use `.ephemeral` `URLSessionConfiguration` where appropriate.

- [ ] **App switcher snapshot masked**: Show a placeholder view in `sceneWillResignActive` / `applicationWillResignActive`.

- [ ] **Screen capture awareness**: Respond to `UIScreen.capturedDidChangeNotification` for highly sensitive screens.

- [ ] **Secure text entry**: `isSecureTextEntry = true` for passwords and secrets. Set `textContentType` correctly so password managers work.

- [ ] **Third-party keyboards blocked for sensitive apps**: Implement `application(_:shouldAllowExtensionPointIdentifier:)` to reject custom keyboards if your threat model requires it.

## 5. Pasteboard

- [ ] **No secrets on the general pasteboard**.

- [ ] **Local-only and expiring items**: When copying sensitive values, use `UIPasteboard.general.setItems(_:options:)` with `.localOnly` and `.expirationDate`.
  - **Why it matters:** Universal Clipboard syncs pasteboard content to the user's other devices.

- [ ] **Read pasteboard only on user action**: Avoid silent reads, which also trigger the iOS paste notification.

## 6. App Transport Security

- [ ] **ATS on with no global exceptions**: No `NSAllowsArbitraryLoads` in `Info.plist`.
  - **How to verify:** Inspect `Info.plist` in the release build for `NSAppTransportSecurity` keys.

- [ ] **Per-domain exceptions documented**: Any `NSExceptionDomains` entry has an owner and a removal plan.

- [ ] **No `NSAllowsArbitraryLoadsInWebContent`** unless the app is a browser.

- [ ] **Pinning (if chosen)**: `NSPinnedDomains` in `Info.plist` (iOS 14+) or `URLSessionDelegate` SPKI pinning with backup pins.

- [ ] **No trust evaluation overrides**: `urlSession(_:didReceive:completionHandler:)` does not accept all server trusts.
  - **How to verify:** Grep for `.useCredential` with `serverTrust` without proper `SecTrustEvaluateWithError` checks.

## 7. URL schemes and Universal Links

- [ ] **Universal Links for sensitive entry points**: Associated Domains with a valid `apple-app-site-association` file.

- [ ] **Custom URL schemes treated as public**: Any app can register the same scheme. Do not carry tokens or trigger sensitive actions through them.

- [ ] **OAuth callbacks through `ASWebAuthenticationSession`**: The callback is delivered to the session that started it.

- [ ] **Incoming URLs validated**: `application(_:open:options:)` and `scene(_:openURLContexts:)` check host, path, and parameters.

- [ ] **No sensitive actions straight from a link**: Require authentication and user confirmation.

## 8. WebViews

- [ ] **`WKWebView` only**: `UIWebView` is removed from the SDK.

- [ ] **JavaScript off when not needed**: `WKWebpagePreferences.allowsContentJavaScript = false`.

- [ ] **Navigation allowlist**: `webView(_:decidePolicyFor:decisionHandler:)` allows only your HTTPS origins.

- [ ] **Message handlers scoped**: `WKScriptMessageHandler` checks `message.frameInfo.securityOrigin` before acting. Expose minimal handlers.

- [ ] **No file URL access to app data**: Restrict `loadFileURL(_:allowingReadAccessTo:)` to a specific directory.

- [ ] **Website data cleared on logout**: `WKWebsiteDataStore.default().removeData(...)` or use a non-persistent store.

## 9. Authentication and biometrics

- [ ] **Biometrics tied to Keychain items**: Use `SecAccessControl` with `.biometryCurrentSet` so a Keychain read needs Face ID or Touch ID. Do not rely only on `LAContext.evaluatePolicy` returning success.
  - **Why it matters:** A boolean result can be hooked on a jailbroken device. A Keychain item protected by the Secure Enclave cannot be read without the biometric match.

- [ ] **`.biometryCurrentSet` for high-value items**: New fingerprints or faces invalidate the item.

- [ ] **`NSFaceIDUsageDescription` set** with a clear reason.

- [ ] **Passkeys via `ASAuthorizationController`**: Use platform passkey support for sign-in.

- [ ] **Sign in with Apple handled server-side**: Verify the identity token on your server.

## 10. Integrity and resilience

- [ ] **App Attest for sensitive actions**: `DCAppAttestService` to attest keys, assertions verified on your server.

- [ ] **DeviceCheck for per-device state**: For example flagging devices that abused a promo.

- [ ] **Jailbreak detection as a risk signal**: Report to the server and adjust risk. Detection is bypassable, so do not rely on it alone. Blocking all jailbroken devices also blocks legitimate users.

- [ ] **Obfuscation where the threat model calls for it**: Swift symbol stripping, string protection for sensitive constants. Do not treat it as a security boundary.

## 11. Privacy

- [ ] **App Tracking Transparency**: Ask before tracking across apps and websites.

- [ ] **Privacy nutrition label accurate**, including third-party SDKs.

- [ ] **Privacy manifest complete**: Required-reason APIs declared with valid reasons.

- [ ] **Approximate location** unless precise is needed.

- [ ] **Limited Photos access** supported: Use `PHPickerViewController` so you do not need full library access.

## 12. Release verification

- [ ] **Inspect the release IPA**: Check `Info.plist`, entitlements, embedded frameworks, and bundled files.

- [ ] **Secrets scan on the IPA contents**.

- [ ] **Dependency scan**: CocoaPods, SPM, and any JS or Dart dependencies.

- [ ] **Static analysis**: MobSF or similar against your own build.
