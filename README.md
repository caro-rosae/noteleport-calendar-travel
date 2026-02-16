![Version](https://img.shields.io/badge/version-v0.1.0-blue)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![Made with Apps Script](https://img.shields.io/badge/Made%20with-Google%20Apps%20Script-blue)
![Chrome Extension](https://img.shields.io/badge/Chrome-Extension-brightgreen)

<p align="center">
  <img src="assets/NoTeleport-banner.png" width="100%">
</p>

# NoTeleport — Google Calendar Travel Automation

<a id="top"></a>

## Table of contents

* [I. Why this exists](#i-why-this-exists)
* [II. What it does](#ii-what-it-does)

  * [Travel event generation](#travel-event-generation)
  * [Smart scheduling logic](#smart-scheduling-logic)
  * [Automatic updates](#automatic-updates)
  * [Cleanup & reliability when source events are deleted](#cleanup--reliability-when-source-events-are-deleted)
  * [Multilingual support](#multilingual-support)
* [III. How it works](#iii-how-it-works)

  * [Execution flow](#execution-flow)
  * [Web App & settings sync](#web-app--settings-sync)
* [IV. Configuration](#iv-configuration)
* [V. Setup](#v-setup)

  * [Apps Script setup](#apps-script-setup)
  * [Web App deployment](#web-app-deployment)
* [VI. Chrome companion extension (optional)](#vi-chrome-companion-extension-optional)

  * [Install extension](#install-extension)
  * [Configure and sync](#configure-and-sync)
* [VII. Troubleshooting](#vii-troubleshooting)
* [VIII. Limitations](#viii-limitations)
* [IX. Design philosophy](#ix-design-philosophy)
* [X. License](#x-license)

---

<a id="i-why-this-exists"></a>
## I. Why this exists
<p align="right"><sub>🠐 <a href="#table-of-contents">Table of contents</a> &nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp; <a href="#ii-what-it-does">II. What it does</a> 🠒</sub></p>

> “Every schedule assumes teleportation until proven otherwise.”
> — someone, probably once

Until owning my own TARDIS, I created this tool to automatically generate travel time in my calendar. With buffers depending on time of travel, return travels, gaps, and all the good things one might need to make it sustainable and useful since Google Calendar doesn’t provide this natively.

<p align="right"><sub><a href="#top">Back to top ↑</a></sub></p>

---

<a id="ii-what-it-does"></a>
## II. What it does
<p align="right"><sub>🠐 <a href="#i-why-this-exists">I. Why this exists</a> &nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp; <a href="#iii-how-it-works">III. How it works</a> 🠒</sub></p>

| [Travel event generation](#travel-event-generation) | [Smart scheduling](#smart-scheduling-logic) | [Updates](#automatic-updates) | [Cleanup](#cleanup--reliability-when-source-events-are-deleted) | [Languages](#multilingual-support) |
| --------------------------------------------------- | ------------------------------------------- | ----------------------------- | --------------------------------------------------------------- | ---------------------------------- |

### Travel event generation

NoTeleport automatically creates travel events in a dedicated calendar for each calendar event that has a location.

For every qualifying event, the script:

* creates a **travel-to event** before the appointment
* creates a **return travel event** after the appointment
* stores travel events in a **separate travel calendar**
* ensures travel blocks are **not glued to appointments**
* keeps the system **idempotent** (safe to run repeatedly without duplicates)

Travel events are positioned using configurable timing gaps:

* travel-to ends `gapBeforeEventMin` before the event
* return travel starts `gapAfterEventMin` after the event

Example:

```text
13:20 Travel → Rehearsal
14:00 Rehearsal
16:00 Return travel ← Rehearsal
```

<p align="left"><sub>← <a href="#ii-what-it-does">Back to section</a></sub></p>

---

### Smart scheduling logic

NoTeleport attempts to reflect how a real day unfolds rather than how a calendar looks in isolation.

The script:

* detects **back-to-back events**
* evaluates the time gap between events
* decides whether you:

  * travel **home**, or
  * travel **directly to the next event**
* uses **homeAddress as fallback origin/destination**
* applies a **dynamic buffer based on travel duration**

Chaining behavior is controlled by:

```text
chainIntervalMin
```

If two events are closer than this threshold, travel is chained between locations instead of routing via home.

This prevents unrealistic sequences like:

```text
Lesson ends 15:00
Meeting starts 15:00 in another town
```

<p align="left"><sub>← <a href="#ii-what-it-does">Back to section</a></sub></p>

---

### Automatic updates

Travel events are continuously synchronized with source calendar events.

If you modify an event:

* changing the **time** updates travel events
* changing the **location** recalculates routes
* moving an event far in time still preserves linking
* rerunning the script does not duplicate travel events

This works because NoTeleport stores a durable mapping:

```text
sourceEventId + direction → travelEventId
```

This mapping allows travel events to be updated reliably even after major calendar changes.

<p align="left"><sub>← <a href="#ii-what-it-does">Back to section</a></sub></p>

---

### Cleanup & reliability when source events are deleted

When a source event is deleted, previously generated travel events can become inconsistent.

NoTeleport automatically:

* scans the travel calendar during each run
* detects travel events referencing missing source events
* removes or marks them depending on configuration
* clears stale mapping entries

This keeps the travel calendar stable and predictable over time.

<p align="left"><sub>← <a href="#ii-what-it-does">Back to section</a></sub></p>

---

### Multilingual support

Generated travel events support multiple UI languages:

* French
* English
* German
* Spanish

The travel calendar name is localized automatically unless overridden:

```text
TRAJET
TRAVEL
FAHRT
TRASLADO
```

Language selection affects:

* event titles
* travel calendar name
* extension UI labels

<p align="left"><sub>← <a href="#ii-what-it-does">Back to section</a></sub></p>

<p align="right"><sub><a href="#top">Back to top ↑</a></sub></p>

---

<a id="iii-how-it-works"></a>
## III. How it works
<p align="right"><sub>🠐 <a href="#ii-what-it-does">II. What it does</a> &nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp; <a href="#iv-configuration">IV. Configuration</a> 🠒</sub></p>

| [Execution flow](#execution-flow) | [Web App & settings sync](#web-app--settings-sync) |
| --------------------------------- | -------------------------------------------------- |

### Execution flow

`syncTrajets()` runs as a deterministic pipeline:

```text
Config → Calendars → Scan → Chain decision → Route → Buffers → Upsert → Cleanup
```

This structure ensures that each execution updates travel events predictably without duplication.

---

#### 1. Load configuration

- Defaults live in `TravelGenerator.gs`
- Extension settings override defaults via `USER_SETTINGS_JSON`
- Language selection affects event titles and travel calendar naming

#### 2. Resolve calendars

- Source calendars are found by exact name
- Travel calendar is created automatically if missing

#### 3. Scan events

- Uses a lookback + lookahead window
- Ignores all-day events
- Only includes events with a non-empty **Location**

#### 4. Sort events chronologically

A single merged list is built across all source calendars.

---

#### 5. Chaining decision

If the gap between two events is ≤ `chainIntervalMin`, routing happens directly between event locations instead of routing via home.

Example:

```text
Event A → Event B
```

instead of:

```text
Event A → Home → Event B
```

---

#### 6. Route computation (quota-aware)

Travel duration is computed using Google Maps Directions with protection layers:

- Cache service
- Persisted event signatures
- Max Maps calls per run
- Cooldown flag when quota is hit

---

#### 7. Dynamic buffer

Additional minutes are added based on travel duration to account for real-world transitions (parking, walking, etc.).

---

#### Travel event creation rules

| Direction | Created when | Timing rule | Notes |
|----------|-------------|------------|------|
| OUT | Before appointment | Ends `gapBeforeEventMin` before event start | Start time computed backwards from duration |
| BACK | After appointment | Starts `gapAfterEventMin` after event end | Skipped when chaining to next event |

---

#### 8. Durable mapping

```text
sourceEventId + direction → travelEventId
```

This allows travel events to be updated reliably even after major calendar changes.

---

#### 9. Cleanup phase

Each execution also:

- scans the travel calendar
- detects orphan travel events
- applies `orphanPolicy`
- removes stale mappings


Quota protection layers:

* cache service
* script properties persistence
* max calls per run
* cooldown until next day if quota is hit

<p align="left"><sub>← <a href="#iii-how-it-works">Back to section</a></sub></p>

### Web App & settings sync

The Apps Script Web App exists so the Chrome extension can configure and trigger the script without editing code.

What it does:

* stores settings into Script Properties (as JSON)
* returns status (including quota cooldown info)
* triggers a “run now” execution

Endpoints used by the extension:

* `GET status`
* `GET settings`
* `POST settings`
* `POST run`

Settings are stored in:

```text
USER_SETTINGS_JSON
```

Authorization:

* the Web App checks a shared `API_KEY` stored in Script Properties
* the extension sends the same key with requests

<p align="left"><sub>← <a href="#iii-how-it-works">Back to section</a></sub></p>

<p align="right"><sub><a href="#top">Back to top ↑</a></sub></p>

---

<a id="iv-configuration"></a>
## IV. Configuration
<p align="right"><sub>🠐 <a href="#iii-how-it-works">III. How it works</a> &nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp; <a href="#v-setup">V. Setup</a> 🠒</sub></p>

### Core

| Parameter     | Description                                                                                                 | Example (JSON)                                                       | Allowed values                 |
| ------------- | ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- | ------------------------------ |
| `homeAddress` | The default origin/destination address used when no previous or next event is close enough to chain travel. | `{ "homeAddress": "Main street 42, 1234 Cool Cat City, Kittyland" }` | Any valid address string       |
| `uiLang`      | Language used for generated event titles, calendar naming defaults, and extension UI.                       | `{ "uiLang": "en" }`                                                 | `"fr"`, `"en"`, `"de"`, `"es"` |

### Calendars

| Parameter        | Description                                                                                                                                        | Example (JSON)                                     | Allowed values      |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------- | ------------------- |
| `sourceCalNames` | Array of calendar names that NoTeleport scans for events with locations. Calendar names must match exactly as they appear in Google Calendar.      | `{ "sourceCalNames": ["WORK", "FAMILY", "KIDS"] }` | Array of strings    |
| `travelCalName`  | Optional override for the travel calendar name. If empty or undefined, NoTeleport uses the localized default (TRAJET / TRAVEL / FAHRT / TRASLADO). | `{ "travelCalName": "TRAVEL" }`                    | Any string or empty |

### Timing & chaining

| Parameter           | Description                                                                                                                                                                                                                | Example (JSON)                | Allowed values |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- | -------------- |
| `chainIntervalMin`  | Maximum time gap (in minutes) between two events to consider them part of the same “trip chain”. If the gap is smaller than this value, travel is calculated directly between event locations instead of routing via home. | `{ "chainIntervalMin": 120 }` | Integer ≥ 0    |
| `gapBeforeEventMin` | Buffer time (in minutes) between the end of the travel-to event and the start of the calendar event. This prevents travel blocks from being glued to appointments.                                                         | `{ "gapBeforeEventMin": 10 }` | Integer ≥ 0    |
| `gapAfterEventMin`  | Buffer time (in minutes) between the end of the appointment and the start of the return travel event. Useful for packing, conversations, or transitions.                                                                   | `{ "gapAfterEventMin": 15 }`  | Integer ≥ 0    |

### Maintenance

| Parameter      | Description                                                                     | Example (JSON)                 | Allowed values                 |
| -------------- | ------------------------------------------------------------------------------- | ------------------------------ | ------------------------------ |
| `orphanPolicy` | Defines how travel events are handled when their source event no longer exists. | `{ "orphanPolicy": "delete" }` | `"delete"`, `"keep"`, `"mark"` |

<p align="right"><sub><a href="#top">Back to top ↑</a></sub></p>

---

<a id="v-setup"></a>
## V. Setup
<p align="right"><sub>🠐 <a href="#iv-configuration">IV. Configuration</a> &nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp; <a href="#vi-chrome-companion-extension-optional">VI. Chrome companion extension (optional)</a> 🠒</sub></p>

| [Apps Script setup](#apps-script-setup) | [Web App deployment](#web-app-deployment) |
| --------------------------------------- | ----------------------------------------- |

### Apps Script setup

<details>
<summary><b>Step 0 — Before you start (what you need)</b></summary>

You need:

* A Google account that has access to Google Calendar
* A computer (desktop/laptop). This is hard to do from a phone.
* This repository downloaded on your computer (ZIP) or cloned

If you downloaded a ZIP from GitHub:

1. Unzip it somewhere easy to find (Desktop / Downloads).
2. You should see folders like:

   * `apps-script/`
   * `chrome-extension/`
   * `assets/`

</details>

<details>
<summary><b>Step 1 — Open Google Apps Script</b></summary>

1. Open this link in your browser:

   * [https://script.google.com](https://script.google.com)
2. If Google asks you to sign in, sign in with the same account you use for Google Calendar.
3. You should land on the Google Apps Script home page.

Tip: Google Apps Script is basically “Google’s small coding environment” that can automate Google products (Calendar, Sheets, etc.).

</details>

<details>
<summary><b>Step 2 — Create a new Apps Script project</b></summary>

1. Click **New project**
2. A new project opens with a default file (often called `Code.gs`).
3. Rename the project (top-left, next to the Apps Script logo) to:

   * `NoTeleport`

You now have an empty project container that will hold the code.

</details>

<details>
<summary><b>Step 3 — Create the first code file: TravelGenerator.gs</b></summary>

1. On the left sidebar, you’ll see **Files**.
2. Click the **+** button (or “Add file”) → choose **Script**
3. Name the file exactly:

   * `TravelGenerator.gs`
4. Click **Create**

Now open the repo folder on your computer and find:

* `apps-script/TravelGenerator.gs`

Open that file (with any text editor), select everything, copy, then paste into the Apps Script editor in `TravelGenerator.gs`.

Then press **Ctrl+S** (Windows) or **Cmd+S** (Mac) to save.

</details>

<details>
<summary><b>Step 4 — Create the second code file: WebApp.gs</b></summary>

You will repeat the same process for the Web App layer.

1. Left sidebar → **+** → **Script**
2. Name it exactly:

   * `WebApp.gs`
3. Click **Create**

Now find:

* `apps-script/WebApp.gs`

Copy the entire file content and paste it into Apps Script’s `WebApp.gs`.

Save again.

</details>

<details>
<summary><b>Step 5 — Add the API key (Script Properties)</b></summary>

This project uses an API key so that the Chrome extension (or you) can talk to the Apps Script Web App.

1. In Apps Script, click the **gear icon** (Project Settings)
2. Scroll to **Script properties**
3. Click **Add script property**
4. Add:

* Property: `API_KEY`
* Value: a long random string, for example:

  * `noteleport_2026_my_private_key_9f3a1c`

5. Click **Save**

Important:

* Keep it private.
* If someone has your Web App URL + this key, they can trigger your script.

</details>

<details>
<summary><b>Step 6 — Run once (this is where you grant permissions)</b></summary>

Apps Script cannot access your calendar until you explicitly authorize it.

1. At the top of the editor, find the dropdown that selects a function to run.
2. Select:

   * `syncTrajets`
3. Click **Run**
4. A popup appears asking for permissions. Follow the steps:

   * choose your Google account
   * click **Allow**

If you see a warning like “Google hasn’t verified this app”:

* click **Advanced**
* click **Go to NoTeleport (unsafe)**
* click **Allow**

This is normal for personal scripts (you wrote the code, Google can’t pre-verify it).

After this step, the script is allowed to:

* read your calendars
* create/update events in the travel calendar
* call Maps to calculate routes

</details>

<details>
<summary><b>Step 7 — First test (make sure it actually creates travel events)</b></summary>

Before automating anything, confirm it works once.

1. Make sure you have at least one calendar event that:

   * is NOT an all-day event
   * has a **Location** set (Google Calendar event → “Location” field)
2. In Apps Script, run `syncTrajets` again.
3. Open Google Calendar in your browser.
4. Look in the calendar list (left sidebar) for a new calendar:

   * `TRAJET` (French) / `TRAVEL` / `FAHRT` / `TRASLADO` (depending on language)
5. Click that calendar to make sure it is visible.
6. Check your day/week view: you should see travel events around your appointments.

If nothing appears:

* go to [VII. Troubleshooting](#vii-troubleshooting)

</details>

<details>
<summary><b>Step 8 — Add an automatic trigger (so it runs by itself)</b></summary>

A trigger is what makes Apps Script run automatically on a schedule.

1. In Apps Script, click the **Triggers** icon (looks like a clock) in the left sidebar
2. Click **Add Trigger** (bottom right)
3. Configure:

* Choose which function to run: `syncTrajets`
* Select event source: `Time-driven`
* Select type: `Minutes timer`
* Select interval: `Every 15 minutes`

4. Click **Save**

Now NoTeleport will keep your travel events up to date automatically.

</details>

<p align="left"><sub><a href="#v-setup">← Back to section</a></sub></p>

### Web App deployment

<details>
<summary><b>Step 9 — Deploy the Web App (required for the Chrome extension)</b></summary>

The Web App is a small “gateway” that lets the Chrome extension:

* sync settings into the script
* trigger “Run now”
* read status

To deploy:

1. In Apps Script, click **Deploy** (top right)
2. Click **New deployment**
3. Click **Select type** → choose **Web app**
4. Set:

* Description: `NoTeleport Web App`
* Execute as: `Me`
* Who has access: `Anyone with the link`

5. Click **Deploy**
6. Copy the **Web App URL**

It will look like:
`https://script.google.com/macros/s/XXXXXX/exec`

You will paste this URL into the Chrome extension options.

Note: If you redeploy later, the URL may change. If the extension stops working, update it with the new URL.

</details>

<p align="left"><sub><a href="#v-setup">← Back to section</a></sub></p>

<p align="right"><sub><a href="#top">Back to top ↑</a></sub></p>

---

<a id="vi-chrome-companion-extension-optional"></a>
## VI. Chrome companion extension (optional)
<p align="right"><sub>🠐 <a href="#v-setup">V. Setup</a> &nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp; <a href="#vii-troubleshooting">VII. Troubleshooting</a> 🠒</sub></p>

| [Install extension](#install-extension) | [Configure and sync](#configure-and-sync) |
| --------------------------------------- | ----------------------------------------- |

### Install extension

<details>
<summary>Load unpacked extension</summary>

Open:

```text
chrome://extensions
```

Enable Developer Mode → Load unpacked → `chrome-extension/`.

</details>

<p align="left"><sub><a href="#vi-chrome-companion-extension-optional">← Back to section</a></sub></p>

### Configure and sync

<details>
<summary>Configure extension</summary>

Fill:

* Web App URL
* API key
* calendars
* home address
* language

Click **Save & Sync**.

</details>

<p align="left"><sub><a href="#vi-chrome-companion-extension-optional">← Back to section</a></sub></p>

<p align="right"><sub><a href="#top">Back to top ↑</a></sub></p>

---

<a id="vii-troubleshooting"></a>
## VII. Troubleshooting
<p align="right"><sub>🠐 <a href="#vi-chrome-companion-extension-optional">VI. Chrome companion extension (optional)</a> &nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp; <a href="#viii-limitations">VIII. Limitations</a> 🠒</sub></p>

No travel events → check location field and calendar names.  
Maps quota error → wait until next day or reduce scan window.  
Extension unauthorized → verify Web App URL and API key.

<p align="right"><sub><a href="#top">Back to top ↑</a></sub></p>

---

<a id="viii-limitations"></a>
## VIII. Limitations
<p align="right"><sub>🠐 <a href="#vii-troubleshooting">VII. Troubleshooting</a> &nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp; <a href="#ix-design-philosophy">IX. Design philosophy</a> 🠒</sub></p>

Apps Script environment required.  
Maps quotas apply.  
Driving mode only.

<p align="right"><sub><a href="#top">Back to top ↑</a></sub></p>

---

<a id="ix-design-philosophy"></a>
## IX. Design philosophy
<p align="right"><sub>🠐 <a href="#viii-limitations">VIII. Limitations</a> &nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp; <a href="#x-license">X. License</a> 🠒</sub></p>

Small. Local-first. Hackable. Free.  
Calendars should reflect the time between things.  
Teleportation is still in beta.

<p align="right"><sub><a href="#top">Back to top ↑</a></sub></p>

---

<a id="x-license"></a>
## X. License
<p align="right"><sub>🠐 <a href="#ix-design-philosophy">IX. Design philosophy</a> &nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp; <a href="#author">Author</a> 🠒</sub></p>

MIT

<p align="right"><sub><a href="#top">Back to top ↑</a></sub></p>

---

<a id="author"></a>
## Author
<p align="right"><sub>🠐 <a href="#x-license">X. License</a> &nbsp;&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp; <a href="#top">Top</a> 🠒</sub></p>

CARO ROSAE
