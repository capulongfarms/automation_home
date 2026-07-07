# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

`automation_home` is the GitHub Pages-hosted remote control web app for Capulong Farms' ESP32-based home/farm automation devices. It is the frontend half of a three-part system: **ESP32 firmware** (device) + **this repo** (web app) + **Firebase** (backend/auth).

Live site: https://capulongfarms.github.io/automation_home/

## Architecture

- Static site, no build step — `index.html` is a single self-contained file (HTML/CSS/JS), deployed via GitHub Pages directly from `main` / root on every push.
- **Firebase Authentication** (email/password) gates access — only signed-in users can read or write device state.
- **Cloud Firestore** (project `automation-home-4b86f`) holds live device state, one document per device under the `devices` collection (e.g. `devices/button_led`).
- Real-time sync on the web side via Firestore's `onSnapshot` listener — no polling.
- ESP32 firmware source is **not** in this repo — it lives locally at:
  `ESP32 Softwares/3. Code and Libraries/Code/1.Button_LED/1.Button_LED.ino` (Capulong Farms workspace, tracked in a separate repo).

## Devices

| Firestore document | Device | Description |
|---|---|---|
| `devices/button_led` | ESP32 Button_LED | Home LED toggle via physical button (GPIO 25), local web page (AP fallback), or this remote app |

## Firebase Project

- Project ID: `automation-home-4b86f`
- Two Auth accounts by design, both referenced in Firestore Security Rules:
  - Personal login — used by this web app
  - One dedicated **device account per ESP32** (e.g. `device@automation-home.local`), used by the Firebase-ESP-Client Arduino library on the device side — kept separate from the personal login so either credential can be rotated independently without affecting the other
- Security Rules restrict all reads/writes to exactly those UIDs — managed in Firebase Console → Firestore Database → Rules (not tracked in this repo).

## Key Gotchas

- **The `firebaseConfig` values in `index.html` are not secret.** `apiKey`, `projectId`, etc. are meant to be public; real access control is enforced server-side by Firestore Security Rules, not by hiding this config.
- **A client-side password prompt is not a security boundary.** The whole page downloads before any JS check runs, so it can be read or bypassed. Auth must go through Firebase Authentication + Security Rules, not app-level checks.
- **The ESP32 side has no real-time listener.** Firestore access via the Firebase-ESP-Client Arduino library is REST-based, so the device polls every ~3 seconds rather than subscribing to a stream. The web app, by contrast, updates instantly via `onSnapshot`.
- **Adding a new device:** create a new document under `devices/`, give the new ESP32 its own dedicated Firebase Auth account (never reuse another device's credentials), and add its UID to the Firestore Security Rules.
