# ESP32 Claude Code Usage Monitor — Project Handoff

> Handoff document for continuing this project in a **local Claude Code session** (on the PC with
> network access to the cabin LAN). It sums up the idea, the owner's precise instructions, decisions
> made so far, and research findings from the initial cloud session (2026-07-16).

---

## 1. Project idea (one paragraph)

A small desktop gadget — an ESP32-C3 "mini TV" with a 1.44" color LCD — that sits on the desk and
shows live Claude Code usage statistics (token/cost/limits) on a multi-page LVGL interface with
Claude branding. It is built **purely with ESPHome**, is adopted by the Home Assistant instance at
the cabin (Petterhaugen), and gets flashed from the ESPHome builder add-on running on that HA
server (the device is plugged into the HA server via USB).

---

## 2. Owner's precise instructions (verbatim intent)

### Device and ESPHome
1. Use **ESPHome** as the basis.
2. Device connected directly to the **Home Assistant server**, which runs the **ESPHome builder** app.
3. **Sensors exposed to HA** and graphics on the display.
4. Create a **new ESPHome device configuration**.
5. Use the template in **`/esphome/template/2024-12 - Template with packages.yaml`** — the
   `substitutions:` and `packages:` sections in particular.
6. **For future use:** improve the template based on findings during this project and save it as a
   **new template**.
7. **Avoid custom components; pure ESPHome only.**

### Hardware
- Spotpear **ESP32-C3 desktop trinket "Mini TV"**, 1.44-inch LCD (ST7735), LVGL-capable, 4 buttons:
  <https://spotpear.com/wiki/ESP32-C3-desktop-trinket-Mini-TV-Portable-Pendant-LVGL-1.44inch-LCD-ST7735.html>
- The wiki description is poor — **derive pins and display type from the schematics and the Arduino
  sample code** linked on that page. Do not guess pin numbers.

### Design
- **Multi-page display application using LVGL.**
- Use the **four device buttons for navigation**.
- Use **Claude logo and design** in the UI (Claude brand: warm terracotta/coral `#D97757` accent,
  dark/cream backgrounds, the starburst logo mark).
- **Page 1:** Graphics / usage bars.
- **Page 2:** Other interesting statistics.
- **Page 3:** List of ongoing projects on the PC → select a project for status.
  *(May be omitted if impossible — see §5.)*
- Original idea was to fetch stats directly from Anthropic servers with ESPHome
  `http_request` — **superseded, see §4.**

### Working mode
- Owner does not want to be involved during the build: **ask questions up front, then work
  autonomously until ready to flash.** Orchestrate with Fable; build with whatever model makes most
  sense; use agents/subagents for the heavy lifting.

---

## 3. Environment & access

| Item | Value |
|---|---|
| Home Assistant (cabin, Petterhaugen) | `10.0.20.22`, SSH port 22 |
| SSH user | `hassio` |
| SSH auth | Password was shared in the original chat — **intentionally not written here** (this repo may be public). Prefer installing an SSH key; consider rotating the password since it was pasted into a chat session. |
| ESPHome template | `/esphome/template/2024-12 - Template with packages.yaml` on the HA box |
| Device connection | Plugged into the HA server via USB → flash from ESPHome builder |
| Anthropic secrets | Reference via `!secret` in YAML; real values go in ESPHome `secrets.yaml` only |

The previous session ran in the cloud and **could not reach 10.0.20.22** (private LAN). That is why
the project moves to local Claude Code — the local session should verify SSH access as its first step:
`ssh hassio@10.0.20.22` (add key first if needed).

---

## 4. Data path — IMPORTANT open decision

**Finding from the cloud session:** the natural-looking source, Anthropic's **Admin Usage & Cost
API** (`GET /v1/organizations/usage_report/messages` and `/v1/organizations/cost_report`, header
`x-api-key: sk-ant-admin01-…`, `anthropic-version: 2023-06-01`, buckets `1m/1h/1d`, polling ≤
1/minute), **requires an org Admin API key and is unavailable for individual accounts.**

**Owner's ruling:** *"Will not work. Different data path. More info later."* — so do **not** build
against the Admin API. The owner will specify the data path; design the firmware so the data source
is swappable.

Likely candidates to be ready for (owner will confirm):
- **PC-side collector → Home Assistant** (recommended architecture regardless of source): a script
  on the PC (e.g. reading Claude Code's local usage data à la `ccusage`, or OTLP metrics from
  `CLAUDE_CODE_ENABLE_TELEMETRY=1`) pushes values into HA via webhook/MQTT/REST. The ESP32 then
  reads **HA entities over the native ESPHome↔HA API** — pure ESPHome, no polling logic on-device,
  sensors double as HA history/dashboards. This satisfies instruction #3 ("sensors exposed to HA")
  in both directions.
