# Smile ID v12 React Native / Expo — integration blockers

**Partner ID:** 8711 · **Environment:** sandbox (`testapi.smileidentity.com`) · **Product:** `biometric_kyc`

**SDK:** `@smileid/usesmileid@12.0.1` · **Platforms:** Android (Samsung, Android 15) and iOS 18
**Host:** Expo SDK 57, React Native 0.86.2, React 19.2.3, `expo-modules-core` 57.0.10

---

## Summary

We cannot complete a biometric KYC submission. Two problems, one of which we believe is a
packaging bug in the published SDK:

1. **No published package registers the `UseSmileIDBridge` Expo native module**, which the SDK
   requires at runtime. Without it the flow throws during render and KYC cannot start at all. We
   have worked around it, and that workaround is probably the cause of (2).
2. **Every submission returns `401 Invalid authentication credentials.`** while the *same* token,
   sent by `curl`, authenticates successfully. We have eliminated the token, the environment, the
   partner ID, expiry and clock skew. What remains are the two signed headers the SDK adds, both
   of which depend on values we cannot obtain because of (1).

We would rather not keep guessing, so the questions are collected at the end.

---

## Issue 1 — `UseSmileIDBridge` is required but not published (blocking)

`@smileid/usesmileid_bridge` resolves a native module by name:

```ts
// src/security/native_module.ts
export const BRIDGE_NATIVE_MODULE_NAME = "UseSmileIDBridge";
```

Four providers depend on it (signing secret, Sentry DSN, privileges, device integrity), plus the
attestation API. `SigningSecretProvider.signingSecret()` throws when it is absent, and
`makeNetworkingFactory` calls it **unconditionally** while constructing the API client — from
inside a `useMemo` during render:

```
Error: UseSmileIDBridge native module is not available; cannot read signing secret
    at signingSecret (signing_secret_provider.ts)
    at makeNetworkingFactory (networking_factory.ts:150)
    at FlowSuccessRenderer (use_smile_id_flow.tsx:213)
```

In production this renders a blank white screen, because `FlowInvalidDispatcher` returns `null`
and reports only through `onResult.failure`.

**No published package provides that module.** Verified against 12.0.1, the latest on npm:

| Package | `ios/` | `android/` | `expo-module.config.json` |
|---|---|---|---|
| `@smileid/usesmileid` | ✗ | ✗ | ✗ |
| `@smileid/usesmileid_bridge` | ✗ | ✗ | ✗ |
| `@smileid/usesmileid_platform_interface` | ✗ | ✗ | ✗ |
| `@smileid/usesmileid_vision_face` | ✓ | ✗ | ✓ |
| `@smileid/usesmileid_vision_document` | ✓ | ✗ | ✓ |
| `@smileid/usesmileid_mlkit_face` | ✗ | ✓ | ✓ |
| `@smileid/usesmileid_mlkit_document` | ✗ | ✓ | ✓ |

The provider packages show what a package registering an Expo module looks like. The core and
bridge packages publish `files: ["dist","src","README.md","LICENSE","CHANGELOG.md"]` — JavaScript
only, across every published version. There is no v12 React Native repository on GitHub
(`smileidentity/react-native-v11` and `react-native-expo-v11` are v11), and no newer release.

A secondary inconsistency: `resolveBridgeNativeModule` is written to return `undefined` when the
module is not linked, documented as *"graceful degradation"*. But its only production consumer
turns that `undefined` into a throw, on a code path that always runs. The degradation the resolver
implements is unreachable.

### What we did

We wrote a local Expo module registering `Name("UseSmileIDBridge")` and forwarding to the public
API of the real Smile ID binaries, which *are* linked into our app:

| | iOS (`UseSmileIDBridge.xcframework`, SPM) | Android (`com.usesmileid:bridge` AAR) |
|---|---|---|
| Signing secret | `ArkanaKeys.Release().sIGNING_SECRET` | `ArkanaKeys.Global.sIGNING_SECRET` |
| Sentry DSN | `ArkanaKeys.Global().sENTRY_DSN` | `ArkanaKeys.Global.sENTRY_DSN` |
| Privileges | `UseSmileIDPrivileges.privileges()` | `UseSmileIDPrivileges.compute(context)` |
| Device signals | `UseSmileIDDeviceSignals.*` | `UseSmileIDDeviceSignals.*` |
| Attestation | `UseSmileIDAppAttestManager.shared` | `UseSmileIDIntegrityManager` |
| Hardware attestation | `DCAppAttestService.isSupported` | `UseSmileIDHardwareKeyAttestation` |

This gets the flow rendering and capture working end to end on both platforms.

---

## Issue 2 — Is the React Native signing secret different from the published Android/iOS keys?

Your own docstring suggests our workaround cannot be correct:

> The secret is obfuscated in the native `ArkanaKeys` generated into this plugin (so it versions
> with the RN SDK release, **rather than reusing the published Android/iOS keys**); this wrapper
> vends it over the native Expo module. The SDK's request-MAC signing keys its PBKDF2 derivation
> with this value.
>
> — `@smileid/usesmileid_bridge/src/security/signing_secret_provider.ts`

If the React Native SDK has its own Arkana key set, then the `sIGNING_SECRET` we read from the
published Android AAR / iOS xcframework is the wrong key, our `smileid-request-mac` is derived
from it, and the server would be right to reject the request. **We have no way to obtain the
correct value**, because it only exists in the unpublished module from Issue 1.

This is our leading explanation for the 401 below, but we cannot confirm it from the client.

---

## Issue 3 — Play Integrity warm-up fails with HTTP 429 on the shared cloud project

