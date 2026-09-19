# 🇹🇼 Automated Travel Risk & Weather Intelligence System

An enterprise-grade, event- and schedule-driven integration pipeline that ingests official Taiwanese meteorological APIs, processes environmental hazards via a custom rule engine, and dynamically updates a centralized **Notion Travel Intelligence Dashboard** and embedded UI widgets.

---

## 📌 Executive Summary & Business Need

Traveling through regions prone to sudden extreme weather events (e.g., Pacific typhoons, severe torrential rain) requires continuous risk monitoring. Relying on manual forecasts is inefficient, reactive, and prone to human error.

This system provides an automated, end-to-end decision-support solution. It continuously monitors official government hazard feeds, maps alerts to a dynamic travel itinerary, evaluates risk severity thresholds, and provides immediate visual alerts across multiple user interfaces.

---

## 📐 System Architecture & Data Flow

The architecture follows a decoupled, event-driven pattern combining scheduled polling with webhook triggers for instant state updates.

```mermaid
graph TD
    subgraph External Data Sources
        A1["CWA Hazard Alerts API<br/>W-C0033-002"]
        A2["CWA Live Station API<br/>O-A0003-001"]
        A3["CWA Typhoon Warnings API<br/>W-C0034-001"]
    end

    subgraph Orchestration Core
        B1["Cron Scheduler Blueprint"] -->|GET Event| B2["Hazards Processing Pipeline"]
        C1["Notion Action Webhook"] -->|Instant Trigger| B3["Spot Weather Webhook"]
        B3 -->|API Call| B4["Spot Weather Pipeline"]
        
        A1 --> B2
        A2 --> B4
        A3 --> B4
        
        B2 --> D1{"Geographical Mapper"}
        B4 --> D2{"Severity Logic Engine"}
    end

    subgraph Data & Frontend Presentation
        D1 -->|Reset & Commit Alerts| E["Notion Itinerary Database"]
        D2 -->|Update Risk Level & Typhoon Flag| E
        E -->|Read Active Station| F["Client-Side JS Widget"]
        F -->|Render Dynamic Forecast| G["User Interface / Dashboard"]
    end

---

## ⚙️ Core Technical Features & Subsystems

### 1. Hazard Alert & Regional Mapping Pipeline (`blueprints/02_hazards_main.json`)
* **Cron-Triggered Ingestion:** Scheduled execution initiated via a decoupled HTTP ping module (`01_hazards_scheduler.json`).
* **State Reset Engine:** Clears previous hazard timestamps and warning states prior to payload processing, guaranteeing zero stale data / false positives.
* **Geographical Mapping Router:** Evaluates CWA regional alerts (County/City level) against the active travel route using custom routing logic:
  * `臺北市` $\rightarrow$ `Taipei` | `嘉義縣` $\rightarrow$ `Alishan` | `屏東縣` $\rightarrow$ `Xiaoliuqiu` | `高雄市` $\rightarrow$ `Kaohsiung` | `花蓮縣` $\rightarrow$ `Hualien`
* **Automated Persistence:** Updates valid hazard start and end timestamps directly in the Notion database.

### 2. Live Weather Evaluation & Typhoon Threat Engine (`blueprints/03_spot_weather_main.json`)
* **Station-Level Metric Ingestion:** Ingests live observational metrics based on location-specific CWA station IDs.
* **Lexical Severity Parsing:** Parses raw Chinese weather descriptors via nested conditional logic to normalize threat severity:

| Chinese Weather Keywords | Risk Level | Description |
| :--- | :---: | :--- |
| `雷` (Thunder), `豪雨` (Torrential), `大雨` (Heavy Rain) | **Level 4** | Severe / Extreme Hazard |
| `短暫` (Showers), `陣雨` (Squalls), `小雨` (Light Rain) | **Level 3** | Moderate Rain Hazard |
| `陰` (Overcast), `雲` (Cloudy) | **Level 2** | Low Hazard / Cloud Cover |
| *Other / Clear Conditions* | **Level 1** | Normal Conditions |

* **Active Typhoon Alert Evaluation:** Queries active tropical cyclone warnings (`Dataset W-C0034-001`), filtering for active Land or Sea-and-Land warnings (`陸上颱風警報` / `海上陸上颱風警報`) where $expires > now$, automatically raising a system-wide threat flag.

### 3. Event-Driven Real-Time Sync (`blueprints/04_spot_weather_webhook.json`)
* **Instant Processing Loop:** Listens for inbound Notion database events via webhooks to trigger immediate execution of the weather evaluation pipeline, bypassing cron wait times during manual itinerary changes.

### 4. Dynamic Dashboard Component (`src/meteoblue-widget.html`)
* **Timezone-Aware Switching:** Native JavaScript handles time parsing in `Asia/Taipei` timezone to transition embedded forecast widgets according to the active schedule.
* **Override Support & Polling:** Supports URL parameter overrides (`?ort=location`) for flexible previewing and executes background polling loops for continuous UI synchronization.

---

## 🛠️ Tech Stack & Integration Protocols

* **Orchestration & Workflow Automation:** Make.com (Scenario Architecture, Routing, Data Transformers, Custom Webhooks)
* **API Integrations:** Taiwanese Central Weather Administration (CWA) Open Data REST API, Notion API
* **Frontend UI:** JavaScript (ES6+, `Intl.DateTimeFormat`), HTML5, CSS3, iFrame Communication
* **Data Formats:** JSON, REST API Payloads, Webhook Callbacks

---

## 🚀 Setup & Installation Guide

1. **Clone the Repository:**
   ```bash
   git clone [https://github.com/YOUR_USERNAME/travel-weather-intelligence-system.git](https://github.com/YOUR_USERNAME/travel-weather-intelligence-system.git)

### Import Blueprints into Make.com
1. Create a new scenario in Make.com.
2. Click **Options** → **Import Blueprint** and select the corresponding `.json` file from the `/blueprints` directory.

### Configure Environment Variables / Connections
1. Replace `{{CWA_API_KEY}}` with your official CWA Open Data API token.
2. Replace `{{YOUR_NOTION_CONNECTION_ID}}` and database UUID placeholders (`{{NOTION_HAZARDS_DATABASE_ID}}`, `{{NOTION_ITINERARY_DATABASE_ID}}`) with your target Notion API credentials.
3. Update the webhook listener URLs in the scheduler and webhook scenarios.

---

## 🔐 Security & Privacy

All internal database identifiers, private tokens, API keys, and webhook callback URLs in the repository blueprints have been fully sanitized and replaced with environmental template variables (`{{YOUR_...}}`).
