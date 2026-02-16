# NoTeleport — Calendar Travel Generator

Automatic travel time for Google Calendar events using **Google Apps Script**, with an optional **Chrome companion extension** for settings + language selection.

> “Every schedule assumes teleportation until proven otherwise.”  
> — someone, probably once

---

## Why this exists

Calendars are great at showing **intentions**.
They’re not great at showing **transitions**: travel, buffer, parking, context switching.

This project was built to reduce cognitive load (especially with ADHD), by making your calendar reflect **reality**.

---

## What it does

For each event in your selected calendars that has a **location**, NoTeleport creates separate travel events in a dedicated calendar.

It can:
- Create a **travel-to** event before each appointment
- Create a **return travel** event after the appointment (only if you’re not chaining into a next event)
- **Chain** consecutive events: travel starts from the previous location (not home) if events are close in time
- Add a **dynamic buffer**
- Avoid duplicates (safe to run repeatedly)
- Reduce Maps quota usage (cache + persistence + throttle + cooldown)

---

## Multilingual (FR / EN / DE / ES)

### 1) Travel event language
Titles/descriptions for travel events are localized.

### 2) Travel calendar name is localized too (by default)
By default the travel calendar name depends on the selected language:
- FR: `TRAJET`
- EN: `TRAVEL`
- DE: `FAHRT`
- ES: `TRASLADO`

You can override it in settings if you want a custom name.

---

## Repository structure

- `apps-script/`
  - `TravelGenerator.gs` — main automation (includes settings sync)
  - `WebApp.gs` — mini API for the extension (settings/status/run)
  - `lang/` — optional “one file per language” versions (defaults)
- `chrome-extension/`
  - Chrome MV3 extension with a beginner-friendly options page
  - `_locales/` — FR/EN/DE/ES translations
- `docs/`
  - step-by-step guides

---

# Setup — Apps Script (beginner-friendly, step-by-step)

## Step 0 — You need
- A Google account
- Google Calendar
- A desktop browser (Chrome recommended)

## Step 1 — Create (or choose) your travel calendar
You **do not need** to create it manually:
- the script will create it automatically using the localized default name (`TRAJET`, `TRAVEL`, `FAHRT`, `TRASLADO`)
- or it will use the calendar name you set in settings

If you prefer creating it manually: create a new calendar in Google Calendar with your chosen name.

## Step 2 — Create the Apps Script project
1. Open: https://script.google.com
2. Click **New project**
3. In the left sidebar, click **Files**.
4. Create **two** files:
   - `TravelGenerator.gs` (copy/paste from `apps-script/TravelGenerator.gs`)
   - `WebApp.gs` (copy/paste from `apps-script/WebApp.gs`)

## Step 3 — Set your API key (required for the extension)
1. In Apps Script, click **Project Settings** (gear icon).
2. Scroll to **Script Properties**.
3. Click **Add script property**:
   - Name: `API_KEY`
   - Value: choose a long random string (example: `noteleport_7f9c...`)

## Step 4 — Run once to authorize permissions
1. Select the function `syncTrajets` in the top dropdown.
2. Click **Run**.
3. Google will ask permissions:
   - Calendar access
   - Maps service access
4. Accept permissions.

## Step 5 — Add the automation trigger (recommended)
1. In Apps Script left sidebar: click **Triggers** (clock icon).
2. Click **Add Trigger**
3. Configure:
   - Choose which function to run: `syncTrajets`
   - Select event source: `Time-driven`
   - Select type: `Minutes timer`
   - Interval: `Every 15 minutes`
4. Save

---

# Setup — Web App (required for extension + settings sync)

1. In Apps Script, click **Deploy** (top right)
2. Choose **New deployment**
3. Select **Web app**
4. Settings:
   - Execute as: **Me**
   - Who has access: **Anyone with the link**
5. Click **Deploy**
6. Copy the **Web App URL** (you will paste it into the extension)

---

# Setup — Chrome extension (beginner-friendly)

## Step 1 — Load the extension locally
1. Open Chrome
2. Go to: `chrome://extensions`
3. Enable **Developer mode** (top right)
4. Click **Load unpacked**
5. Select the folder: `chrome-extension/`

## Step 2 — Configure it
1. Find the extension and click **Details**
2. Click **Extension options**
3. Fill:
   - Web App URL (from Apps Script deployment)
   - API key (Script property `API_KEY`)
   - Choose Language (FR/EN/DE/ES)
   - Set your HOME address and calendars
4. Click **Save**
   - This saves locally AND pushes settings into Apps Script (settings sync)

## Step 3 — Run manually once (optional)
Click **Run now** in the extension popup or options page.

---

## License
MIT — see `LICENSE`.

---

## Support (optional)
If this project saves you time/stress, you can add a tip jar:
- GitHub Sponsors
- Ko-fi / Buy Me a Coffee

(Completely optional — the project stays free and open.)

---

## Roadmap
- publish-ready Chrome Web Store package (still free)
- optional public transport mode
- more robust log/status page