`Attestation.warmUp()` fails on every attempt, so the SDK stamps `smileid-device-nonce: ERR_*`
rather than a real token. Meanwhile the session token carries `policy: 15` — all four bits set,
including bit 3, the Play Integrity check.

The cause is only visible in the platform log, not through the SDK (see Issue 4):

```
PlayCore: StandardIntegrity : warmUpIntegrityToken(610984185237)
PlayCore: StandardIntegrity : ServiceConnectionImpl.onServiceConnected(...ExpressIntegrityService)
Finsky  : Integrity key attestation record generated successfully.
Volley  : Unexpected response code 429 for https://play-fe.googleapis.com/fdfe/intermediateIntegrity
Finsky  : com.google.android.finsky.expressintegrityservice.ExpressIntegrityException
PlayCore: OnWarmUpIntegrityTokenCallback : onWarmUpExpressIntegrityToken
```

`610984185237` is the cloud project number read from `ArkanaKeys.Global.getGOOGLE_CLOUD_PROJECT_NUMBER()`
in the bridge AAR and passed to `PrepareIntegrityTokenRequest.Builder.setCloudProjectNumber(...)`
by `com.usesmileid.bridge.internal` during `warmUpTokenProvider()`.

Note what this is **not**: the project number is valid (an invalid one fails differently), the
device has a healthy Play Store, service binding succeeds, and key attestation records generate
normally. The request reaches Google and is **rate limited**.

Because that cloud project belongs to Smile ID rather than to the partner, a partner has no way to
inspect or raise the quota. Some of our 429s are likely self-inflicted from repeated testing, but
we cannot tell our share from the project-wide load.

---

## Issue 4 — The Play Integrity error is swallowed before it reaches the partner

`warmUpTokenProvider()` wraps the underlying failure in an exception whose message is the generic
*"Failed to warm up integrity token provider"*. The Play Integrity error — the part that
distinguishes a rate limit from an invalid project from an unrecognised app — is only in the
`cause`, which is not surfaced.

Everything a partner sees is:

```
AttestationWarmupFailedException: Failed to warm up integrity token provider
```

We found the 429 by reading `logcat` for `PlayCore`/`Finsky`/`Volley`. Propagating the cause chain
into `errorMessage` would have made Issue 3 self-diagnosing — `mapError` in `attestation.ts`
already passes `errorMessage` through to the thrown exception, so the channel exists.

---

## Issue 5 — Double slash in submission URLs (cosmetic)

Every request goes to `https://testapi.smileidentity.com//v3/biometric_kyc`. `SmileIDUrls.sandbox`
ends with a trailing slash and every path constant in `smile_id_api.ts` begins with `/v3/`, so
axios joins them into `//v3`. It is being normalised today and is not causing our failure, but it
is worth fixing before it meets a stricter gateway.

---

## The blocking symptom, and what we have eliminated

Every submission:

```
POST https://testapi.smileidentity.com//v3/biometric_kyc
→ 401 { "message": "Invalid authentication credentials.", "status": "Unauthorized" }
```

The same token, minted by the same backend call, sent with `curl` carrying only `smileid-token`
and `smileid-partner-id`:

```
→ 400 { "message": "consent is required", "status": "Bad Request" }
```

A `400` means authentication **passed** and the request reached business validation. So:

| Hypothesis | Status | Evidence |
|---|---|---|
| Wrong environment | Eliminated | App posts to `testapi`, session says `use_sandbox: true`, token claims `api_url: https://testapi.smileidentity.com/v3` |
| Invalid / wrong-partner token | Eliminated | Same token reaches `400 consent is required` |
| Token expiry | Eliminated | `exp - iat` = 900s; `ttl_left` logged at 899 on mint and 899 at capture start, submission ~17s later |
| Stale or cached token | Eliminated | Raw JWT logged at mint and at flow mount is byte-identical; each refresh mints a new one |
| Clock skew | Eliminated | `smileid-request-timestamp` matches UTC on a correct device clock |
| Gateway rejection | Eliminated | Body differs from the API Gateway's `AccessDeniedException` envelope — this is an application-layer rejection |
| **`smileid-request-mac`** | **Open** | Derived from a signing secret we believe is the wrong one — Issue 2 |
| **`smileid-device-nonce`** | **Open** | Ships as `ERR_*` because of Issue 3, while `policy: 15` requires attestation |

Both remaining candidates trace back to Issue 1.

For completeness: `curl` probes carrying a deliberately bogus `smileid-request-mac`, or
`smileid-device-nonce: ERR_test`, still returned `400 consent is required`. We do not read that as
evidence those headers are ignored — an incomplete body appears to be rejected before signature
validation, so those probes never reached the relevant stage. The app's request has a complete
multipart body and is the only one that gets far enough to be rejected with a 401.

---

## What we are asking

1. **Which package is supposed to register the `UseSmileIDBridge` Expo module?** If it is
   unpublished, is there a release timeline, or a supported way to obtain it now?
2. **Does the React Native SDK use a different Arkana signing secret from the published
   Android/iOS SDKs?** A yes confirms our workaround cannot produce a valid MAC and closes this
   line of investigation immediately.
3. **What is the expected Play Integrity quota behaviour for partner apps on cloud project
   `610984185237`?** Is the 429 a per-device throttle we caused, or project-wide?
4. **Can `policy` bits 0 (payload signing) and 3 (Play Integrity) be cleared for partner 8711 in
   sandbox?** That would remove both suspect headers and let us verify the rest of the integration
   while the above is resolved. This is our preferred short-term unblock.
5. **What does the API return when `smileid-request-mac` fails validation, versus when
   `smileid-device-nonce` is an `ERR_*` value?** If either is `Invalid authentication credentials.`,
   that alone identifies which of the two is failing.

We are happy to supply a full `logcat` capture, the raw failing JWT, or a test build.

