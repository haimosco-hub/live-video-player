# How it works

A static page (GitHub Pages) with an HLS stream library and a custom player. The stream list is **encrypted** and only decrypted in the browser after the correct password is entered, so this repository contains no readable stream data.

This document describes the design. The build inputs (source page, data export, build scripts) are kept privately and are not part of this repository.

## Flow

1. **Data export (private):** a list of games and their streams is exported with these fields per stream: game name, stream id, date, system (competition), stream language, home/visitor team logo URL, plus one logo per system.
2. **Build (private):** a script takes the plaintext page and the export, encrypts the data as a single JSON payload, removes it from the page, adds a lock screen, and writes `index.html`.
3. **Publish:** only the generated `index.html` is committed here; GitHub Pages serves it.
4. **Browser:** the visitor enters the password; the page derives a key, decrypts the payload and renders the library.

A build-time check fails the build if any stream id or internal address is still present in plaintext in the output.

## Encryption

- Cipher: **AES-256-GCM** (authenticated, so a wrong password fails cleanly).
- Key derivation: **PBKDF2-HMAC-SHA256**, 600,000 iterations, 256-bit key, using the browser's WebCrypto API.
- Layout: `ciphertext || 16-byte tag`, base64, embedded in the page as `ENC = {v, iter, salt, iv, ct}`.
- Salt is fixed (it is not secret) so remembered devices keep working across data refreshes; the **IV is random and new on every build**, which is required for GCM safety.
- Encrypted payload: the stream list, the stream base address and the system logo map.
- The passphrase is never stored in the repository or in any file; it is supplied only at build time.

### Honest limits

The attempt limit is enforced in the visitor's own browser, so someone who downloads the page can try passwords offline. The slow key derivation helps, but the protection ultimately depends on a long, non-guessable passphrase. Changing the passphrase invalidates every remembered device.

## Lock screen

- Password field (plus a hidden username so password managers can save it).
- **5 wrong attempts lock the form for 60 seconds** (state kept in the browser, survives reload).
- **"Remember this device (30 days)"** (on by default) keeps the derived key, not the password, in the browser's local storage and unlocks automatically next time. It is dropped after 30 days, if the passphrase changes, or if browser data is cleared.
- Decryption needs a secure context (https or `localhost`), not `file://`.

## Player

- [hls.js](https://github.com/video-dev/hls.js) with a native HLS fallback (Safari); no autoplay.
- Custom controls: play/pause, -5 s, +15 s, volume, time, live tag, seek bar, **High/Low** quality switch, **language** menu, fullscreen.
- Title and description overlay at the top-left, visible only while the controls are visible.
- **Keeps your position** when switching quality or language: it resumes at the same offset from the start, or at the last available point if the other stream is shorter.

## Library

- One card per system + game + date; a game's streams are its language options. Default language preference: `national`, `english`, `en-en`, `en`, then the first available.
- Filters: system multi-select (with search for long lists) and a game-name search, with Apply and Reset.

## Refreshing the data

Done privately: re-export the data, rebuild with the passphrase, verify the build (decrypt check and plaintext leak check), commit the new `index.html` and push. Devices that remembered the password stay unlocked.

## Notes

- The streams are served from a private network address, so they only play for viewers who have access to it.
- Samsung/Tizen TVs have no Google Cast; use AirPlay/Smart View or the TV's own browser.
