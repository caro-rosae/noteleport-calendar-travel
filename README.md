![Version](https://img.shields.io/badge/version-v0.1.0-blue)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![Made with Apps Script](https://img.shields.io/badge/Made%20with-Google%20Apps%20Script-blue)
![Chrome Extension](https://img.shields.io/badge/Chrome-Extension-brightgreen)
<p align="center">
   <source media="(prefers-color-scheme: dark)" srcset="assets/NoTeleport-banner.png">
 <source media="(prefers-color-scheme: light)" srcset="assets/NoTeleport-banner.png">
  <img src="assets/NoTeleport-banner.png" width="100%">
</p>

# NoTeleport — Google Calendar Travel Automation

## Why this exists


> “Every schedule assumes teleportation until proven otherwise.”
> — someone, probably once

Until owning my own TARDIS, I created this tool to automatically generate travel time in my calendar.
With buffers depending on time of travel, with return travels, gaps, and all the good things one might need to make it sustainable and useful since gCal doesn't have this option natively.

---

## What it does
* [Travel event generation](#travel-event-generation)
* [Smart scheduling logic](#smart-scheduling-logic)
* [Automatic updates](#automatic-updates)
* [Clean up reliability](#cleanup--reliability)
* [Multilingual support](#multilingual-support)

### Travel event generation

NoTeleport automatically creates travel events in a dedicated calendar.

```
13:25  Travel → Lesson
14:00  Lesson
15:00  Return travel ← Lesson
```

Features:

* travel-to events before appointments
* return travel after appointments
* configurable gaps so events never touch

---

### Smart scheduling logic

NoTeleport understands real schedules.

* detects back-to-back events
* chains travel between locations
* uses your home address as fallback
* adds dynamic buffer time

---

### Automatic updates

Travel events stay synchronized.

* changing event time updates travel
* changing location updates travel
* large calendar moves still work
* duplicates are prevented

---

### Cleanup & reliability

NoTeleport keeps the travel calendar clean.

* orphan travel events are removed automatically
* event linking survives time changes
* Maps calls are cached and throttled

---

### Multilingual support

Generated travel events support:

* French
* English
* German
* Spanish

The travel calendar name is localized automatically.

---

## Example

Instead of this:

```
14:00 — Rehearsal
```

You get this:

```
13:20 — Travel → Rehearsal
14:00 — Rehearsal
16:00 — Return travel ← Rehearsal
```

---

## Setup (Apps Script)

<details>
<summary>Click to expand step-by-step setup</summary>

### Create the script project

Open:
https://script.google.com

Create a new project named:

```
NoTeleport
```

Add two files:

```
TravelGenerator.gs
WebApp.gs
```

Paste the code from the repository.

---

### Add API key

Project Settings → Script properties

```
API_KEY = noteleport_test_key
```

---

### Run once to authorize

Run:

```
syncTrajets()
```

Allow Calendar and Maps permissions.

---

### Add trigger

Triggers → Add trigger

```
syncTrajets
Time-driven
Every 15 minutes
```

</details>

---

## Chrome companion extension (optional)

The extension provides:

* settings UI
* language selection
* run button
* parameter sync

Load via:

```
chrome://extensions
```

Enable Developer Mode → Load unpacked → `chrome-extension/`.

---

## Design philosophy

This project is intentionally:

* small
* readable
* hackable
* local-first
* free

Automation should reduce friction, not add complexity.

---

## License

MIT

---

## Author

Caroline

