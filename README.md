# AFT – Android Forensics Toolkit
Owner / Author: **Airin Thomas** / **airin1228**
Copyright © 2026 Airin Thomas.
All rights reserved.

AFT is a small, local web application for examining an Android phone. It connects to the phone over USB using **ADB** (Android Debug Bridge), reads call logs, SMS, browser history and application usage, lets you import the same kinds of data from CSV files or browser databases, and produces a printable HTML report with integrity hashes.

It is built with **Python + Flask** (backend, `app.py`) and a single-page HTML interface (`templates/index.html`). Everything runs on your own computer at `http://127.0.0.1:5000`. Nothing is sent over the internet.

> **Legal / ethical note.** Use AFT only on devices you own or are explicitly authorised to examine. AFT is read-only and does not bypass Android security, but the data it can show (messages, call history, browsing) is highly sensitive.

---

## Table of contents

1. [What is in the package](#1-what-is-in-the-package)
2. [Requirements](#2-requirements)
3. [Installation and running](#3-installation-and-running)
4. [Preparing the phone](#4-preparing-the-phone)
5. [The interface, tab by tab](#5-the-interface-tab-by-tab)
6. [How each data source is acquired](#6-how-each-data-source-is-acquired)
7. [Importing data (CSV and database files)](#7-importing-data-csv-and-database-files)
8. [Targeted extraction](#8-targeted-extraction)
9. [Date / time filter](#9-date--time-filter)
10. [Evidence tools (hashing, SQLite inspection, export)](#10-evidence-tools)
11. [The AFT report](#11-the-aft-report)
12. [Where data is stored](#12-where-data-is-stored)
13. [API reference](#13-api-reference)
14. [USB troubleshooting](#14-usb-troubleshooting)
15. [Limitations and things to know](#15-limitations-and-things-to-know)

---

## 1. What is in the package

```
AFT_Android_Forensics_Toolkit/
├── app.py                  Flask backend: ADB handling, parsing, import, report
├── requirements.txt        Python dependency list (Flask only)
├── run_linux.sh            One-step launcher for Linux / Kali
├── run_windows.bat         One-step launcher for Windows
├── README.md               This file
├── templates/
│   └── index.html          The whole web interface (HTML + CSS + JavaScript)
└── data/
    ├── sample_communications.csv        Example call/SMS import file
    ├── sample_browser_history.csv       Example browser import file
    ├── sample_application_history.csv   Example app-usage import file
    └── (created at runtime) case.json, AFT_Report.html, browser_acquired/, uploaded files
```

| File | What it does |
|---|---|
| `app.py` | Contains every feature. It finds `adb`, runs commands on the phone, converts raw output into records, merges imports, calculates hashes, answers the web page's API calls, and builds the report. |
| `requirements.txt` | Contains one line: `Flask>=3.0,<4.0`. No other third-party Python packages are needed; everything else (`sqlite3`, `csv`, `hashlib`, `subprocess`, …) is in the Python standard library. |
| `run_linux.sh` | Creates a virtual environment (`.venv`), upgrades `pip`, installs `requirements.txt`, and starts `app.py`. Stops immediately on any error (`set -e`). |
| `run_windows.bat` | Does the same as the Linux script, using Windows paths (`.venv\Scripts\activate`). |
| `templates/index.html` | The user interface: left navigation, eight tabs, tables, import forms, filters and the report form. |
| `data/sample_*.csv` | Small example files that show the exact column layout expected by the import buttons. |

---

## 2. Requirements

- **Python 3.8+** (with the `venv` module).
- **Flask 3.x** (installed automatically by the launch scripts).
- **Android Platform Tools (`adb`)**
  - Linux/Kali: `sudo apt install adb`
  - Windows: download "platform-tools" from Google and either add it to `PATH` or place the `platform-tools` folder next to `app.py`.
- An Android phone with **USB debugging** enabled and a **data-capable USB cable** (charge-only cables will not work).
- A modern web browser.

### How AFT finds `adb`

AFT searches, in order: your `PATH`; `platform-tools/`, `adb/` or the file itself next to `app.py`; the folders named by the `ANDROID_HOME` / `ANDROID_SDK_ROOT` environment variables; then common install locations (Windows: Android SDK, Downloads, `C:\platform-tools`, `C:\adb`; Linux: `/usr/bin`, `/usr/local/bin`, `/snap/bin`, `/opt/android-sdk`, `~/Android/Sdk`). If none is found, the USB tab tells you so and explains how to install it.

---

## 3. Installation and running

### Linux / Kali

```bash
sudo apt update
sudo apt install adb python3-venv -y
chmod +x run_linux.sh
./run_linux.sh
```

Run AFT from the **same terminal** in which `adb devices -l` already shows your phone. Environment differences (PATH, user, sudo) are the most common reason the phone appears in one place but not the other.

### Windows

Double-click `run_windows.bat` (or run it from Command Prompt). Make sure `adb.exe` is available as described above.

### Open the interface

Go to **http://127.0.0.1:5000**. The server listens on `127.0.0.1` only, so it cannot be reached from other computers. Stop it with `Ctrl+C`.

---

## 4. Preparing the phone

1. Open **Settings → About phone** and tap **Build number** seven times to enable Developer options.
2. Open **Settings → Developer options** and switch **USB debugging** on.
3. Connect the phone with a data cable and set the USB mode to **File Transfer (MTP)**.
4. Unlock the phone. When the **"Allow USB debugging?"** prompt appears, tick **Always allow** and tap **Allow**. Until you do, ADB lists the phone as `unauthorized`.
5. Confirm in a terminal: `adb devices -l` should show the serial with state `device`.

---

## 5. The interface, tab by tab

The left navigation has eight tabs. Calls, Messages, Browser and Applications each work the same way: a table, a **Re-acquire from device** button, a **Targeted extraction** box and a **Date / time filter**.

| Tab | Purpose |
|---|---|
| **Overview** | Case summary. Shows the case ID, counters (calls, messages, browser entries, app events, unique contacts, high-risk "anomalies") and a **Recent Activity** list that merges the newest communications, browser events and app events. Has a **Check USB connection** shortcut, an **Open Browser History** shortcut and an **Export JSON** button. |
| **Call Logs** | All call records: timestamp, contact, direction (incoming, outgoing, missed, …), duration, source, risk. Opening the tab automatically tries to read the phone. Has an **Import call records** button for CSV. |
| **Messages / SMS** | All SMS/MMS records, including the message text when read from the phone. Has an **Import message records** button for CSV. |
| **Browser History** | Page title, URL, browser, visit count, source. Has **Import browser history** (CSV) and **Import browser database** (Chrome `History` / Firefox `places.sqlite`). |
| **Application History** | App name, package, event, duration, source. Has **Import application history** (CSV) and an **Auto-refresh from device every 15 s** checkbox. |
| **USB Device** | Detect the phone, show all ADB devices and their states, restart the ADB server, read basic device metadata, and show raw ADB diagnostic output. |
| **Evidence Tools** | SHA-256 hashing of any file, schema inspection of a SQLite file, and full case export to JSON. |
| **AFT Report** | Enter examiner name and optional target number/keyword, then generate the HTML report. |

### Record fields

Every record carries these fields (not all are shown in every table):

| Field | Meaning |
|---|---|
| `timestamp` | Date and time of the event (`YYYY-MM-DD HH:MM[:SS]`). |
| `contact` / `title` / `app_name` | Who, what page, or which app. |
| `channel` | `CALL` or `SMS` (communications). |
| `direction` | Incoming, Outgoing, Missed, Voicemail, Rejected, Blocked (calls); Incoming, Outgoing, Draft, Outbox, Failed, Queued (SMS). |
| `duration` | Call length or app foreground time (`MM:SS` or `HH:MM:SS`; `—` when not applicable). |
| `url`, `browser`, `visits` | Browser fields. |
| `package`, `event` | Application fields (for example `com.android.chrome`, "Foreground session"). |
| `body` | SMS text (device acquisition only). |
| `risk` | `Low`, `Medium` or `High`. Device acquisition always sets `Low`; other values only come from the `risk` column of imported CSV files. The Overview counts `High` entries as "anomalies". |
| `source` | `ADB` for data read from the phone, `DB import` for browser database imports, empty for CSV/sample rows (shown as "Imported/Sample" in the report). |

---

## 6. How each data source is acquired

Opening a data tab (or pressing **Re-acquire from device**) runs a **read-only** ADB query. If no authorised phone is connected, AFT says so and shows imported data only. Every successful acquisition **replaces** the previous ADB-sourced rows of that module; imported rows are kept.

### 6.1 Call logs
- Reads Android's call-log provider: `content://call_log/calls` (number, date, duration, type, name).
- Call type codes are translated: 1 Incoming, 2 Outgoing, 3 Missed, 4 Voicemail, 5 Rejected, 6 Blocked.
- Timestamps are converted from epoch milliseconds to local time; durations from seconds to `MM:SS` / `HH:MM:SS`.
- If the contact is saved, its name is shown after the number.

### 6.2 Messages / SMS
- Reads `content://sms` (address, date, type, body).
- Type codes: 1 Incoming, 2 Outgoing, 3 Draft, 4 Outbox, 5 Failed, 6 Queued.
- The message text is stored in the record and included in the report.

### 6.3 Browser history
AFT combines up to three methods and merges the results:

1. **Legacy ADB provider** (`content://com.android.chrome.browser/history`, then `content://browser/bookmarks`). Works only on very old Android versions.
2. **Live, no root** – reads the **currently open tabs** of a running Chrome-family browser through its DevTools socket (the same mechanism as `chrome://inspect`). Keep the browser open on the phone with the pages loaded. These rows are labelled "(open tab)" and stamped with the acquisition time, because open tabs have no visit time.
3. **Stored, rooted phone** – with `su` or `adb root`, copies the history databases of Chrome, Chrome Beta, Brave, Edge, Opera, Vivaldi, Samsung Internet and Firefox/Firefox Beta (including `-wal` / `-journal` side files), then parses them. Up to the **20,000 most recent visits** per database are read. Copies are saved in `data/browser_acquired/`.

On a **non-rooted modern phone**, the stored history cannot be read over ADB. Use **Import browser database** (section 7.2) with a file from an authorised backup or image, or import a CSV.

### 6.4 Application history
- Reads `dumpsys usagestats` and rebuilds **foreground sessions** (start time and duration) from resume/pause events. Short gaps (≤ 3 seconds) between pause and resume are merged into one session. Apps still in the foreground are shown as "Opened (still in foreground)".
- If event data is not available, it falls back to the "last time used" value for each package.
- Also lists installed **user-installed apps** (`pm list packages -3`). Apps with no usage events appear without a timestamp ("Installed user app").
- System entries (`android`, `com.android.systemui`) are ignored.
- Up to 5,000 events are kept. Tick **Auto-refresh from device every 15 s** to poll continuously.

### 6.5 Acquisition log
Each acquisition appends an entry to the case: time, module, device serial, number of records and a **SHA-256 hash of the acquired record set**. The last 100 entries are kept.

---

## 7. Importing data (CSV and database files)

Imports are **added** to the existing case. Each uploaded file is saved in the `data/` folder and its **SHA-256** is returned so you can document it. CSV files are read as UTF-8 (a BOM is tolerated) with a header row.

### 7.1 CSV formats

| Import | Required header | Alternative column names accepted |
|---|---|---|
| Communications | `timestamp,contact,channel,direction,duration,risk` | `date`/`datetime` for timestamp; `number`/`address` for contact |
| Browser | `timestamp,title,url,browser,visits,risk` | `visit_time`/`date`/`last_visit_time`; `page_title`; `URL`/`visit_url`; `source`; `visit_count` |
| Applications | `timestamp,app_name,package,event,duration,risk` | `date`/`time`; `application`/`name`; `package_name`/`packageName`; `event_type`/`type`; `foreground_time` |

Defaults when a column is missing: channel `CALL`, risk `Low`, duration `—`, browser `Unknown`, visits `1`, event `Usage record`.
In a communications file, rows whose `channel` is `SMS`, `MMS` or `MESSAGE` go to the **Messages** tab; everything else goes to **Call Logs**.

Example files are in `data/`.

### 7.2 Browser database import
Upload a Chrome-family `History` file or a Firefox `places.sqlite` file (optionally choose the browser name). AFT:
- works on a **temporary copy**, so the original is not modified (and copies `-wal`/`-journal` files if present);
- detects the type (Chrome `urls`/`visits` tables or Firefox `moz_places`/`moz_historyvisits`);
- converts Chrome's 1601-epoch microsecond timestamps and Firefox's Unix microsecond timestamps to normal dates;
- imports up to 20,000 most recent visits, and falls back to per-URL last-visit data if no visit table exists.

---

## 8. Targeted extraction

Each data tab has a **Targeted extraction** box. Enter a phone number or keyword and press **Extract target details**.

- **Phone numbers** (7+ digits) match on the **last 10 digits**, so `+91 98765 43210`, `098765 43210` and `9876543210` all match the same contact.
- **Anything else** is a case-insensitive text search across every field of every record (URL, title, message body, package, …).
- Results show: number of matches, **first seen**, **last seen**, and module-specific statistics:
  - Calls: breakdown by direction and **total call time**.
  - Messages: breakdown by direction.
  - Browser: **top 5 domains**.
  - Applications: number of distinct apps.
- The matching rows are listed beneath, newest first.

---

## 9. Date / time filter

Every data tab has **From** and **To** fields. The table narrows instantly. **Clear filter** restores it. Records with no readable timestamp (for example installed apps with no usage events) are **hidden while a filter is active**.

---

## 10. Evidence tools

| Tool | What it does |
|---|---|
| **Calculate SHA-256** | Upload any file; returns its name, size in bytes and SHA-256. Use it to fingerprint evidence files before and after handling. |
| **SQLite schema inspection** | Upload a `.db`/`.sqlite` file; opens it **read-only** and lists every table with its column names, plus the file's SHA-256. Useful to check what a database contains before importing. |
| **Export case JSON** | Downloads the complete case (`case.json`): case ID, all calls, messages, browser and app records and the acquisition log. |

---

## 11. The AFT report

On the **AFT Report** tab:

1. Enter the **Examiner name**.
2. Optionally enter a **Target number / keyword**.
3. Press **Generate**.

If a phone is connected and authorised, AFT first **re-acquires all four modules**, then writes `data/AFT_Report.html`. Open it at `http://127.0.0.1:5000/report/AFT` (or add `?download=1` to download) and print to PDF from the browser.

Report sections:

| Section | Content |
|---|---|
| Header | Case ID, generation time, examiner, target. |
| 1. Device Information | Manufacturer, model, Android version, SDK level and serial number (or a note that no device was connected). |
| 2. Acquisition Results | For this run: per module, acquired / not acquired, with details or the reason. |
| 3. Evidence Summary | Record count and high-risk count per module. |
| 4. Target Analysis | Only if a target was given: matches, first/last seen, statistics and rows for every module. |
| Data sections | Full tables of Call Logs, Messages / SMS, Browser History and Application History with a **Source** column. |
| Acquisition Log & Integrity | Last 20 acquisitions with their SHA-256 hashes, and the **SHA-256 of the entire case data** at report time. |
| Footer | States that ADB-marked rows were read with read-only queries and that other rows are imported or sample data. |

All values are HTML-escaped in the report.

---

## 12. Where data is stored

Everything is kept in the `data/` folder next to `app.py`:

| Item | Description |
|---|---|
| `case.json` | The case database. Created on first run with **sample data** (case ID `CT-2026-041` and one example record per module). The sample rows remain until you delete `case.json` or edit it. |
| `AFT_Report.html` | The most recently generated report (overwritten each time). |
| `browser_acquired/` | Browser databases copied from a rooted phone. |
| Uploaded files | CSVs, databases and hashed files are saved under their original file name (overwriting a file of the same name). |

To start a new case: stop AFT, move or delete `data/case.json`, and restart. A fresh sample case is created.

---

## 13. API reference

The web page talks to these endpoints; you can also call them yourself (for example with `curl`).

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/` | The web interface. |
| GET | `/api/case` | Counts, contacts, anomalies and all records (newest first). |
| POST | `/api/import/<kind>` | Import CSV; `kind` = `communications`, `browser` or `apps`. |
| POST | `/api/import-browser-db` | Import a Chrome `History` / Firefox `places.sqlite` file (optional `browser` form field). |
| POST | `/api/hash` | Save a file and return its size and SHA-256. |
| POST | `/api/inspect-sqlite` | List tables and columns of a SQLite file. |
| GET | `/api/usb/status` | ADB path, device list, state and a plain-language diagnosis. |
| POST | `/api/usb/restart` | `adb kill-server` then `adb start-server`. |
| POST | `/api/usb/device-info` | Read device properties for a given `serial`. |
| POST | `/api/acquire/<kind>` | Acquire from the phone; `kind` = `calls`, `messages`, `browser` or `apps`. |
| GET | `/api/target?module=&q=` | Targeted extraction. |
| POST | `/api/report` | Generate the report (`examiner`, `target`). |
| GET | `/report/AFT` | View (or `?download=1`) the report. |
| GET | `/api/export` | The whole case as JSON. |

---

## 14. USB troubleshooting

The **USB Device** tab shows the ADB path, ADB's raw output and error text, and (on Linux) `lsusb` output, which tells you whether the operating system sees the phone at all.

| ADB state / symptom | Meaning and fix |
|---|---|
| `device` | Ready. AFT shows "Authorized Android device detected". |
| `unauthorized` | Unlock the phone and accept the USB-debugging/RSA prompt (tick **Always allow**). |
| `offline` | Replug the cable, toggle USB debugging off and on, then restart the ADB server. |
| `no permissions` (Linux) | Missing udev rules: `sudo apt install android-sdk-platform-tools-common`, replug, then `adb kill-server && adb start-server`. |
| No devices listed | Use a data-capable cable, enable USB debugging, set USB mode to File Transfer (MTP), try another port. Check `lsusb` in the tab. |
| Phone shows in terminal but not in AFT | Stop AFT (`Ctrl+C`) and restart it from the **same terminal** where `adb devices -l` works. |
| Two ADB servers (one with `sudo`, one without) | Avoid mixing users. Run `adb kill-server && adb start-server` as one user. |
| "ADB command timed out" | Press **Restart ADB server** or run `adb kill-server` then `adb start-server`. |
| "adb not found" | Install Platform Tools (see section 2) and restart AFT. |

Always run `adb devices -l` first; if it does not show the phone with state `device`, AFT cannot see it either.

---

## 15. Limitations and things to know

- **Read-only, but not a bypass.** AFT uses standard ADB shell commands and content providers. If Android denies permission, AFT reports the denial and suggests the phone maker's backup tool, a lawful forensic acquisition, or CSV import. It never tries to defeat security.
- **Modern Android restricts data.** Call logs, SMS and browser data are often blocked over plain ADB on newer versions. Results depend on the Android version, manufacturer and whether the phone is rooted.
- **Browser history**: on non-rooted phones only open tabs can be read live; stored history requires root or an imported database/CSV.
- **Application history** depends on what `dumpsys usagestats` still holds (Android keeps only recent events).
- **Risk ratings are not automatic.** Acquired data is always marked `Low`; "High" values come only from imported CSV files. The "anomalies" counter is just the count of High rows.
- **Sample data** is present in a new case. Remove it before a real examination so it is not mixed with evidence.
- **Case ID** is stored in `case.json` (default `CT-2026-041`); edit that file to change it.
- **Local only, no login.** The server binds to `127.0.0.1` and has no authentication. Do not expose it to a network.
- **Evidence handling.** AFT copies uploaded files into `data/` and saves hashes, but it is a triage/analysis tool. It is not a write-blocker and does not replace a validated forensic imaging workflow. Hash your source files yourself before and after handling.
- **Time zone.** Timestamps from the phone are converted using the **time zone of the computer running AFT**.
