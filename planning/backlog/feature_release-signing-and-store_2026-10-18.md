---
kind: stub
status: open
date: 2026-10-18
---

# Release Signing and Distribution

**Discovered During:** personal-thoughts-maui-app_2026-10-18

## Context
The MVP ships as a sideloaded debug or release APK from CI.

## Known Scope
Create a keystore (a person has to do this), store the signing secrets in CI, build a signed AAB, and decide on Play Store or sideload-only.

## Trigger for Promotion
When the app is ready to install permanently on the user's phone.
