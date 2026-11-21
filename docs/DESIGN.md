# Lob — design

_Status: draft v0.1 (2026-11-21). Decisions marked **Decided**; everything under "Open questions" is not._

Lob lets you throw ("lob") a file, URL or piece of text from one of your devices to another device, yours or someone else's in the household. Items wait on the server for up to 24 hours, so the receiving device does not need to be online when you send.

## 1. Goals and non-goals

**Goals (v1)**

- Clients for iOS, Android, macOS, Windows and Linux from one codebase.
- Send files up to 95 MB (see §6.3), URLs and text to a chosen device (the main use case) or to all of a user's devices (§9.3).
- Items queue on the server for 24 hours, then expire.
- A device can be registered with more than one server.
- Each server has several users; each user owns their own devices.
- A user can allow another user to send to some of their devices (e.g. Rodney may send to his wife's iPad, but cannot read its inbox).
- End-to-end encryption: the server, Cloudflare and anything in between see only ciphertext.
- Native "Share to…" integration on each platform.

**Non-goals (v1)**

- Instant push notifications on iOS (the inbox refreshes when the app opens). See §8.
- Syncing folders or keeping files after delivery. Lob is a hand-off, not storage.
- Sharing with people outside a server you control.
- Our own server. Lob uses an unmodified ntfy server.

## 2. Architecture

**Decided:** use an unmodified, self-hosted [ntfy](https://ntfy.sh) server as the transport and queue, and build Lob as a set of clients.

```
┌──────────── Lob client (Flutter) ────────────┐                 ┌──────── ntfy server (Docker) ────────┐
│ Share sheet / tray / drag-drop / app UI      │                 │                                       │
│        │                                     │  HTTPS publish  │  topic  lob_<user>_<device>           │
│ lob_core: envelope + E2E crypto + ntfy client├────────────────▶│  message cache (24h)                  │
│        │                                     │                 │  attachment store (24h, 100M max)     │
│ Keychain/Keystore: device keys, ntfy tokens  │◀────────────────┤  users + ACL (deny-all default)       │
└──────────────────────────────────────────────┘ JSON stream/poll└───────────────────────────────────────┘
```

Why ntfy: it already does the hard server-side parts (auth, per-topic ACL, a time-limited message cache, attachments with size limits and expiry, catch-up via `since=`, delete/clear via sequence IDs), it is Apache-2.0/GPLv2, a single Go binary, and runs well in Docker. Lob only needs a naming convention and an encrypted payload format on top.

What ntfy does not give us, and how Lob fills the gap:

| Gap | Lob's answer |
|---|---|
| No concept of a "device" or its owner | Topic naming convention (§4) and device cards (§5.3) |
| No end-to-end encryption | Client-side encryption envelope (§6) |
| Non-admins cannot grant other users access | v1: the server admin grants it with `ntfy access` (§5.2). Later: an admin screen in Lob |
| iOS instant push needs a vendor push key | v1: poll on open (§8) |

## 3. Server

### 3.1 Deployment

One ntfy container per server. Example files in [`server/`](../server/): `compose.yaml` and `server.yml`.

Key settings (ntfy v2.38.1 docs; defaults in brackets):

| Setting | Lob value | Why |
|---|---|---|
| `cache-file` | `/var/lib/ntfy/cache.db` | Persist the queue across restarts [in-memory] |
| `cache-duration` | `24h` | The 24-hour queue [12h] |
| `attachment-cache-dir` | `/var/lib/ntfy/attachments` | Enables attachments |
| `attachment-file-size-limit` | `100M` | Max item size [15M] |
| `attachment-expiry-duration` | `24h` | Must be ≤ `cache-duration` [3h] |
| `attachment-total-size-limit` | `5G` or to taste | Disk budget for all queued files [5G] |
| `visitor-attachment-total-size-limit` | `2G` | Per-user queued bytes; the 100M default allows only one large file [100M] |
| `visitor-attachment-daily-bandwidth-limit` | `5G` | Uploads + downloads per user per day [500M] |
| `auth-file` | `/var/lib/ntfy/user.db` | Users and ACL |
| `auth-default-access` | `deny-all` | Nothing is open unless granted |
| `behind-proxy` | `true` | Behind Traefik / cloudflared |
| `enable-signup`, `enable-login` | `false`, `true` | Admin creates users |

Per-user limits can also be set with ntfy **tiers** if one user needs more.

SQLite is enough for a household. ntfy also supports PostgreSQL (`database-url`) if the server ever needs HA.

### 3.2 Remote access

Clients are not browsers, so the ntfy hostname must **not** sit behind an SSO login redirect (e.g. an Authentik proxy provider or Cloudflare Access with an interactive login). ntfy's own user/token auth protects it.

| Option | Notes |
|---|---|
| **A. Cloudflare Tunnel → ntfy directly** (**Decided** for away-from-home access) | No open port. Cloudflare's Free/Pro plans cap request bodies at **100 MB**, so Lob caps files at ~95 MB (§6.3). Cloudflare terminates TLS, so it could see payloads: E2E encryption (§6) makes that moot. Long-lived subscribe connections are fine: ntfy sends a keepalive every 45s, under Cloudflare's 100s idle timeout. Cloudflare's terms discourage serving large volumes of non-HTML content through the free CDN; household-scale use is low volume, but keep it in mind. |
| B. Router port 443 → reverse proxy → ntfy | No 100 MB cap, TLS ends on your hardware. Exposes a port (already the case on some setups). |
| C. VPN only (WireGuard/Tailscale) | Simplest security; every device needs the VPN on when away from home. |

On the LAN, split DNS can send the same hostname straight to the reverse proxy, which avoids the Cloudflare cap at home.

The client does not care which option is used: it only needs the server's base URL.

## 4. Topic naming

**Decided:** one ntfy topic per device per server.

```
lob_<user>_<device>        e.g.  lob_rodney_macbook, lob_wife_ipad
```

- `<user>` is the ntfy username; `<device>` is a short slug chosen at registration (`[a-z0-9-]`, max 24 chars). ntfy topic names allow `[-_A-Za-z0-9]{1,64}`.
- The `lob_` prefix keeps Lob topics apart from any other use of the same ntfy server (alerts, etc.).
- Topic names are visible to the server; message content is not (§6).

## 5. Users, devices and permissions

### 5.1 Users

A Lob user is an ntfy user. The server admin creates them:

```sh
ntfy user add rodney
ntfy user add wife
```

### 5.2 Permissions (ACL)

| Intent | ntfy command |
|---|---|
| A user owns all their own Lob topics | `ntfy access rodney 'lob_rodney_*' rw` |
| Rodney may **send to** the wife's iPad (not read it) | `ntfy access rodney lob_wife_ipad wo` |
| Rodney may send to all of the wife's devices | `ntfy access rodney 'lob_wife_*' wo` |
| Revoke | `ntfy access --reset rodney lob_wife_ipad` |

`wo` (write-only) is the key permission: a sender can drop items into another person's inbox but can never list or read it. With encryption, even a misconfigured `rw` would only expose ciphertext.

In v1 the admin runs these commands (`docker exec ntfy ntfy access ...`). ntfy has an internal, undocumented admin API (`/v1/users`, `/v1/users/access`) and an experimental admin Unix socket; a later Lob "server admin" screen could use them, but we should not depend on undocumented APIs in v1.

### 5.3 Device registration

Registering a device with a server:

1. User enters the server URL, username and password.
2. Client creates an ntfy **access token** for this device (labelled with the device name), stores the token in the OS keychain and discards the password. _To verify: whether tokens can be created over the HTTP account API or only in the web app/CLI. Fallback: the user pastes a token created in the ntfy web app._
3. User picks a device name → topic `lob_<user>_<device>`.
4. Client generates the device's key pair (§6.1), stored in the keychain/keystore, never leaves the device.
5. Client subscribes to its topic.

A device can repeat this for any number of servers; each registration is independent (own token, own topic, own key pair).

### 5.4 Device cards (address book and key exchange)

ntfy keeps nothing for more than the cache duration, so there is no server-side directory of devices. Instead each device has a **device card**:

```json
{ "v": 1, "server": "https://lob.example.com", "topic": "lob_wife_ipad",
  "name": "Wife's iPad", "user": "wife", "pk": "<base64 X25519 public key>" }
```

- **Adding a recipient:** the recipient's device shows its card as a QR code (or a copyable `lob:` link); the sender scans it. Scanning in person is also how the public key is trusted (no trust-on-first-use over the network).
- **Your own devices:** when a new device is registered, pair it once with an existing device by QR; the existing device then lobs its whole address book to the new one (encrypted, like any item).
- The client shows a short fingerprint of each key so it can be compared by eye.
- A send only works if the ACL also allows it (§5.2). The client reports "not permitted" (ntfy 403) clearly.

## 6. End-to-end encryption

**Decided:** every item is encrypted on the sending device for one recipient device. The server and Cloudflare store and forward only ciphertext.

### 6.1 Keys

- Each device registration has an X25519 key pair (libsodium `crypto_box`).
- Each item gets a fresh random 32-byte content key `K`.

### 6.2 What is sent

One ntfy message per item per recipient device:

- **Message body** (JSON, under ntfy's 4 KB limit):
  ```json
  { "lob": 1, "from": "<sender card fingerprint>", "spk": "<sender public key>",
    "kn": "<nonce>", "wk": "<K wrapped with crypto_box(sender sk, recipient pk)>",
    "mn": "<nonce>", "m": "<envelope encrypted with secretbox(K)>" }
  ```
- **Envelope** (encrypted, inside `m`):
  ```json
  { "kind": "text|url|file", "text": "...", "url": "...",
    "file": { "name": "photo.jpg", "mime": "image/jpeg", "size": 1234, "sha256": "..." },
    "sent_at": "2026-11-21T10:00:00Z", "from_name": "Rodney's MacBook" }
  ```
- **File** (kind `file`): encrypted with `crypto_secretstream_xchacha20poly1305` under `K` in 64 KB chunks, uploaded as the message's ntfy attachment with a meaningless name (`<random>.lob`).
- Text too long for the 4 KB body is sent like a file.
- ntfy fields that the server can read (`Title`, `Tags`, `Filename`, `Click`) are left empty or generic. The server learns: sender user, recipient topic, time, and the approximate size.

`crypto_box` authenticates the sender to the recipient, so a device can tell which paired card sent an item; items from unknown keys are shown as "unverified sender".

### 6.3 Size limit

Encryption adds ~17 bytes per 64 KB chunk (~27 KB on 100 MB). With Cloudflare Tunnel's 100 MB request cap, Lob limits files to **95 MB (decimal)** to leave room for overhead and headers. If the server is not behind Cloudflare the client may allow up to the server's `attachment-file-size-limit` (configurable per server).

### 6.4 Library

libsodium through the `sodium_libs` Flutter package (all five platforms). _To verify at build time: package maturity and secretstream support on each platform._

## 7. Receiving: the inbox

- On each device, for each server: subscribe to `/<topic>/json?since=<last id>` while the app runs (desktop, Android foreground service, iOS foreground), or poll with `?poll=1&since=<last id>` (iOS on open, or iOS background refresh when the system allows).
- Items are decrypted and shown in an inbox: text (copy), URL (open / copy), file (save / open / share onward). Desktop can auto-save files to a chosen folder.
- After the user acts on or dismisses an item, the client deletes it on the server (`DELETE /<topic>/<sequence id>`, ntfy ≥ 2.16) so it does not come back on reinstall and frees disk early. Otherwise ntfy expires it after 24h.
- Expiry is shown on each item ("expires in 5h").
- Multiple servers are merged into one inbox, labelled by server.

## 8. Notifications per platform

| Platform | v1 behaviour |
|---|---|
| Android | Foreground service keeps the subscription open; local notification on arrival. |
| macOS, Windows, Linux | Tray/menu-bar app keeps the subscription open while running (start at login); local notification on arrival. |
| iOS / iPadOS | Inbox refreshes when the app opens (plus opportunistic background refresh). No instant push. |

Later options for iOS push: our own Firebase project with an Apple push key wired into ntfy's `firebase-key-file`, or a small relay that watches the topics and sends APNs pushes. Both need the Apple Developer account that iOS distribution needs anyway.

## 9. Client

**Decided:** Flutter, one codebase.

### 9.1 Structure

```
app/                Flutter app (UI, platform integration)
packages/lob_core/  pure Dart: ntfy client, device cards, envelope + crypto, storage
cli/                optional Dart CLI ("lob send ...", "lob inbox"), handy on Linux and for testing
```

### 9.2 Platform integration

| Platform | Share in | Background / tray | Native code |
|---|---|---|---|
| iOS | Share Extension | — | Swift share extension target |
| Android | `ACTION_SEND` / `ACTION_SEND_MULTIPLE` intent filters | Foreground service | Manifest + small Kotlin |
| macOS | Share Extension; drag onto menu-bar icon | Menu-bar app, launch at login | Swift share extension target |
| Windows | Explorer "Send to" shortcut and/or context-menu entry from the installer; drag onto window | Tray app, start at login | Installer config |
| Linux | `.desktop` entry with `MimeType`; file-manager actions (Nautilus/Dolphin); drag onto window | Tray app (where the desktop supports it) | `.desktop` files |

Desktop share entries launch the app with file paths as arguments; a single-instance guard forwards them to the running instance.

Candidate packages (verify when starting): `receive_sharing_intent` or `share_handler`, `tray_manager`, `window_manager`, `flutter_local_notifications`, `desktop_drop`, `launch_at_startup`, `flutter_secure_storage`, `mobile_scanner` + `qr_flutter`, `dio` (streamed uploads with progress), `sodium_libs`.

### 9.3 Sending flow

1. Pick content (share sheet, drag-drop, paste, or the app's "Lob" button).
2. Pick recipient device(s) from the address book (grouped by server, then user). Picking a **user** sends to every device card of that user in the address book (client-side fan-out: one encrypted publish per device). Sending to a single device is the default and main path.
3. Encrypt and publish once per recipient device, with progress; failures (403, too large, offline) are shown per recipient.
4. Unsent items are retried while the sender's own app is running (v1 keeps no long-lived sender-side queue).

## 10. Milestones

| # | Milestone | Done when |
|---|---|---|
| M0 | Server | ntfy running in Docker with the §3.1 settings, two users, ACL from §5.2, reachable remotely; checked with `curl` (publish a 95 MB attachment, read it back, confirm 24h expiry, confirm a `wo` user gets 403 on read). |
| M1 | `lob_core` + CLI | Register, device card, encrypt/send text, URL, file; receive and decrypt; unit tests for the envelope. |
| M2 | App on macOS + Android | Registration, address book via QR, send, inbox, notifications, share integration. |
| M3 | Windows + Linux | Tray app, "Send to"/file-manager entries, packaging. |
| M4 | iOS | Share extension, poll-on-open inbox, TestFlight (needs Apple Developer account). |
| M5 | Later | Admin screen for ACLs, iOS push, upload-once for multiple recipients. |

## 11. Open questions

1. **Hosting placement:** which homelab guest runs ntfy (new LXC vs an existing Docker host), and the public hostname.
2. ~~Remote access~~ **Decided:** Cloudflare Tunnel (A); the 95 MB cap is acceptable (lower is fine too).
3. ~~Recipient model~~ **Decided:** send to a device is the main path; send to a user is client-side fan-out over that user's device cards (§9.3).
4. **Token creation:** can the client create an ntfy token over the HTTP API (§5.3), or must users paste one?
5. **Multiple recipients:** upload once and send a pointer to each recipient (needs checking how ntfy authorises attachment downloads), or upload once per recipient (simple, used in v1)?
6. **License** for the Lob repo (the repo is public: `github.com/rodneyshupe/lob`).
7. **Store name:** "Lob" alone clashes with Lob.com (a direct-mail API company); the store listing may need a longer name.
