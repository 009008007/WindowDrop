# Validation — WindowDrop 0.1 preview

Built on an Apple Silicon Mac running macOS 26.5.1. The bundle requires macOS 15 or later because the bundled official cloudflared binary has that minimum OS version. Other supported macOS releases and recipient browsers have not been individually tested.

## Passed

- Native Swift/AppKit frontend compiled for arm64 with a macOS 15 deployment target.
- Self-contained Python service built with PyInstaller; it runs without relying on the development virtual environment.
- App bundle locally signed; `codesign --verify --deep --strict` passed.
- Official cloudflared 2026.8.3 download verified against the GitHub release-asset SHA-256 digest. Original and packaged checksums are in THIRD-PARTY/cloudflared-release.json.
- 13 automated application tests passed: unauthorized metadata/file access; single-use pairing and cookie settings; exact file round-trip and safe attachment headers; byte ranges; HEAD handling; expiry/revocation; ten-attempt lockout; Host/Origin/path/method controls; modified-file refusal; pinned file when pathname is replaced; symlink/directory refusal; oversized/invalid pairing input; empty files.
- Final packaged service, using a loopback-only listener: exact 2 MiB binary download with SHA-256 comparison and byte-range verification.
- Explicit Stop command: process exited and listener closed immediately in the local test (under 0.01 second after the command, no public tunnel in that test).
- Parent pipe EOF, simulating loss of the owner window: process exited and listener closed immediately in the local test.
- Missing heartbeat: process exited and listener closed after 15.07 seconds.

## Blocked or not yet verified

- The public test successfully created a temporary Cloudflare hostname but could not establish its data connection. Cloudflared reported TLS handshake EOF errors and failed outbound TCP/UDP port 7844 connectivity checks; its HTTPS setup API remained reachable. The exact network component responsible was not identified. The user asked to keep their VPN running, so its settings were not changed and the public test was not retried without it.
- Therefore an actual internet download, Windows recipient behavior, and public active-download cancellation are **not verified** in this delivery.
- Native-window visual inspection and clicking the red close button were not completed because computer-control permissions were pending. The frontend compiled, and its worker shutdown protocol was exercised independently as described above. No claim of a full manual GUI test is made.
- Manual sleep/lid-close behavior and hardware energy measurements are not tested. The source observes macOS sleep notifications and uses a scoped idle-sleep prevention activity.
- No security audit, large-file stress test, relay SLA, or cross-network compatibility guarantee.

## Your acceptance check

1. Open the app, choose a non-sensitive sample and start sharing.
2. If the tunnel connects, open its link on a device using a different internet connection and enter the code.
3. Download the sample; compare its SHA-256 if desired.
4. Start a larger download, then close the sender app. The remaining transfer should fail once already-buffered bytes drain.
5. Check Activity Monitor: WindowDrop and its sharecore/cloudflared processes should be gone.
6. Reopen the app: no share should start automatically, and the old code/link should not become valid again.

All temporary tunnel tests were terminated. Test files, secrets and temporary tunnel configuration are excluded from the distribution.
