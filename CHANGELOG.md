# Changelog

## 0.1.7 / 0.1.6 - 2026-08-19

### Browser and Companion access

- Added independent browser enrollment for an already paired computer.
- Added a local DSH settings action to generate a browser-access QR code and link.
- Browser enrollment accepts both the normal computer pairing QR and the browser-access QR.
- Phone, iPhone Safari, and additional browsers can use the same computer concurrently without replacing each other's E2EE key.
- Added Companion update checks and a local one-click updater. DSH must be restarted after installation.

### Security

- Browser E2EE keys remain in the URL fragment during enrollment and are stored only in the current browser's IndexedDB.
- Browser update actions are local-only and use validated tags from the official plugin repository.
- Relay continues to route opaque sealed frames without storing DSH request or response content.

### Releases

- Companion: [v0.1.7](https://github.com/april-jk/dsh-mobile-plugin/releases/tag/v0.1.7)
- Relay: [v0.1.6](https://github.com/april-jk/dsh-relay/releases/tag/v0.1.6)

The unified Suite release remains `v0.1.5` until the Android, Relay, Companion, and website component versions are aligned for a single signed release.
