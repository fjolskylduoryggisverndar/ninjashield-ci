# ninjashield CI

Build workflows for the **ninjashield** Flutter VPN app. Same pipeline as BuddhaJump's,
including the store-publish gating: an ordinary run builds downloadable
artifacts only, and the Apple/Microsoft store-publish steps are skipped unless
explicitly requested via the `workflow_dispatch` inputs.

The workflows are brand-agnostic — app name, Android `applicationId` and
version are read from the checked-out app repo (`pubspec.yaml`,
`android/app/build.gradle.kts`).

## Required repository secrets

Checkout:
- `PRIVATE_REPO` — `BuddhaJumpApp/ninjashield`
- `ORG_TOKEN` — read-only org token used to check out that private repo

Android / Google Play:
- `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_PASSWORD`, `ANDROID_KEY_ALIAS`
- `GOOGLE_SERVICE_ACCOUNT_BASE64` — Play Console service-account JSON (only needed to upload)

Apple:
- `APPLE_PKCS12_BASE64`, `APPLE_PKCS12_PASSWORD`, `PKCS12_NAME`
- `APPLE_PROVISIONING_BASE64` — provisioning profiles for `com.fjolsky.ninjashield` and `com.fjolsky.ninjashield.service`
- `APPLE_API_KEY_BASE64`, `APPLE_API_KEY_ID`, `APPLE_API_ISSUER_ID`

Microsoft Partner Center (only for the Windows Store path):
- `PARTNER_TENANT`, `PARTNER_SELLER`, `PARTNER_CLIENT`, `PARTNER_SECRET`, `PARTNER_STORE`, `PARTNER_APP`

Until these exist, run the workflow with both store flags **false** — it will
still produce a downloadable APK/AAB, IPA, macOS `.app` zip and a portable
Windows zip on the GitHub release.
