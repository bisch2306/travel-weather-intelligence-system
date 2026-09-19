# 🇹🇼 Travel Risk & Weather Intelligence System (Taiwan)

A schedule- and event-driven automation built with **Make.com** that pulls official Taiwanese weather data from the **CWA Open Data API**, derives simple risk indicators, and writes them into a **Notion travel itinerary**. A small standalone HTML widget shows a Meteoblue forecast for the current stop of the trip.

---

## 📌 Purpose

When traveling through regions prone to typhoons and heavy rain, checking forecasts manually is easy to forget. This project automates the following:

- Fetch official hazard alerts and map them to the stops of the itinerary.
- Evaluate the current weather at the active stop and store a severity level (1–4).
- Raise a typhoon flag if a land typhoon warning is active.
- Show a location-aware forecast widget inside the Notion dashboard.

---

## 📐 Architecture & Data Flow

```mermaid
graph TD
    subgraph CWA["CWA Open Data API"]
        A1["W-C0033-002<br/>Hazard Alerts"]
        A2["O-A0003-001<br/>Station Observations"]
        A3["W-C0034-001<br/>Typhoon Warnings"]
    end

    subgraph Make["Make.com (zone eu1)"]
        S1["01 Hazards Scheduler<br/>(HTTP GET on webhook)"] --> H["02 Hazards Pipeline"]
        W1["04 Spot Weather Webhook Listener"] -->|"Make API: run scenario"| P["03 Spot Weather Pipeline"]
    end

    T["Notion automation<br/>(configured outside this repo)"] -->|"calls webhook"| W1

    subgraph Notion["Notion"]
        HD[("Hazard Status DB")]
        ID[("Travel Itinerary DB")]
    end

    A1 --> H
    A2 --> P
    A3 --> P
    H -->|"reads active stops"| ID
    H -->|"reset, then write Rain Start / Rain End"| HD
    P -->|"reads StationID"| ID
    P -->|"writes HazardLevel and Forecast-MAX"| ID

    WG["meteoblue-widget.html<br/>(hard-coded schedule, independent of Notion)"] --> MB["Meteoblue forecast iframe"]
```

