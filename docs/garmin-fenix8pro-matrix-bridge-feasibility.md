# Garmin Fenix 8 Pro → Matrix: Messaging Bridge Feasibility

*A technical feasibility assessment of two approaches for routing messages from a Garmin Fenix 8 Pro into a self-hosted Matrix homeserver, with the watch as the input device.*

---

## Executive summary — bottom line up front

The goal is to keep the Fenix 8 Pro as the *input* device but deliver messages into a self-hosted **Matrix** homeserver over channels recipients already use, instead of the Garmin web-link experience nobody wants. The phone is often **not present** — that is the entire point of the watch's independent LTE/satellite comms — so any solution that only works when a phone is tethered fails the brief.

Two approaches were evaluated:

- **Path 1 — reverse-engineer the Garmin Messenger backend and forward to Matrix.** **Not viable** for the core use case. When the phone is absent the watch talks straight to Garmin's cloud over its own LTE/satellite radio; there is no phone-side traffic to intercept, nobody has reverse-engineered the *consumer* Messenger backend, the authentication is device-bound and actively hardened, and interception breaches Garmin's terms of service.

- **Path 2 — build a native Connect IQ (Monkey C) app that talks to Matrix directly.** **This is the recommended direction.** Contrary to the widely repeated lore that Connect IQ networking always needs a phone, the Fenix 8 Pro exposes its own LTE to third-party `makeWebRequest` calls when the phone is out of range. This is not theoretical: the popular **GarminHomeAssistant** Connect IQ app ships an explicit Wi-Fi/LTE mode that runs phone-free, and it is already in use on a Fenix 8 Pro talking to a self-hosted server. A Matrix client is architecturally the same shape — a watch app making HTTPS/JSON calls directly to a self-hosted server — so the same transport works.

The honest constraints on Path 2: there is no real-time push (you poll), the homeserver needs a publicly trusted TLS certificate, large Matrix `/sync` responses exceed the watch's memory budget and want a thin server-side proxy or a tight sync filter, LTE data rides Garmin's carrier partners, and **satellite-only operation (no LTE, no phone) is closed to third-party apps**. None of these is a showstopper for a send-first, poll-to-receive messaging app.

**Recommendation:** build a native Connect IQ Matrix client scoped around one-shot outbound requests (send a message; short-poll to receive), backed by a small server-side proxy next to the homeserver. It is the only approach that keeps the watch as the input device, delivers into self-hosted Matrix, and works with the phone left at home.

> **A correction, up front and on the record.** An initial read of the evidence — the Forerunner 945 LTE precedent (where Connect IQ had no LTE access) plus Garmin's own WhatsApp app hanging without a phone — pointed toward "Connect IQ on the Fenix 8 Pro is phone-tethered, so Path 2 is a dead end." That conclusion was **wrong**, and it was corrected by first-party app documentation and the device owner's firsthand use. The reconciliation is explained in the Path 2 section; it is left visible here deliberately, because the brief asked for intellectual honesty over a tidy narrative.

---

## Path 1 — Reverse-engineer Garmin Messenger, forward to Matrix

**The idea:** intercept the traffic between the watch (and/or the Garmin Messenger phone app) and Garmin's backend, extract the message payload and metadata, and forward it into Matrix via the client-server API. Architecturally a bridge: watch → Garmin servers → interception point → Matrix.

### Protocol and where interception would have to happen

The *consumer Garmin Messenger* app-to-cloud protocol is **not publicly documented anywhere**. Its sibling system, Garmin Connect (health/fitness), is well established as REST/JSON over HTTPS with OAuth — that is what the unofficial client libraries target — but no public capture, specification, or repository describes the Messenger message-send endpoint. The satellite leg (device ↔ Iridium/Skylo) is a proprietary, heavily size-constrained binary protocol, and the watch ↔ phone BLE leg is Garmin's proprietary GFDI/Multi-Link framing documented by the [Gadgetbridge](https://gadgetbridge.org/internals/specifics/garmin-protocol/) project. Any claim about the specific Messenger cloud protocol (plain JSON, protobuf-over-HTTPS, a push socket) is speculation; there is no evidence to confirm it.

### The phone-absent problem — the fatal one

