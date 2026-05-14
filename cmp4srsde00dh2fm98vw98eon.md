---
title: "CultureHub — I Built a Zero-Server Team Culture Tracker with Vanilla HTML, CSS & JS"
seoTitle: "Track Office social events"
seoDescription: "Want to track your office events here is a simple app to create and track your social events in your office"
datePublished: 2026-05-14T01:15:58.950Z
cuid: cmp4srsde00dh2fm98vw98eon
slug: culturehub-i-built-a-zero-server-team-culture-tracker-with-vanilla-html-css-js
cover: https://cdn.hashnode.com/uploads/covers/6a0519d6a0c15402778c59a1/8efe4e4d-6f60-4653-80d0-fd6a8b290b25.jpg
tags: office, premier-destination-for-social-gatherings, officeeventfood, officeevents, culturehub

---

> Track events, check in attendees, run a live TV leaderboard, and store photo galleries — all in the browser, no backend required.

* * *

## The Problem

Most companies have culture events — trivia nights, team bowlings, charity drives, holiday parties. Someone books the room, sends the invite, and then... nothing gets tracked. Who showed up? Who won? How engaged is each team over time? Six months later you're trying to justify the culture budget with vibes and a few Slack photos.

I wanted a tool that made this effortless: open it at the event, tap people as they walk in, mark the winner, put the leaderboard on the TV. No logins, no setup, no IT tickets.

**CultureHub** is that tool. It runs entirely in the browser, stores everything in `localStorage`, and ships as a folder of plain HTML files.

* * *

## What It Does

Eight pages, one CSS file, one shared data layer:

| Page | Purpose |
| --- | --- |
| `index.html` | Dashboard home — stats, top-5 leaderboard, upcoming events |
| `events.html` | Create and manage events, filter by type |
| `roster.html` | Add people or bulk-import from JSON / Microsoft Teams |
| `checkin.html` | Day-of attendance — tap to check in, mark winners |
| `dashboard.html` | Live leaderboard with TV fullscreen mode |
| `search.html` | Global search across people and events |
| `gallery.html` | Per-event photo grids with drag-drop and slideshow |
| `settings.html` | Org name, backup / restore, danger zone |

* * *

## Features

### 📅 Event Management

Create events with a title, date, type, location, and description. Six built-in types each get a distinct colour badge for fast scanning:

*   🎮 **Game** — trivia nights, competitions
    
*   🥂 **Social** — happy hours, team lunches
    
*   🤝 **Team Building** — workshops, offsites
    
*   🎉 **Holiday** — seasonal parties
    
*   ❤️ **Charity** — volunteer days, fundraisers
    
*   📌 **General** — everything else
    

Each event card shows live check-in counts, registered attendees, points awarded, and photo count at a glance.

* * *

### 👥 Roster & Import

Add people manually (name, team, email) or bulk-import from a JSON array. The importer handles both a native format and the Microsoft Teams / Azure AD admin export:

```json
[
  { "name": "Alice Johnson", "team": "Engineering", "email": "alice@co.com" },
  { "displayName": "Bob Smith", "mail": "bob@co.com", "department": "Marketing" }
]
```

Both formats work — the importer checks for `name` or `displayName`, `email` or `mail`, `team` or `department`. Duplicate detection prevents double-imports.

* * *

### ✅ Check-In & Scoring

The check-in page is the core day-of tool. Open it on a laptop or tablet at the event:

*   **Tap a card** → person is checked in, scores **1 point**
    
*   **Tap 🏆** → mark as winner, scores **3 points**
    
*   **Tap again** → undo the check-in
    
*   **Check All In** → bulk check-in the whole registered list
    

The scoring is intentionally simple:

| Action | Points |
| --- | --- |
| ✅ Attending and checking in | 1 pt |
| 🏆 Winning an event | 3 pts |
| Points cap | None — accumulate forever |

Tie-breaking on the leaderboard: equal points → sorted by number of events attended.

* * *

### 📺 TV Dashboard & Live Leaderboard

The dashboard page doubles as a lobby display. Hit **TV Mode** (or `Alt+T` / `F11`) to go fullscreen with the nav hidden:

