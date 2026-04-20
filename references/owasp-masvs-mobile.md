# OWASP MASVS Mobile Security — Reference

Source: OWASP Mobile Application Security Verification Standard (MASVS) v2. Focus on Critical and High findings.

---

## MASVS-STORAGE — Sensitive Data Storage

**What to look for:**
- PII, credentials, or tokens stored in plaintext on device
- Sensitive data written to shared storage (external storage, system clipboard, logs, screenshots)
- Data stored in `SharedPreferences` or `NSUserDefaults` unencrypted
- Sensitive data visible in app screenshots / recents

**Code patterns:**
```kotlin
// Android: plaintext in SharedPreferences
prefs.edit().putString("auth_token", token).apply()

// iOS: sensitive data in UserDefaults (not Keychain)
UserDefaults.standard.set(authToken, forKey: "auth_token")

// Logging credentials
Log.d("Auth", "Token: $token")
```

**Severity:** Auth tokens or passwords in plaintext storage → **Critical**. PII in shared/external storage → High.

**Mitigation:** Use Android Keystore / iOS Keychain for credentials and tokens. Encrypt sensitive data at rest. Exclude sensitive views from screenshots (FLAG_SECURE / sensitive content attribute).

---

## MASVS-CRYPTO — Cryptography

**What to look for:**
- Custom crypto implementations
- Hardcoded encryption keys
- Weak algorithms (DES, 3DES, RC4, MD5 for crypto purposes, ECB mode)
- Insufficient key length (RSA < 2048-bit, AES < 128-bit)
- Predictable IV (static IV for AES-CBC)

**Code patterns:**
```java
// Hardcoded key
SecretKeySpec key = new SecretKeySpec("hardcoded_key123".getBytes(), "AES");

// ECB mode
Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");

// Static IV
byte[] iv = "0000000000000000".getBytes();
```

**Severity:** Hardcoded encryption key → **Critical**. ECB mode for user data → High.

**Mitigation:** Key derivation via Keystore/Keychain; AES-GCM with random IV; PBKDF2/Argon2 for key derivation from passwords.

---

## MASVS-AUTH — Authentication and Session Management

**What to look for:**
- Biometric authentication that falls back to password without re-authentication
- Auth tokens with excessive lifetime stored on device
- No session invalidation on logout
- Authentication state stored in client-controlled storage (not Keystore/Keychain)
- Missing certificate pinning for auth endpoints

**Severity:** Auth bypass or token theft enables full account takeover → **Critical**.

---

## MASVS-NETWORK — Network Communication

**What to look for:**
- HTTP (not HTTPS) for any communication
- Certificate validation disabled (`TrustManager` that accepts any cert, `AllowsAnyHTTPSCertificate`)
- Missing certificate pinning for high-value endpoints (payment, auth)
- Sending sensitive data in URL query parameters (appears in proxy logs)

**Code patterns:**
```java
// Android: trust all certificates (MITM-vulnerable)
TrustManager[] trustAllCerts = new TrustManager[]{
    new X509TrustManager() {
        public void checkClientTrusted(X509Certificate[] chain, String authType) {}
        public void checkServerTrusted(X509Certificate[] chain, String authType) {}
    }
};

// iOS: disabling ATS
NSAllowsArbitraryLoads = YES  // in Info.plist
```

**Severity:** Disabled certificate validation → **Critical** (MITM attack enables full traffic inspection). HTTP for auth/payment → **Critical**.

**Mitigation:** Enforce HTTPS; use system certificate store; implement certificate pinning for auth and payment endpoints; remove any trust-all certificate bypass code.

---

## MASVS-PLATFORM — Platform Interaction

**What to look for:**
- WebView with JavaScript enabled loading untrusted URLs
- `addJavascriptInterface` exposing native code to web content
- Deep link / intent handling that doesn't validate origin
- Exported activities, services, or broadcast receivers that can be invoked by other apps

**Code patterns:**
```java
// Android: exposed WebView with JS
webView.getSettings().setJavaScriptEnabled(true);
webView.addJavascriptInterface(new NativeBridge(), "Android");
webView.loadUrl(intent.getStringExtra("url"));  // user-controlled URL

// Exported activity
<activity android:name=".AdminActivity" android:exported="true" />
```

**Severity:** WebView JS bridge with unrestricted URL loading → **Critical**. Exported admin activity → **Critical**.

**Mitigation:** Validate and whitelist URLs loaded in WebView; disable `addJavascriptInterface` if not needed; use `android:exported="false"` for activities not meant for external use; validate deep link origins.

---

## MASVS-CODE — Code Quality

**What to look for:**
- Free (C/C++) memory used after free, buffer overflow
- Logging of sensitive data in production
- Hardcoded passwords, API keys, or internal endpoints in the binary
- Enabled debug builds / debug flags in production

**Severity:** Hardcoded API key in production binary → **Critical** (binary can be extracted). Debug mode in production app → High.

**Mitigation:** Remove all hardcoded secrets; use build configuration flags to strip debug code in release builds; run `strings` or a secrets-scanning tool on the release binary.
