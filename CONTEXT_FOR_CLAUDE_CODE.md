# Fuel Order Management System — Full Context for Claude Code

> **Purpose:** This document captures all client requirements, business constraints, technical decisions, and implementation details for the gas station fuel ordering web app. Pass this to Claude Code to get full context without needing prior conversation history.

---

## 1. Project Overview

A real-time web app for managing daily fuel orders across a network of five gas stations in Honduras. Built to replace a manual, spreadsheet-based workflow with an intelligent ordering system that:

- Forecasts demand using rolling sales history
- Validates cistern compartment assignments against physical tank capacity
- Syncs live across multiple users (Ale + 2 operators)
- Handles cistern sharing between multiple stations on a single truck

**Live app:** https://lustrous-gnome-afff44.netlify.app  
**GitHub repo:** https://github.com/a-escobar/fuel-order-management  
**Firebase project:** `pedidos-combustible`

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18 (via CDN — no build step, in-browser Babel transform) |
| Database | Firebase Firestore (real-time NoSQL cloud sync) |
| Auth | Firebase Authentication — Google OAuth (Google Sign-In) |
| Hosting | Netlify — auto-deploys from GitHub `main` on every push |
| Storage fallback | Browser localStorage (mirrors Firestore; used when offline) |
| File structure | Single `index.html` — entire app in one file, no bundler |

The app intentionally lives in a single HTML file with no build step, making it deployable without Node.js knowledge.

---

## 3. Business Context & Client Requirements

### 3.1 The Five Gas Stations

| Station | Notes |
|---|---|
| Puma Juticalpa | Primary/largest station; has two separate underground tank systems (Set 1 + Set 2) per fuel type |
| Limones | Occasionally shares a cistern with Juticalpa |
| Las Delicias | Part of the "western group" — always shares Cisterna 7150 with Puma Paz + La Granja |
| La Granja | Part of the western group |
| Puma Paz | Part of the western group |

### 3.2 Fuel Types

- **Regular**, **Super**, **Diesel**
- **Critical rule for Juticalpa:** Regular fuel goes ONLY to Set 1. Set 2 can only receive Super and Diesel.
  ```
  JUTA_SETS = {
    'Set 1': ['Regular', 'Super', 'Diesel'],
    'Set 2': ['Super', 'Diesel']
  }
  ```

### 3.3 The Four Cisternas (Trucks)

Each cistern truck holds exactly **8,000 gallons** divided into fixed compartments. Compartment sizes are fixed hardware — they cannot be changed.

| Cistern ID | Compartments (gallons) | Total |
|---|---|---|
| 923 | [2500, 2500, 1100, 900, 1000] | 8,000 |
| 6738 | [2000, 2000, 2000, 1000, 1000] | 8,000 |
| 1909 | [2000, 2000, 1500, 500, 900, 1100] | 8,000 |
| 7150 | [3000, 2500, 1500, 1000] | 8,000 |

### 3.4 Operational Groupings (Critical)

- **Western group** (Las Delicias + Puma Paz + La Granja): ONLY uses Cisterna 7150. These three stations always share one truck. The compartments of that truck get divided among the three stations.
- **Juticalpa**: mainly uses cisternas 923, 6738, 1909; orders approximately twice a week.
- **Limones**: occasionally joins Juticalpa's orders (same cisternas: 923, 6738, 1909). When sharing, Limones gets some compartments and Juticalpa gets the rest.

---

## 4. Core Business Constraints

### 4.1 Compartment Rule (Physical Hardware Constraint)
- **Compartments can only be opened once** — once a compartment valve is opened, all the fuel in it flows into the connected underground tank. You cannot partially unload a compartment.
- Therefore, each compartment must be assigned to exactly ONE underground tank set, and the entire compartment volume goes into that set.
- For Juticalpa specifically: each compartment is assigned to Set 1 OR Set 2 entirely. No splitting.

### 4.2 Capacity Validation
Before assigning a compartment to a set, you must verify it physically fits:
```
available_space = max_capacity(set, fuel) - projected_level_at_delivery - already_assigned_to_set
```
If `compartment_size > available_space`, the assignment is invalid and must be blocked.

### 4.3 Minimum Order Constraint
- Minimum order = **50% of underground tank capacity**
- It is not worth requesting a truck unless you're filling at least half the tank

### 4.4 Order Timing (3-Day Cycle)
Orders follow a strict 3-day sequence:

| Day | Event |
|---|---|
| Day 1 | Order submitted by 12 PM |
| Day 2 | Truck loads at terminal (fuel is locked in) |
| Day 3 | Truck arrives at station (could be afternoon of Day 2 or morning of Day 3) |

**What this means:** When planning an order, you must project the tank level forward to the moment the truck arrives (Day 3 morning), not just look at today's level. The order must ensure the tank doesn't drop dangerously low before the truck arrives.

### 4.5 Level at Delivery Formula
```
level_at_delivery = morning_level_today + descarga_today - Day1_sales - Day2_sales
```
Where `Day1_sales` and `Day2_sales` are forecasted (not yet known).

### 4.6 Sales Calculation (Backward-looking)
```
sale_yesterday = morning_yesterday + descarga_yesterday - morning_today
```
This is how actual sales are computed from the morning readings.

---

## 5. Forecasting Logic

### 5.1 Default Forecast
The forecast for a future day's sales uses a rolling average of the **same weekday** from the past two weeks:
```
forecast_day = average(
  sale_on_same_weekday_last_week,
  sale_on_same_weekday_two_weeks_ago
)
```

