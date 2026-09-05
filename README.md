# WindowDrop — send a file, close to stop

WindowDrop is a small Mac app that gives your friend a temporary download link and access code. Your friend can be in another house or country and needs only a web browser.

## Start here

1. Extract the ZIP and open **WindowDrop.app**. This build is for **Apple Silicon Macs** (M1 or newer), macOS 15 or later; it was built and tested on the current Mac. No Python installation, Terminal command, account, or recipient app is required.
2. Click **Choose file…** and select a regular file. Zip a folder first.
3. Choose an expiry of 15 minutes, 30 minutes, 1 hour, or 2 hours.
4. Click **Start sharing**. Allow up to a minute to connect.
5. Click **Copy link**, then **Copy code**, and give them to your friend. For sensitive files, communicate the code separately from the link.
6. Your friend opens the link in a browser, enters the code, and clicks **Download file**. Their browser shows download progress. Windows Edge, Chrome, Firefox, and Safari use an ordinary file download.
7. Click **Stop sharing**, close the red window button, or press **Command-Q** to end sharing. Closing the window quits the app.

Opening the app does not start a server or tunnel. Selecting a file does not send it anywhere. Only Start sharing creates the connection. The filename is shown to the recipient only after the correct code is entered.

The app is locally signed for this build, not notarized by Apple. If macOS blocks this personally built copy, use the per-app **Open Anyway** option in System Settings → Privacy & Security after reviewing the included source. Do not disable Gatekeeper globally. This is a preview, not an App Store release.

## Exactly when it runs

| State | Behavior |
|---|---|
| Window open, not sharing | Native window only; no file listener or tunnel |
| Sharing | One local file service and one outbound tunnel; only your selected file is available |
| Minimized or another app in front | Sharing continues; the window is still open |
| Stop sharing | Server and tunnel exit; window remains ready for a new share |
| Red close button / Command-Q | Server and tunnel exit, then the entire app quits |
| Native window process crashes / Force Quit | Its control pipe closes; the worker stops the tunnel and exits |
| Window becomes unresponsive | A missing-heartbeat timeout stops sharing after about 15 seconds |
| Automatic link expiry | Server and tunnel exit; window remains open |
| Manual sleep / lid close | A macOS sleep notification stops sharing; wake does not restart it automatically |
| Shutdown | Processes stop; the previous share is not restored |

The app prevents **automatic idle system sleep only while sharing**. It does not prevent manual sleep or promise closed-lid operation. It releases the sleep-prevention activity when sharing ends. There is no login item, installed daemon, menu-bar service, scheduled task, background updater, or automatic restart.

“Close to stop” refers to **your sender app window**. Closing your friend’s browser tab does not quit your app. Bytes already received or buffered cannot be recalled; files already downloaded remain on the recipient’s machine.

A browser tab was deliberately not used as the sender interface: browser close/unload callbacks are unreliable, and background-tab throttling can confuse heartbeat-based lifetimes.

## What the link means

WindowDrop uses an outbound Cloudflare Tunnel to get a temporary `https://…trycloudflare.com/s/…` URL. No incoming router port, public home IP, or same-Wi-Fi connection is required.

```text
Your friend’s browser
        │ HTTPS
        ▼
Cloudflare’s temporary public address
        │ encrypted tunnel
        ▼
Your Mac: cloudflared → loopback file service → selected file
```

The service listens only on `127.0.0.1` with a randomly selected port. Cloudflared is started only for this share, is not installed as a system service, and has automatic updates disabled. Its configuration is isolated in a temporary folder; your existing Cloudflare configuration is not changed.

**This is not end-to-end encryption against Cloudflare.** HTTPS terminates at Cloudflare, so Cloudflare is part of the trust boundary and can process file content. WindowDrop does not upload a stored cloud copy and requests no caching; that is not a claim of zero intermediary buffering or provider logging. The link and access code protect recipient access, not against the relay operator. Use a trusted end-to-end transfer system for files requiring that stronger threat model.

The bundled default uses Cloudflare **Quick Tunnels**, which Cloudflare describes as a testing/development service without guaranteed uptime. This preview is suitable for trying the workflow. A supported long-term edition should use a managed tunnel with your domain/account or a dedicated relay, preserving the same close-to-stop behavior. Official documentation: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/trycloudflare/ .

## Access and file handling

