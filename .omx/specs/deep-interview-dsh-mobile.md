# DSH Mobile Execution Specification

## Metadata

- Profile: standard
- Context: greenfield mobile application
- Final ambiguity: 0.12
- Threshold: 0.20
- Context snapshot: `.omx/context/dsh-mobile-20260816T060000Z.md`

## Intent

Provide a trustworthy mobile shell for remotely operating the user's own local DeepSeek Harness without exposing the local DSH listener.

## In scope

- Flutter iOS and Android application under `dsh-mobile/`.
- Email/password registration, login, secure token storage, rotation, and silent startup authentication.
- QR scanner and manual six-digit pairing.
- Device list, status, rename, unbind, refresh, and empty/error/loading states.
- Ticket-authenticated `flutter_inappwebview` container, same-origin navigation, external-browser routing, Cookie-session refresh, Android back behavior, and explicit offline states.
- Fixed light DSH product visual system.
- Mock and real Relay data sources.
- MVP analytics abstraction for the six documented events.
- Unit, widget, and build verification.

## Out of scope

- Native conversation UI, push notifications, native approvals, biometrics, account switching, tablet landscape, end-to-end encryption, and custom themes.

## Decision boundaries

Internal architecture and presentation details may be selected without further approval. Cross-team contracts, credential boundaries, product scope, and security claims may not be changed silently.

## Acceptance criteria

1. New users can register, log in, pair using QR or manual code, and see the device after confirmation.
2. Tokens live only in platform secure storage and a 401 triggers one shared refresh operation.
3. Device states map to online, DSH offline, and computer offline UI.
4. Rename and unbind refresh the device list and display contextual failures.
5. WebView uses a one-time ticket and Cookie, preserves WebSocket traffic, opens external origins in the system browser, and refreshes an expired session silently.
6. Android back navigates WebView history before leaving the page.
7. Mock mode supports deterministic development and widget tests.
8. Formatting, static analysis, tests, and platform builds pass or any environment-only gap is explicitly reported.
