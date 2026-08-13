# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

`automation_home` is the GitHub Pages-hosted remote control web app for Capulong Farms' ESP32-based home/farm automation devices. It is the frontend half of a three-part system: **ESP32 firmware** (device) + **this repo** (web app) + **Firebase** (backend/auth).

Live site: https://capulongfarms.github.io/automation_home/

## Architecture

- **`index.html` is a dashboard/landing page, not a device page** (changed 2026-07-13) — pure navigation (three cards linking out), no Firebase calls of its own. Local source: `Home/index.html` in the workspace. Each page is self-contained (own HTML/CSS/JS, no shared includes), deployed via GitHub Pages directly from `main` / root on every push:
  - `index.html` — **Dashboard** (links to the pages below)
  - `button.html` — Button_LED
  - `incubator.html` — Incubator
  - `irrigation.html` — Irrigation Headend **and** Zone Node, tabbed (see below)
- **Firebase Authentication** (email/password) gates access on each device page — only signed-in users can read or write device state. The dashboard itself has no auth gate since it shows no device data.
- **Cloud Firestore** (project `automation-home-4b86f`) holds live device state, one document per device under the `devices` collection (e.g. `devices/button_led`).
- Real-time sync on the web side via Firestore's `onSnapshot` listener — no polling.
- ESP32 firmware source is **not** in this repo — it lives in the Capulong Farms workspace (a separate, private repo), under `Project Details/4.Automation/0Master/Codes/`:
  - `1.Button_LED/1.Button_LED.ino`
  - `12.Incubator/12.Incubator.ino`
  - `13.Headend/13.Headend.ino`
  - `14.Nodes/14.Nodes.ino`
  - That folder also has a `Home/` subfolder — no firmware, just the dashboard's local source, `architecture-diagram.html`, a multi-root `.code-workspace`, and a `README.md` describing this whole deployment strategy (including how to add a new project's page).

## Devices

Four boards, three pages — Irrigation is two boards on one page.

| Firestore document | Device | Page | Description |
|---|---|---|---|
| `devices/button_led` | ESP32 Button_LED | `button.html` | Home LED toggle via physical button (GPIO 25), local web page (AP fallback), or this remote app |
| `devices/incubator` | ESP32 Incubator | `incubator.html` | Egg incubator: heater/humidifier (auto + manual override), fan, light, egg-turner. Also has a fully standalone local OLED + keypad menu, no WiFi required — this remote page is a convenience, not a dependency. |
| `devices/irrigation_headend` | ESP32 Irrigation Headend | `irrigation.html` (Headend tab) | Pump room: duty/standby pumps (auto-alternating), fertigation, filtration, exhaust fan, water level + pressure + rain sensors, and the irrigation calendar |
| `devices/irrigation_node1` | ESP32 Irrigation Zone Node | `irrigation.html` (Zone Node tab) | Six zone control valves, shared soil-moisture probe, rain sensor, one manual button per valve. Field-mounted — usually **out of WiFi range**. |

### Irrigation is the one page that breaks the usual patterns

Three deliberate exceptions, all worth knowing before editing `irrigation.html`:

1. **Two boards, one page.** They are one irrigation system that happens to be split across two ESP32s, so they share one login and get a tab each, with a live status dot per tab so a fault on the tab you are *not* looking at stays visible.
2. **It watches two documents.** The Zone Node tab reads `devices/irrigation_node1` when fresh, and falls back to the Headend's mirror of that state (`nodeValves`, `nodeSoilMoisture`, `nodeLinkAlive` inside `devices/irrigation_headend`) when the node has been silent for over a minute — the expected normal state. In mirror mode the tab is read-only and says so, because valve commands still need the node's own WiFi.
3. **The safety interlock does not run through Firebase at all.** The pump/valve interlock runs board-to-board over LoRa. Nothing on this page is load-bearing for safety, and nothing added here should become so — a control path that depends on the internet cannot be a safety path.

Its local source is `13.Headend/index.html` in the workspace. `14.Nodes/` deliberately has **no** web page — one deployed page, one source file.

## Firebase Project

- Project ID: `automation-home-4b86f`
- **One shared device account for every ESP32** (`device@automation-home.local`), plus a personal login used by this web app. Devices are separated by **Firestore document path only**, not by credential.
- Security Rules restrict all reads/writes to those UIDs — managed in Firebase Console → Firestore Database → Rules (not tracked in this repo).

**This is a settled decision, confirmed 2026-08-13 — do not propose changing it.** An earlier version of this file recommended one dedicated Auth account per device and called reuse a "known divergence." That guidance has been retired. The owner was shown the trade-off explicitly — any one device's credentials can read and write every other device's document, which matters more now that the irrigation Node lives outside a locked building — and chose to keep the shared account, because document-level separation gives the state isolation actually wanted. Revisit only if the owner raises it, or if a device is lost or stolen.

## Key Gotchas

- **The `firebaseConfig` values in the pages are not secret.** `apiKey`, `projectId`, etc. are meant to be public; real access control is enforced server-side by Firestore Security Rules, not by hiding this config.
- **A client-side password prompt is not a security boundary.** The whole page downloads before any JS check runs, so it can be read or bypassed. Auth must go through Firebase Authentication + Security Rules, not app-level checks.
- **The ESP32 side has no real-time listener.** Firestore access via the Firebase-ESP-Client Arduino library is REST-based, so devices poll every ~3 seconds rather than subscribing to a stream. The web app, by contrast, updates instantly via `onSnapshot`.
- **Page filenames are lowercase, and GitHub Pages is case-sensitive.** A mixed-case name has already caused a 404 on a hand-typed URL once (`Button.html` → `button.html`, 2026-07-13). The local Windows filesystem will not catch this for you.
- **Every page here is a deployed *copy*.** The editable source lives in the workspace repo under each project's folder as `index.html`; deploying means copying it in under its deployed name and pushing here. Editing a page in this repo directly means the next deploy silently overwrites it.
- **Adding a new device:** create a new document under `devices/`, reuse the existing `device@automation-home.local` account and the same `API_KEY`/`FIREBASE_PROJECT_ID` constants (see the Firebase Project section above — do not provision a new account), deploy its page here under its own lowercase name (e.g. `newdevice.html`), and add a card for it to `index.html` (the dashboard) — full checklist in the workspace's `Home/README.md`.