This is the point on which Path 1 fails. The Fenix 8 Pro is a standalone device: with an inReach subscription it has its own built-in LTE eSIM (on Garmin's carrier partners, not your own SIM) and a two-way satellite modem via [Skylo's GEO satellites](https://www.dcrainmaker.com/2025/09/garmin-fenix-8-pro-lte-microled-satellite-messaging-hands-on-everything-you-need-to-know.html). Connectivity priority is phone → the watch's own LTE → satellite. Per Garmin's [feature-requirements page](https://www.garmin.com/en-US/connectivity/fenix8pro/feature-requirements/), when the phone is out of range the watch uses its own LTE, then satellite, and talks to the Garmin Messenger backend directly.

The consequence for interception:

- **Phone present, watch relaying via BLE:** the message goes watch → (BLE/GFDI) → phone → (HTTPS) → Garmin. A phone-side man-in-the-middle *could* observe the cloud leg — but this is exactly the mode the brief does *not* care about, and even here the BLE messaging leg has never been reverse-engineered (Gadgetbridge implements pairing, battery, activity sync and weather for inReach devices, but [explicitly cannot send inReach messages](https://codeberg.org/Freeyourgadget/Gadgetbridge/wiki/Garmin-devices)).
- **Phone absent (the core case):** the watch talks to Garmin over its own LTE eSIM or satellite. **There is zero phone traffic.** A phone-side Frida/MITM setup sees nothing. Intercepting here means attacking the watch's own encrypted cellular/satellite radio traffic on locked embedded firmware — not a realistic project.

So the entire phone-side interception approach is architecturally mismatched to the requirement.

### Prior art

There is none for consumer messaging. The community reverse-engineering effort is entirely health/activity data or BLE:

- [cyberjunky/python-garminconnect](https://github.com/cyberjunky/python-garminconnect) — 130+ Garmin Connect REST endpoints for stats, activities, sleep, heart rate, body composition. **No messaging.**
- [matin/garth](https://github.com/matin/garth) — the auth library the ecosystem depended on, now **deprecated**: its README states Garmin changed the auth flow and broke the mobile-auth approach garth relied on. Health data only.
- [Gadgetbridge](https://codeberg.org/Freeyourgadget/Gadgetbridge/wiki/Garmin-devices) — the deepest RE work (BLE/GFDI), but inReach *messaging* is explicitly unimplemented.
- [j-arens/garmin-ipc](https://github.com/j-arens/garmin-ipc) — inReach-to-inReach forwarding built on the official *enterprise* IPC API, i.e. the excluded ecosystem.

A Matrix ↔ Garmin bridge does not exist in the mautrix/matrix bridge ecosystem or on GitHub. This would be entirely greenfield, on an undocumented, actively-hardened target.

### Authentication / session model

Garmin Connect authenticates via a mobile SSO flow (`sso.garmin.com` → service ticket → DI OAuth bearer tokens at `diauth.garmin.com`; refresh tokens last roughly a year). This flow recently changed and broke garth — Garmin actively hardens it. Crucially, the Fenix 8 Pro's messaging is bound to an inReach subscription tied to the device's identity and eSIM, not just a username and password. Watch-originated satellite/LTE messages are bound to device-provisioned credentials on the carrier/satellite network and **cannot be replayed from a server**. A server-side relay of *account-level* actions is theoretically conceivable the way python-garminconnect relays health data — but only if the consumer Messenger endpoints were known (they are not) and the auth flow were stable (it is not).

### Legal / ToS exposure

Garmin's SDK/API terms and terms of use [explicitly prohibit](https://www.garmin.com/en-US/legal/terms-of-use/) reverse engineering, decompiling, and use of the service for "interception" equipment. The inReach Portal Connect API terms likewise bar reverse engineering. Realistic exposure for personal use on your own account is account/subscription **termination** (unofficial clients already get rate-limited and blocked). MITM of a live cellular/satellite messaging service and defeating certificate pinning could additionally implicate anti-circumvention statutes (e.g. DMCA §1201 in the US) and computer-misuse law depending on jurisdiction — this is a flagged risk, not legal advice, and it rises sharply for anything redistributed or public-facing.

### Effort and maintenance

High effort, hostile maintenance profile. You would be first to reverse-engineer an undocumented consumer protocol, likely behind certificate pinning (untested but likely, given Garmin's posture), on a target whose auth ecosystem is currently broken — and even a complete success only covers the phone-present case, which is not the requirement. Every Garmin app or backend update is a potential break. **Not recommended.**

*The one documented server-to-server path for inReach messages, noted only as contrast, is the enterprise [inReach Portal Connect (IPC) Outbound webhook](https://developer.garmin.com/inreach-portal/). It is a separate, approval-gated ecosystem (reportedly not accepting new applicants, and it auto-suspends after five days with no successful deliveries). It is explicitly outside the scope of this consumer Fenix 8 Pro investigation and is not a solution here — but if IPC access were ever obtainable, it would be the only sanctioned server-side bridge.*

---

## Path 2 — Native Connect IQ (Monkey C) app talking to Matrix directly

**The idea:** write a Connect IQ application for the Fenix 8 Pro that speaks the Matrix client-server API directly over HTTPS, bypassing Garmin Messenger entirely. Architecturally a direct client: watch → Matrix homeserver.

### Can Connect IQ make arbitrary HTTPS requests to a self-hosted server? Yes.

`Toybox.Communications.makeWebRequest` supports arbitrary URLs with standard HTTP methods, JSON request bodies (`REQUEST_CONTENT_TYPE_JSON`), and JSON responses auto-parsed into a Monkey C dictionary ([API docs](https://developer.garmin.com/connect-iq/api-docs/Toybox/Communications.html)). This is proven in the wild by self-hosted apps: [GarminHomeAssistant](https://github.com/house-of-abbey/GarminHomeAssistant), the [wrist-ntfy](https://github.com/kgs0142/wrist-ntfy) client, and [geogram](https://github.com/geograms/geogram) (which calls chat-room endpoints via `makeWebRequest`).

Two hard constraints matter:

- **Public CA certificate required.** The homeserver must present a full certificate chain from a CA in Garmin's trust store — a self-signed certificate will not work. Let's Encrypt works, with occasional issues on older devices. GarminHomeAssistant's [troubleshooting guide](https://github.com/house-of-abbey/GarminHomeAssistant/blob/main/TroubleShooting.md) documents exactly this hurdle.
- **Response-size limits.** Responses that exceed the app's memory sandbox fail with `-402 NETWORK_RESPONSE_TOO_LARGE` (developers report failures in the ~8–44 kB range, device-dependent) or `-403` when free memory is low. A raw Matrix `/sync` response is far too big — so a **thin server-side proxy**, or an aggressive server-side `filter`/`limit`, is needed to keep receive payloads small.

### The decisive question — phone-free networking over LTE — resolved: yes, on the Fenix 8 Pro

The widely repeated lore is that Connect IQ networking is always mediated by the phone (Garmin Connect Mobile over BLE): a `makeWebRequest` with no phone connected returns `-104`, and on most watches Wi-Fi only powers up for Garmin's own bulk sync, not for third-party requests. On the Forerunner 945 LTE, LTE was reserved for Garmin's built-in functions and unavailable to Connect IQ apps entirely. Taken alone, that lore says Path 2 is phone-tethered and therefore pointless.

**That is wrong for the Fenix 8 Pro,** and three lines of evidence establish it:

1. **Garmin's own fēnix 8 owner's manual** describes the transport: "when your phone is within Bluetooth range, your watch uses your phone's LTE connection. When your phone is not within range, your watch connects to the LTE network." The watch brings up its own LTE when the phone is absent ([manual](https://www8.garmin.com/manuals/webhelp/GUID-EECCAC99-90D6-4AB1-9A3A-EC433D3365E2/EN-US/GUID-10A12B55-5C24-4412-9998-B6B620084AAA.html)).

2. **GarminHomeAssistant ships an explicit Wi-Fi/LTE mode** that operates phone-free, and it is in active use on a Fenix 8 Pro talking to a self-hosted Home Assistant server. Its README states the configuration must be reachable "from your phone… or your watch if it has direct Internet access." This is first-party confirmation that third-party `makeWebRequest` calls ride the watch's own connectivity with no phone in the loop.

3. **The apparent counter-evidence dissolves on inspection.** Garmin's own WhatsApp Connect IQ app hangs without the phone — but WhatsApp needs the *WhatsApp phone app* as an application-layer relay (the account and encryption keys live on the phone). That is a design dependency, not a transport limitation. HomeAssistant — and a Matrix client — talk **directly to a self-hosted HTTP server** with no phone-side companion app. **Matrix is the HomeAssistant case (direct-to-server), not the WhatsApp case (phone-relay).** The WhatsApp app tells us nothing about whether the transport is available; it tells us WhatsApp's design needs the phone.

So the make-or-break question — can a Fenix 8 Pro Connect IQ app reach a self-hosted server over its own LTE with no phone present — is answered **yes**, empirically, by an app already doing it.

### The honest caveats (real, but favourable for messaging)

GarminHomeAssistant warns its phone-free mode is "best used with tap items only," because without a BLE connection the app cannot initialise its menu *state* to match the server. In other words: **one-shot outbound requests are reliable; stateful reconciliation is flaky.** For a messaging app this cuts the right way:

- **Sending** a Matrix message is a single one-shot outbound `PUT` — exactly the "tap item" pattern already confirmed to work phone-free.
- **Receiving** via short-poll `/sync` is also a one-shot outbound `GET`. It works, subject to the response-size limit above and to background-execution limits.

Other honest constraints: there is **no real-time push** — Connect IQ has no websockets, so you poll (continuously in the foreground; background "temporal events" fire at most every five minutes and run for up to 30 seconds with a tight memory budget). LTE data rides Garmin's carrier partners (cost and coverage apply). And on non-LTE watches Wi-Fi is only powered up for Garmin's own sync, so this phone-free behaviour is specific to the Fenix 8 Pro's dedicated LTE — it would not generalise to, say, a plain Fenix 7.

### The one thing a Connect IQ app cannot do: satellite

Satellite messaging is exclusively Garmin's inReach stack; no Toybox API touches the satellite modem. So the single sub-case Path 2 cannot serve is **phone-absent and LTE-absent (satellite-only)** — deep-backcountry, no cellular. In that regime you are back to Garmin Messenger's native satellite path and there is no third-party app route. Worth stating plainly, because it is the one place the watch's independent comms genuinely can't be redirected.

### Input: keyboard yes, voice no

`Toybox.WatchUi.TextPicker` (with `TextPickerDelegate`) is the official text-entry API, and the Fenix 8 gained a native on-screen QWERTY keyboard with prediction in firmware 13.13. Garmin's own WhatsApp app demonstrates free-text composition with the on-watch keyboard on this hardware, so third-party text entry for chat is viable. **Voice/dictation is not available to Connect IQ apps** — there is no microphone API in Toybox (it is an open developer feature request). So the address book plus on-watch keyboard is the input model; voice is out.

### Local storage for an address book: adequate

`Toybox.Application.Storage` provides a key/value store (per-value limits on the order of tens of kB, total store on the order of ~128 kB, device-dependent). Hundreds of Matrix user IDs, display names, and an access token fit trivially. `Properties` handles small settings. No concern here.

### Matrix side — not the bottleneck

A constrained, occasionally-connected client maps cleanly onto the Matrix client-server API:

- **Login:** `POST /_matrix/client/v3/login` returns an `access_token`, `user_id`, and `device_id`. Supplying a fixed `device_id` for the watch keeps token rotation clean.
- **Send:** `PUT /_matrix/client/v3/rooms/{roomId}/send/m.room.message/{txnId}` with `{"msgtype":"m.text","body":"…"}`. The client-generated `txnId` gives idempotency — a retried send after flaky connectivity is de-duplicated, which matters on a watch.
- **Receive:** `GET /_matrix/client/v3/sync`. Long-polling is **optional** — it is controlled entirely by the `timeout` parameter, and `timeout=0` makes `/sync` return immediately (plain short polling). Persist the `next_batch` token and pass it as `since` to get only deltas. A restrictive server-side `filter` (one room, small `timeline.limit`, no presence/typing) keeps each response small enough for the watch — and this is precisely where the thin proxy earns its place.

For credentials, log the watch in as its own `device_id`, ideally a dedicated low-privilege Matrix account that is only a member of the target room, so the token stored in on-watch storage (which is not hardened) has minimal blast radius and can be revoked independently.

*(A server-side Matrix **appservice** bridge — matrix-appservice-bridge or mautrix — is the clean pattern if messages ever arrive at a server. For a consumer Fenix 8 Pro there is no channel that delivers the watch's messages to a server, so the direct on-watch client is the right model; the appservice is only relevant to the excluded IPC path.)*

### Submission and effort

Connect IQ store review is light-touch and largely automated, with no rule against messaging apps (third-party WhatsApp widgets shipped years before Garmin's official one), and you can sideload freely for personal use without the store at all. Monkey C itself is easy for an experienced developer to pick up in days — Java/JavaScript-ish, garbage-collected, optional typing. The real cost is the platform, not the language: per-device memory budgets, simulator-versus-device behavioural gaps, sparse docs, and tooling quirks. SDK churn is moderate (System 7 → 8 → 9 across roughly three years) but additive rather than breaking — old API levels keep working. **Overall effort: moderate, and the maintenance profile is far kinder than Path 1's** — you depend on a stable, documented SDK and a Matrix API you control, not on an undocumented backend that breaks on every vendor update.

---

## Comparative analysis

| Dimension | Path 1 — RE the Garmin API | Path 2 — Connect IQ Matrix client |
|---|---|---|
| Works phone-absent (core requirement) | **No** — nothing to intercept when the watch talks to Garmin directly | **Yes over LTE** — confirmed via GarminHomeAssistant's phone-free mode |
| Works satellite-only | No | No (satellite closed to third-party apps) |
| Prior art | None for consumer messaging; auth ecosystem broken | Direct-to-self-hosted-server apps exist and work today |
| Send | Would require decoding an undocumented protocol | One-shot `PUT` — the confirmed tap-item pattern |
| Receive | Same | Short-poll `/sync` (needs thin proxy / tight filter) |
| Real-time push | N/A | No — polling only; 5-min background floor |
| Legal / ToS | Prohibited; account-termination + anti-circumvention risk | Within Connect IQ terms; store-legitimate or sideload |
| Effort | High, greenfield, hostile | Moderate |
| Maintenance | Breaks on every Garmin update | Stable SDK + a Matrix API you control |
| Verdict | **Not viable** | **Recommended** |

The decisive line is the first one. Path 1 cannot satisfy the phone-not-present requirement even in principle, because the interception point it depends on does not exist when the phone is away. Path 2 does satisfy it — over LTE — and the evidence is not a spec-reading exercise but an app already doing the exact transport a Matrix client needs. The only requirement neither path meets is satellite-only messaging, which is closed to all third-party software; that is a genuine limit of the platform, not of either design.

On long-term maintainability, the two paths are not close. Path 1 sits on an undocumented backend with a hostile vendor and an auth flow that already broke the ecosystem's main library — it would demand continuous reverse-engineering just to stay working. Path 2 sits on a documented, stable SDK and a Matrix API you fully control; the main external risk is Garmin changing the LTE/`makeWebRequest` behaviour, which would equally break GarminHomeAssistant and every similar app, making it a well-watched shared dependency rather than a private cliff.

---

## Recommendation

**Build a native Connect IQ Matrix client for the Fenix 8 Pro.** It is the only approach that keeps the watch as the input device, delivers directly into self-hosted Matrix, and works with the phone left at home. Do not pursue Path 1: it fails the core use case, is terms-of-service-hostile, targets an undocumented backend on a broken-auth ecosystem, and carries an unmaintainable break-on-every-update profile.

Scope the build honestly around what the platform actually supports:

- **Design for one-shot requests.** Sending is a single `PUT`; receiving is short-poll `/sync`. Keep the UI action-oriented — compose/send, and a manual or timed refresh to pull new messages — rather than trying to hold live, perfectly-synchronised state, which is the part that is flaky without BLE.
- **Stand up a thin server-side proxy** next to the homeserver that collapses `/sync` into small, watch-sized JSON responses (latest N messages for one room), sidestepping the `-402` response-size limit. This is a couple of hundred lines, not a bridge.
- **Use a publicly trusted TLS certificate** (Let's Encrypt) on the homeserver or proxy — self-signed will not work.
- **Give the watch its own low-privilege Matrix account** (dedicated `device_id`, member of only the target room) so the token in on-watch storage has minimal blast radius and is independently revocable.
- **Accept the boundaries:** no real-time push (you poll), and no satellite-only support (deep-backcountry with no LTE falls back to Garmin Messenger's native path). Input is on-watch keyboard plus a local address book; voice is not available to third-party apps.

### Lower-effort partial wins worth naming

- **Recipients install Garmin Messenger.** Zero build. It removes the web-link friction immediately — it just isn't Matrix, and it doesn't serve the self-hosting preference. Worth doing as a stopgap regardless.
- **Enterprise IPC bridge**, *only* if inReach Portal Connect access is ever obtainable (currently uncertain and outside the consumer ecosystem). That would enable a clean server-side Matrix bridge via the official Outbound webhook — the one sanctioned server-to-server route — but it is a different device ecosystem and gated behind Garmin's approval.

### Cheapest possible validation

The make-or-break question is already answered empirically: GarminHomeAssistant's phone-free Wi-Fi/LTE mode runs on the Fenix 8 Pro today. The next confirmation step before committing real effort is tiny — sideload a minimal Connect IQ app that does a single phone-free `makeWebRequest` `PUT` to the homeserver's `/send` endpoint over LTE and confirm the message lands in the Matrix room. If that one call works with the phone in another room, the whole approach is validated and the rest is ordinary app-building.

---

## Sources

**Path 1 — Garmin backend / reverse engineering**
- Gadgetbridge Garmin protocol internals — https://gadgetbridge.org/internals/specifics/garmin-protocol/
- Gadgetbridge Garmin devices (inReach messaging not implemented) — https://codeberg.org/Freeyourgadget/Gadgetbridge/wiki/Garmin-devices
- cyberjunky/python-garminconnect — https://github.com/cyberjunky/python-garminconnect
- matin/garth (deprecated; auth flow broke) — https://github.com/matin/garth
- j-arens/garmin-ipc (enterprise IPC, contrast) — https://github.com/j-arens/garmin-ipc
- inReach Portal Connect developer portal — https://developer.garmin.com/inreach-portal/
- Garmin terms of use — https://www.garmin.com/en-US/legal/terms-of-use/

**Device / connectivity**
- fēnix 8 Pro hands-on (Skylo satellite, LTE, consumer Messenger backend) — https://www.dcrainmaker.com/2025/09/garmin-fenix-8-pro-lte-microled-satellite-messaging-hands-on-everything-you-need-to-know.html
- Fenix 8 Pro feature requirements (connectivity priority) — https://www.garmin.com/en-US/connectivity/fenix8pro/feature-requirements/
- fēnix 8 owner's manual — LTE and satellite connected features (phone-absent LTE behaviour) — https://www8.garmin.com/manuals/webhelp/GUID-EECCAC99-90D6-4AB1-9A3A-EC433D3365E2/EN-US/GUID-10A12B55-5C24-4412-9998-B6B620084AAA.html

**Path 2 — Connect IQ**
- Toybox.Communications API — https://developer.garmin.com/connect-iq/api-docs/Toybox/Communications.html
- WatchUi.TextPicker API — https://developer.garmin.com/connect-iq/api-docs/Toybox/WatchUi/TextPicker.html
- Application.Storage API — https://developer.garmin.com/connect-iq/api-docs/Toybox/Application/Storage.html
- GarminHomeAssistant (self-hosted CIQ app; Wi-Fi/LTE mode) — https://github.com/house-of-abbey/GarminHomeAssistant
- GarminHomeAssistant troubleshooting (cert chain, connectivity) — https://github.com/house-of-abbey/GarminHomeAssistant/blob/main/TroubleShooting.md
- "Adapting Your Connect IQ App to LTE" (connectionInfo :lte) — https://www.garmin.com/en-US/blog/developer/adapting-connect-iq-app-to-lte/
- Connect IQ background service FAQ (5-min / 30-sec limits) — https://developer.garmin.com/connect-iq/connect-iq-faq/how-do-i-create-a-connect-iq-background-service/
- wrist-ntfy (self-hosted messaging CIQ client) — https://github.com/kgs0142/wrist-ntfy

**Matrix**
- Matrix client-server API (login, send, /sync, tokens) — https://spec.matrix.org/latest/client-server-api/
- Low Bandwidth Matrix implementation guide — https://matrix.org/blog/2021/06/10/low-bandwidth-matrix-an-implementation-guide/
- Application Service API — https://spec.matrix.org/v1.14/application-service-api/
- matrix-appservice-bridge — https://matrix.org/docs/projects/as/matrix-appservice-bridge/