### 5.2 Holiday Overrides
For holidays (Semana Santa, Christmas, Feriado Morazánico), sales patterns differ significantly. The app supports manual overrides where the operator can substitute last year's holiday sales instead of the rolling average.

---

## 6. Data Model (Firestore)

All data lives in a single Firestore document at path: `gasolineras/datos`

Top-level keys:

```
config         — tank capacities, station/set/fuel configuration
dailyLog       — daily morning levels and deliveries per station
cisternas      — cistern definitions (compartments, etc.)
orderGroups    — which cisternas belong to which station groupings
nextCisterna   — which cisterna is expected next for each station
```

### 6.1 dailyLog Structure
```
dailyLog[station][date] = {
  morning,        // morning tank levels
  descarga,       // fuel delivered (by compartment)
  descCistId,     // which cistern was used
  descComps,      // compartment assignments
  notes           // optional text
}
```

For **Juticalpa** specifically, morning levels are nested by set:
```
morning = {
  'Set 1': { Regular: <gallons>, Super: <gallons>, Diesel: <gallons> },
  'Set 2': { Super: <gallons>, Diesel: <gallons> }
}
```

For all other stations, morning levels are flat:
```
morning = { Regular: <gallons>, Super: <gallons>, Diesel: <gallons> }
```

### 6.2 nextCisterna
```
nextCisterna = {
  'Puma Juticalpa': '923',
  'Las Delicias': '7150',
  ...
}
```
This is set by the operator after each delivery to indicate which truck is coming next. The order planning tab auto-selects this cistern as the default.

---

## 7. App Features (Current Tabs)

### Tab 1 — Registro Diario
- Enter morning tank levels for each station
- Record incoming deliveries structured by cistern compartments
- Real-time capacity validation: each compartment assignment is checked against available tank space before committing
- Set "next cisterna" for each station

### Tab 2 — Pronóstico 3 Días
- 3-day forward projection per fuel type and set
- Editable estimated sales per day
- Holiday override toggle to substitute last year's data
- Shows projected level at delivery time

### Tab 3 — Planear Pedido
- Compartment allocation: assign each compartment to a station, set, and fuel type
- Auto-optimize button (suggests allocations to maximize fill without overflowing)
- Manual override of any assignment
- Defaults to the "next cisterna" from the daily log
- Validates that all assignments respect tank capacity at delivery time

### Tab 4 — Historial
- View 7 / 14 / 30 day sales history per station per fuel type
- Used to verify forecasts and identify anomalies

### Tab 5 — Configuración
- Tank capacities per station / set / fuel
- Cistern compartment management (add/edit/remove cisternas)
- User access list management (emails allowed in Firestore rules)

---

## 8. Auth & Security

- **Authentication:** Firebase Google OAuth (Google Sign-In)
- **Authorization:** Firestore security rules check that the authenticated user's email is in an explicit allowlist
- **Authorized users:** 3 people (Ale + 2 operators — exact emails in Firestore rules, not stored in code)
- **Rule structure:**
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /gasolineras/datos {
      allow read, write: if request.auth != null
        && request.auth.token.email in [
          "authorized_email_1@...",
          "authorized_email_2@...",
          "authorized_email_3@..."
        ];
    }
  }
}
```

---

## 9. Files in the Repository / Workspace

| File | Purpose |
|---|---|
| `index.html` | Deployed app — the canonical source, connected to GitHub/Netlify CI |
| `gas_station_manager.html` | Legacy standalone version (localStorage only, no Firebase) — kept for reference |
| `README.md` | Developer-facing setup guide |
| `GUIA_CONFIGURACION_NUBE.md` | Spanish-language end-user setup guide (for non-technical operators) |
| `Registro_Gasolineras.xlsx` | Original spreadsheet the app was built to replace |

---

## 10. Known Operational Nuances

1. **Cistern 7150 is exclusively for the western group.** Never assign it to Juticalpa or Limones in the order planning logic.

2. **Limones sharing with Juticalpa is occasional, not always.** The UI must support both cases: Juticalpa ordering alone, and Juticalpa + Limones sharing the same cistern.

3. **Juticalpa Set 2 cannot receive Regular.** Any UI or auto-optimizer must hard-block this assignment.

4. **The "descarga" entry must record which specific compartments were used,** not just total gallons, because this feeds back into capacity tracking for future orders.

5. **Morning levels are the source of truth.** They are entered each morning by an operator and are used to compute yesterday's sales, project forward, and validate orders.

6. **Offline resilience matters.** Operators in the field may have unreliable internet. The localStorage fallback ensures data can be entered and synced later.

7. **No build step by design.** The client (Ale) and operators need to be able to update the app by simply replacing the HTML file, without needing Node.js, npm, or any build tooling. The React + Firebase setup via CDN links is intentional.

---

## 11. Deployment Workflow

1. Edit `index.html` locally (or in the GitHub web editor)
2. Push to `main` branch on GitHub
3. Netlify automatically detects the push and redeploys within ~1 minute
4. Live at: https://lustrous-gnome-afff44.netlify.app

The Firebase config is hardcoded in `index.html` (not in environment variables, since there's no build step). The config is not secret — Firestore security rules are the actual security layer.

---

## 12. Project History / Evolution

The project started as:
1. A manual spreadsheet (`Registro_Gasolineras.xlsx`) — fully manual, error-prone
2. A standalone single HTML file with localStorage — worked on one device, no sharing
3. **Current state:** Cloud-based React + Firebase app with real-time sync, Google auth, and CI/CD deploy

The data model shape was preserved during the localStorage → Firestore migration so that existing logic didn't need to change.

---

*Document generated 2026-05-23. Contact: Ale (y2rmrcq76k@privaterelay.appleid.com)*