> **Note:** The widget does **not** read from Notion. It selects the location from a date schedule that is hard-coded in the HTML file (see [Widget](#4-dashboard-widget-srcmeteoblue-widgethtml)).

---

## 📁 Repository Structure

```
.
├── README.md
├── blueprints/
│   ├── 01_hazards_scheduler.json
│   ├── 02_hazards_main.json
│   ├── 03_spot_weather_main.json
│   └── 04_spot_weather_webhook.json
└── src/
    └── meteoblue-widget.html
```

---

## ⚙️ Subsystems

### 1. Hazard Alerts & Regional Mapping (`01_hazards_scheduler.json`, `02_hazards_main.json`)

**01 – Scheduler:** A single HTTP module that sends a GET request to the webhook URL of scenario 02. The run interval is configured in the Make.com scenario settings and is **not** part of the exported blueprint.

**02 – Hazards Pipeline** (triggered by a custom webhook):

1. **Reset:** Searches the *Hazard Status* database and sets all five fields to empty (`Typhoon Start`, `Typhoon End`, `Storm Level`, `Rain Start`, `Rain End`).
2. **Load itinerary:** Searches the *Travel Itinerary* database for entries with status `🟢 Current`, `🟡 Next Stop` or `⚪ Upcoming`.
3. **Fetch alerts:** Calls CWA dataset `W-C0033-002` and iterates over all returned locations.
4. **Map & filter:** A router filter ("Relevant Location Filter") compares the `Location` property of each itinerary entry with the CWA county/city name:

   | CWA `locationName` | Itinerary `Location` |
   | :--- | :--- |
   | `臺北市` | `Taipei` |
   | `嘉義縣` | `Alishan` |
   | `屏東縣` | `Xiaoliuqiu` |
   | `高雄市` | `Kaohsiung` |
   | `花蓮縣` | `Hualien` |

5. **Write:** For matching pairs, the alert's `startTime` / `endTime` are written to **`Rain Start`** / **`Rain End`** in the *Hazard Status* database.

### 2. Spot Weather & Typhoon Evaluation (`03_spot_weather_main.json`)

1. Reads a fixed Notion page (`{{YOUR_NOTION_MAIN_PAGE_ID}}`) in the itinerary database and resets `Forecast-MAX` and `HazardLevel`.
2. Takes the CWA station ID from the page's formula property **`StationID`** and requests the current observation from `O-A0003-001`.
3. Reads the weather text of the returned station and derives a severity level via nested keyword matching (first match wins, checked from top to bottom):

   | Keywords (Chinese) | Meaning | Level |
   | :--- | :--- | :---: |
   | `雷`, `豪雨`, `大雨` | thunder, torrential rain, heavy rain | **4** |
   | `短暫`, `陣雨`, `小雨` | brief/short-lived, showers, light rain | **3** |
   | `陰`, `雲` | overcast, cloud | **2** |
   | anything else | normal / clear | **1** |

4. Writes the level to **`HazardLevel`**.
5. Requests typhoon warnings from `W-C0034-001` and iterates over the warnings. For each warning the flag is `1` if the type is `陸上颱風警報` (land warning) or `海上陸上颱風警報` (sea and land warning) **and** `expires` is in the future, otherwise `0`.
6. Writes the flag to **`Forecast-MAX`**. (Despite its name, this field holds the typhoon flag.)

### 3. Webhook Listener (`04_spot_weather_webhook.json`)

Receives a call on a Make custom webhook and starts scenario 03 immediately via the Make API (`POST /api/v2/scenarios/{scenarioId}/run`, token authentication). This allows an on-demand refresh, e.g. after changing the itinerary.

The Notion-side trigger that calls this webhook (e.g. a Notion automation or button) is **not** part of this repository and has to be set up separately. Scenario 03 has no schedule in this repository; it runs when triggered through this webhook, or when a schedule is added manually in Make.

### 4. Dashboard Widget (`src/meteoblue-widget.html`)

A standalone HTML page embedding a Meteoblue forecast widget (dark layout, 4 days) in an iframe.

- **Schedule-based location:** `locationForDate()` maps the current date in the `Asia/Taipei` timezone to one of five locations: `taipei`, `alishan`, `xiaoliuqiu`, `kaohsiung`, `hualien`. The schedule is hard-coded and is **not** synchronized with Notion. If the itinerary changes, the dates in this function must be adjusted manually.
- **Manual override:** `?ort=<location>` forces a location, e.g. `meteoblue-widget.html?ort=hualien`.
- **Auto-switch:** Every 60 seconds the page re-evaluates the schedule locally and re-renders the iframe only if the location changed. No network requests are made besides loading the Meteoblue widget.

---

## 🗄️ Required Notion Structure

**Travel Itinerary database** (used by scenarios 02 and 03)

| Property | Type | Used for |
| :--- | :--- | :--- |
| `Location` | Select | Matching against CWA county/city names (values as in the mapping table) |
| `Status` | Status | Must contain `🟢 Current`, `🟡 Next Stop`, `⚪ Upcoming` |
| `StationID` | Formula (text) | CWA station ID of the active stop |
| `HazardLevel` | Number | Weather severity 1–4 |
| `Forecast-MAX` | Number | Typhoon flag (0/1) |

**Hazard Status database** (used by scenario 02)

| Property | Type | Written by 02? |
| :--- | :--- | :---: |
| `Rain Start` | Date | ✅ |
| `Rain End` | Date | ✅ |
| `Typhoon Start` | Date | reset only |
| `Typhoon End` | Date | reset only |
| `Storm Level` | Number | reset only |

> The blueprints reference Notion properties by their **internal IDs**. After importing into your own workspace, re-select the database and remap the fields in each Notion module.

---

## 🚀 Setup

1. **Clone the repository**

   ```bash
   git clone https://github.com/bisch2306/travel-weather-intelligence-system.git
   ```

2. **Import the blueprints into Make.com:** create a new scenario, choose **Options → Import Blueprint** and select one of the files from `blueprints/`. Repeat for all four files. The blueprints target the `eu1.make.com` zone; if your account is in a different zone, adjust the URLs in scenarios 01 and 04.

3. **Replace the placeholders** (see the table below) in the imported modules.

4. **Set up the triggers:** copy the webhook URLs of scenarios 02 and 04 into the corresponding places, schedule scenario 01, and configure the Notion automation that calls webhook 04.

5. **Adjust the widget:** insert your Meteoblue keys and change the dates in `locationForDate()` to your own trip.

### Placeholders

| Placeholder | Where | Description |
| :--- | :--- | :--- |
| `{{CWA_API_KEY}}` | 02, 03 | CWA Open Data API authorization token |
| `{{YOUR_NOTION_CONNECTION_ID}}` | 02, 03 | Your Notion connection in Make |
| `{{NOTION_HAZARDS_DATABASE_ID}}` | 02 | *Hazard Status* database |
| `{{NOTION_ITINERARY_DATABASE_ID}}` | 02 | *Travel Itinerary* database |
| `{{NOTION_MAIN_DATABASE_ID}}` | 03 | *Travel Itinerary* database (same as above) |
| `{{YOUR_NOTION_MAIN_PAGE_ID}}` | 03 | Notion page whose `StationID` is read and whose fields are updated |
| `{{YOUR_HAZARDS_WEBHOOK_ID}}` | 01, 02 | Webhook ID of scenario 02 |
| `{{YOUR_SPOT_WEATHER_WEBHOOK_ID}}` | 04 | Webhook ID of scenario 04 |
| `{{YOUR_SCENARIO_ID}}` | 04 | ID of scenario 03 (target of the run call) |
| `{{YOUR_MAKE_API_TOKEN}}` | 04 | Make API token allowed to run scenarios |
| `{{YOUR_USER_KEY}}`, `{{YOUR_EMBED_KEY}}`, `{{YOUR_SIG}}` | widget | Meteoblue widget credentials |

> **Tip:** Make uses `{{ ... }}` for its own mapping expressions. After import, some placeholders may show up as empty or invalid mappings. Simply overwrite the affected fields with your real values in the module settings.

---

## ⚠️ Known Limitations

- **Typhoon flag:** the flag is written once per warning in the CWA response, so the **last warning** in the list determines the final value. If there are no warnings at all, the loop does not run and `Forecast-MAX` stays empty (reset state) instead of `0`. The flag is not location-specific.
- **Hazard alerts:** all alert types from `W-C0033-002` are written to `Rain Start` / `Rain End`; there is no filtering by phenomenon. The `Typhoon Start`, `Typhoon End` and `Storm Level` fields are reset but never filled.
- **Reset before fetch:** scenario 02 clears the hazard fields before calling CWA. If the API call fails, the fields stay empty until the next successful run.
- **Single-row assumption:** scenarios 02 and 03 assume one row in the *Hazard Status* database and one fixed main page in the itinerary.
- **Missing observations:** the severity logic is keyword-based. A missing or invalid weather value results in Level 1 ("normal").
- **Widget schedule:** duplicated logic – the itinerary in Notion and the widget schedule are maintained separately.

---

## 🛠️ Tech Stack

- **Automation:** Make.com (scenarios, routers, custom webhooks, Make API)
- **Data:** CWA Open Data API (`W-C0033-002`, `O-A0003-001`, `W-C0034-001`), Notion API
- **Frontend:** plain HTML/CSS/JavaScript (`Intl.DateTimeFormat`, iframe embed), Meteoblue widget

---

## 🔐 Security & Privacy

- API keys, tokens, connection IDs, database/page IDs and webhook IDs in the blueprints have been replaced by placeholders (see table above). Never commit real credentials.
- The Make API token in scenario 04 can start scenarios in your account. Keep it secret and scope it as narrowly as possible.
- The widget's `locationForDate()` contains the dates and places of the trip. Replace them with sample dates if you do not want to publish your itinerary.