*   Animated rank rows with gold/silver/bronze highlights
    
*   A scrolling ticker cycling through top scores and recent events
    
*   Event attendance progress bars in the sidebar
    
*   Auto-refreshes every 30 seconds
    
*   `Esc` to exit TV Mode
    

This is the one that gets the room going at the end of an event.

* * *

### 🖼 Photo Gallery

Per-event photo grids with drag-and-drop upload. Click any photo to open a fullscreen slideshow — navigate with arrow keys, close with `Esc`.

> ⚠️ **Storage note:** Photos are stored as base64 data URLs in `localStorage`, which browsers typically cap at 5–10 MB. Use the built-in Backup feature regularly. A "Storage full" toast will appear if you hit the quota.

* * *

### 🔍 Search & PDF Reports

Global search across people and events with a split-panel detail view — click a result on the left to see full stats on the right without leaving the page.

Three report types, all generated client-side as print-ready popups:

*   **Overview report** — top-10 leaderboard + full events table
    
*   **Event report** — attendance list with status and points
    
*   **Person report** — individual stats with event history
    

* * *

## Getting Started

No install. No build step. No account.

**Step 1 — Configure your org**

Open `settings.html`, set your organisation name. Hit **Load Sample Data** to pre-populate 8 people and 4 events so you can explore every feature immediately.

**Step 2 — Add your team**

Go to `roster.html` → **Import** and paste your team JSON, or add people one by one with **\+ Add Person**.

**Step 3 — Create your first event**

Go to `events.html` → **\+ New Event**. Give it a name, date, type, and location.

**On the day:**

1.  Open `checkin.html` on a tablet at the door
    
2.  Tap people as they arrive
    
3.  Mark winners with 🏆 when the competition ends
    
4.  Open `dashboard.html` on a screen at the front of the room
    

* * *

## Architecture

### The philosophy: no framework, no build step

The whole thing is three files doing all the heavy lifting:

```plaintext
culturehub/
├── data.js      ← data layer (IIFE, localStorage, event bus)
├── utils.js     ← UI helpers (nav, theme, toasts, modals, reports)
└── style.css    ← complete design system (dark + light, ~380 lines)
```

Each HTML page is ~150–280 lines of vanilla JS, all following the same pattern:

```js
// Every page — same three lines to bootstrap
document.getElementById('nav-root').innerHTML = navHTML();
setActiveNav();

// Reactive render — re-runs whenever any data changes
function render() { /* build innerHTML */ }
window.addEventListener('culturehub:updated', render);
render();
```

* * *

### `data.js` — the data layer

An IIFE that exposes the `CH` singleton globally. All reads and writes go through it — nothing touches `localStorage` directly from a page.

```js
const CH = (() => {
  const KEY = 'culturehub_data';

  function save(data) {
    localStorage.setItem(KEY, JSON.stringify(data));
    window.dispatchEvent(new CustomEvent('culturehub:updated', { detail: data }));
  }

  // ... all CRUD methods

  return { getPeople, addPerson, getEvents, addEvent, checkIn, toggleWinner, ... };
})();
```

Every mutation calls `save()`, which serialises the whole state and fires `culturehub:updated`. Every open page listens for that event and re-renders — no state management library needed.

The public API:

```plaintext
CH.getPeople()       CH.addPerson(name, team, email)
CH.getEvents()       CH.addEvent(title, date, type, desc, location)
CH.checkIn(evId, pId)       CH.toggleWinner(evId, pId)
CH.getLeaderboard()         CH.getPersonStats(personId)
CH.exportBackup()           CH.importBackup(jsonStr)
```

* * *

### `utils.js` — shared UI helpers

Loaded on every page after `data.js`. Provides:

```plaintext
Theme.toggle()          toast(msg, type, duration)
openModal(html)         confirmDialog(message, title)
avatarEl(name, size)    fmtDate(dateStr)
eventTypeBadge(type)    generateReport(type, id)
navHTML()               setActiveNav()
```