- Each share gets a cryptographically random 256-bit link identifier and a separate random 12-character hexadecimal code (48 bits), displayed in groups.
- The code can unlock **one browser session**. It is consumed on success. That browser may retry or download again while the share is active; “single use” refers to pairing, not one HTTP request.
- Ten wrong code attempts lock pairing for that share. Stop and start to generate a new code and link.
- The recipient receives a Secure, HttpOnly, SameSite Strict cookie scoped to the random share path. The code is not embedded in the URL.
- The receiver cannot browse your folders, upload files, or access another file. No local administration endpoint is exposed.
- Responses instruct the browser and CDN not to cache. Downloads use an attachment disposition, and the page uses a restrictive Content Security Policy with script/style hashes.
- The selected file is held open and read in chunks. Replacing its pathname does not silently share the replacement. Modifying the open file causes subsequent requests/chunks to be refused: do not edit the file during a transfer.
- Download memory use is bounded per chunk; the app does not copy the whole file into memory or create a cloud archive.
- Byte-range requests are supported for browser retries while the same share and cookie remain valid. Resume after stopping or restarting a share is not supported.
- At most two file streams run at once. There is no explicit file-size cap, but tunnel behavior, network quality, your upload speed, browser behavior and the chosen expiry constrain practical transfers. Large-file internet reliability must be tested on your connection.
- The sender’s progress measures bytes yielded to the connection, not proof that the friend saved them to disk. The recipient’s browser confirms completion.

Links/codes exist in memory. There is no share-history database or app access log. The OS, browser and relay can retain their own records. Clipboard cleanup is best effort: a copied current link or unused code is cleared on Stop if the clipboard still contains that value; otherwise the app leaves your clipboard alone.

## Troubleshooting

**Creating your link fails or times out:** Check internet access. A VPN, Clash TUN route, firewall, security software, or network policy can block Cloudflare Tunnel. The helper needs outbound TCP port 7844 for its HTTP/2 connection and HTTPS for setup. Do not open inbound router ports. Compare with the VPN paused if appropriate, or try another network. Provider availability is outside the app’s control.

**Link not reachable immediately:** Give the new hostname a few seconds to propagate, then reload. Ensure the sender window still shows that sharing is live. A stopped link may show a Cloudflare error instead of a custom “expired” page because the host itself is offline.

**Used code:** Reuse the same browser/profile with its cookie, or have the sender Stop and Start for a new code. Do not consume the code by testing in the sender’s browser before your friend uses it.

**Download fails midway:** Keep the Mac awake and connected, avoid modifying the selected file, and choose a longer expiry for large files. Check the recipient’s free disk space. Refresh/retry while the share is active.

**No background energy use after close:** Check Activity Monitor for WindowDrop, sharecore, and the app’s cloudflared process. They should be gone shortly after closing (normally within a couple seconds, with a 3-second normal-close failsafe). Other cloudflared instances you independently run are not owned by this app.

## Files in this package

- `WindowDrop.app` — self-contained Apple Silicon sender app.
- `Source/WindowDrop.swift` — native AppKit window and process lifetime control.
- `Source/sharecore.py` — one-file HTTP application and tunnel supervisor.
- `Source/receiver.html` — browser receiver; no third-party scripts or assets.
- `Source/test_sharecore.py` — access, file-handling and range tests.
- `Source/build-mac.sh` — instructions/script to reproduce the native build.
- `TEST-RESULTS.md` — actual validation and remaining limitations.
- `THIRD-PARTY/` — dependency versions, licenses and tunnel release checksum.

This is an internet-facing preview, not a security-audited product. Test with a non-sensitive sample first. A malicious holder of the full link can consume the attempt budget, and basic resource limits do not provide comprehensive denial-of-service protection.

## Rebuild and run checks

The delivered app is already built. Rebuilding needs Xcode Command Line Tools and Python 3.12 on an Apple Silicon Mac. From `Source`, run `bash build-mac.sh`; its output is `.build/WindowDrop.app`. It reuses the official tunnel helper already in the delivered app. The build script uses a build-local workaround for duplicate module maps found in this Mac’s Command Line Tools; it does not edit system files.

For source checks after creating a Python environment and installing `Source/requirements.txt`:

```sh
python -m unittest discover -s Source -p 'test_sharecore.py' -v
python Source/test_lifecycle.py WindowDrop.app/Contents/Resources/Core/sharecore
```

The lifecycle check opens only loopback listeners and automatically closes them. It does not create an internet tunnel.