- **ntfy** as transport (see §5 comparison).
- Direct HTTPS via `http_request` — keep the package stub, but it's now the fallback, not the plan.

**Design implication:** put the data acquisition in its own ESPHome **package** (e.g.
`packages/data_source_ha.yaml`) so swapping the source never touches the UI pages.

---

## 5. Page 3 ("ongoing projects") — omit vs. ntfy, elaborated

Owner asked for an elaboration comparing **omitting Page 3** against using **ntfy**.

**Option A — Omit Page 3 (ship 2 pages).**
- Zero external moving parts; the device is self-contained once HA sensors exist.
- Firmware is smaller (ESP32-C3 has 4 MB flash and only ~400 KB RAM — LVGL pages are cheap, but
  dynamic lists with per-project drill-down mean more fonts/widgets/text buffers).
- Nothing to maintain on the PC side. Can be added later as a new package without touching pages 1–2.

**Option B — ntfy-fed project list.**
- ntfy is push-oriented pub/sub over HTTP. ESPHome has **no native ntfy subscriber** (no SSE/WebSocket
  client in pure ESPHome), so the ESP would have to **poll** `https://ntfy.sh/<topic>/json?poll=1&since=…`
  with `http_request` and parse newline-delimited JSON in lambdas — workable but clunky and against
  the "pure ESPHome, minimal cleverness" spirit.
- Where ntfy *does* shine here: as the **PC → HA hop**. The PC publishes project status to a ntfy
  topic; HA subscribes (RESTful sensor or a small automation) and exposes entities; ESP reads
  entities natively. That keeps the ESP dumb and pure.
- Payload model: one retained "state" message (JSON: `[{name, status, last_activity}, …]`) rather
  than event spam, so a freshly booted subscriber gets current state.

**Recommendation:** Ship pages 1–2 first (Option A). If Page 3 happens, feed it via
**HA entities** (populated by ntfy or a webhook — owner's choice on the PC side), never by the ESP
talking to ntfy directly. Navigation/drill-down with 4 buttons: up/down to highlight a project,
one button = select/status, one button = back/page-cycle.

---

## 6. Hardware notes gathered so far

- SoC: **ESP32-C3** (single-core RISC-V, 4 MB flash, ~400 KB SRAM, WiFi 2.4 GHz). No PSRAM — keep
  LVGL buffer modest (e.g. 1/4 screen) and limit fonts/images.
- Display: **1.44" ST7735**, 128×128, SPI. ESPHome supports it via the `mipi_spi` /
  `st7735` display platform; LVGL component on top.
- Inputs: **4 physical buttons** (GPIOs from schematic) → `binary_sensor` + LVGL page navigation.
- **Task for local session:** pull exact GPIO mapping (SPI pins, DC, RST, backlight, buttons, and
  the onboard buzzer/LED if present) from the Spotpear schematic PDF / Arduino demo source linked on
  the wiki page. Verify display offsets (ST7735 128×128 panels often need col/row offsets of 1–3 px
  and a specific `invert_colors`/color order).

---

## 7. Plan for the local Claude Code session

1. **Verify SSH to HA** (`hassio@10.0.20.22`); install SSH key if only password works.
2. **Fetch the template** `/esphome/template/2024-12 - Template with packages.yaml`; study its
   `substitutions:` and `packages:` conventions.
3. **Pull hardware facts** from the Spotpear wiki (schematic + Arduino sample) → pin map.
4. **Get the data-path decision from the owner** ("more info later") — then define the HA-side
   sensors/webhook and the PC-side collector if needed.
5. **Build the ESPHome config** as packages: `hardware.yaml` (board/display/buttons),
   `data_source_*.yaml`, `ui_lvgl.yaml` (pages 1–2, Claude branding), device YAML gluing them via
   the template's `substitutions`/`packages` pattern. Pure ESPHome, `!secret` for anything secret.
6. **Compile locally / in builder**, iterate until clean.
7. **Flash** from ESPHome builder (device on USB at the HA server); adopt in HA; verify sensors.
8. **Template improvement deliverable:** fold lessons learned back into a new template
   (e.g. `/esphome/template/2026-07 - Template with packages.yaml`).

### Open questions for the owner
- What is the "different data path" for usage stats?
- Confirm Page 3: omit for v1, or build the HA-fed version (§5)?
- Which usage numbers matter most for Page 1 bars (5-hour window utilization, weekly limit, daily
  tokens, cost)? These drive the layout.

---

## 8. Suggested kickoff prompt for the local session

> Read `esp32-claude-monitor/HANDOFF.md`. Verify SSH access to hassio@10.0.20.22, read the ESPHome
> template `/esphome/template/2024-12 - Template with packages.yaml`, extract the pin map from the
> Spotpear ESP32-C3 Mini-TV schematic/Arduino demo, then build the device config per the handoff
> plan. Ask me only about the data path and Page 3, then work autonomously until ready to flash.