The nav is a function — `navHTML()` returns the header markup as a string, injected into a `<div id="nav-root">` on every page. It reads the org name from settings on each call, so renaming the org updates all pages on next load.

* * *

### `style.css` — the design system

A single CSS file with full dark/light theming via custom properties:

```css
:root,
html[data-theme="dark"] {
  --bg:     #07090f;
  --accent: #00c9b1;
  --gold:   #ffb830;
  /* ... */
}

html[data-theme="light"] {
  --bg:     #f2f5fc;
  --accent: #007f6f;
  /* ... */
}
```

All colours, shadows, and radii reference these tokens — switching theme is a single attribute change on `<html>`.

**FOUC prevention:** Every page has a tiny inline script in `<head>` that applies the saved theme before the stylesheet parses, eliminating any flash of the wrong theme on load:

```html
<script>
  (function(){
    var t = localStorage.getItem("culturehub_theme") || "dark";
    document.documentElement.setAttribute("data-theme", t);
  })();
</script>
```

* * *

## Data Model

Everything lives under a single `culturehub_data` key in `localStorage`:

```json
{
  "version": 1,
  "settings": {
    "orgName": "Our Organization",
    "theme": "dark"
  },
  "people": [
    {
      "id": "lx3k8a",
      "name": "Alice Johnson",
      "team": "Engineering",
      "email": "alice@company.com",
      "avatar": ""
    }
  ],
  "events": [
    {
      "id": "m2p9xq",
      "title": "Summer Trivia Night",
      "date": "2025-07-15",
      "type": "Game",
      "description": "Teams compete in trivia!",
      "location": "Rooftop",
      "gallery": [
        { "id": "...", "url": "data:image/jpeg;base64,...", "caption": "" }
      ],
      "attendees": [
        {
          "personId": "lx3k8a",
          "checkedIn": true,
          "points": 3,
          "isWinner": true
        }
      ]
    }
  ]
}
```

`attendees[].personId` is a soft foreign key to `people[].id`. The leaderboard calculation handles the edge case of an attendee whose person record was later deleted — it skips gracefully rather than crashing.

UIDs are generated as `Date.now().toString(36) + Math.random().toString(36).substr(2, 6)` — not cryptographically secure, but collision-proof enough for a local-only app.

* * *

## Backup & Restore

The **💾 Backup** button in the nav bar (or Settings) downloads the full `culturehub_data` object as a dated JSON file:

```plaintext
culturehub-backup-2025-07-15.json
```

Photos are embedded as base64, so the file is completely self-contained. To restore: Settings → load the file or paste the JSON → click Restore. The importer validates that `people` and `events` keys exist before writing anything.

* * *

## Keyboard Shortcuts

| Shortcut | Action |
| --- | --- |
| `Alt + T` | Toggle TV Mode on Dashboard |
| `F11` | Toggle TV Mode on Dashboard |
| `Esc` | Exit TV Mode |
| `→` | Next photo in slideshow |
| `←` | Previous photo in slideshow |
| `Esc` | Close slideshow or modal |

* * *

## What I'd Change Next

A few things I've noted for future iterations:

*   **Photo storage** — base64 in localStorage is the biggest practical constraint. The natural next step is the [Origin Private File System API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system) (`navigator.storage.getDirectory()`), which would lift the 5–10 MB cap entirely and keep everything local.
    
*   **CSV import** — the roster importer handles JSON today. A CSV path would make it easier to copy-paste from a spreadsheet without any JSON formatting.
    
*   **Points customisation** — the 1pt / 3pt system is hardcoded. Exposing it in Settings would take five minutes and make the app more flexible.
    

* * *

## Final Thoughts

The constraint of "no server, no framework, no build step" turned out to be a feature. The whole app is inspectable, forkable, and deployable by dropping a folder anywhere — a local filesystem, a USB drive, a GitHub Pages site, an intranet server. There's nothing to break, no dependencies to update, no account to log into.

If your team runs culture events and you want something you can actually own, give it a try.

* * *

*Built with vanilla HTML, CSS, and JavaScript. All data stored locally in the browser — no server, no sign-up, no cloud.*