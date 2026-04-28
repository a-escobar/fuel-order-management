# ⛽ Fuel Order Management System

A real-time web app for managing daily fuel orders across a network of five gas stations in Honduras. Built to replace a manual, spreadsheet-based workflow with an intelligent ordering system that forecasts demand, validates cistern compartment assignments, and syncs live across multiple users.

**Live demo:** https://lustrous-gnome-afff44.netlify.app  
**Repository:** https://github.com/a-escobar/fuel-order-management

---

## Features

- **Daily log** — Enter morning tank levels and record incoming cistern deliveries compartment-by-compartment, with real-time capacity validation (ensures each compartment physically fits in the target underground tank before committing)
- **3-day demand forecast** — Automatically calculates projected tank levels at delivery time using a rolling average of the same weekday from the previous two weeks; supports manual overrides for holidays (Semana Santa, Christmas, Feriado Morazánico)
- **Order planning** — Smart compartment allocator that matches cistern compartments to fuel types and underground tank sets (Set 1 / Set 2), with one-click auto-optimization and manual override
- **Multi-station cistern sharing** — Handles cases where two stations split a single 8,000-gallon cistern across its compartments (e.g. Juticalpa + Limones)
- **Real-time sync** — All changes instantly visible to all authorized users via Firebase Firestore live subscriptions
- **Google OAuth** — Sign in with Google; access restricted to a specific allowlist of emails via Firestore security rules
- **Offline resilience** — localStorage mirror ensures the app always loads even without internet; writes sync to Firestore when connection resumes
- **Export / import** — Full JSON backup and restore at any time

## Business Context

The app models the operational constraints of a real fuel distribution network:

- **5 stations:** Puma Juticalpa, Limones, Las Delicias, La Granja, Puma Paz
- **4 cisterns** (8,000 gal each), each divided into fixed compartments that must be unloaded entirely into a single underground tank
- **Juticalpa** has two separate underground tank systems (Set 1 and Set 2) per fuel type; Regular fuel is Set 1 only
- **Order timing:** orders submitted by 12 PM on Day 1, truck loads Day 2, arrives Day 2 afternoon – Day 3 morning; minimum tank level at delivery must stay above 50% capacity
- **Fuel types:** Regular, Super, Diesel

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18 (via CDN, no build step) |
| Styling | Inline styles / CSS-in-JS |
| Database | Firebase Firestore (real-time) |
| Auth | Firebase Authentication — Google OAuth |
| Hosting | Netlify (connected to this repo — auto-deploys on push) |
| Storage fallback | Browser localStorage |

## Setup

### Prerequisites
- A [Firebase](https://firebase.google.com) project with Firestore and Google Authentication enabled
- A [Netlify](https://netlify.com) account connected to this GitHub repo

### 1 — Configure Firebase

In `index.html`, find this block near the top of the `<script>` section and replace with your Firebase project config:

```js
const FIREBASE_CONFIG = {
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "..."
};
```

### 2 — Set Firestore security rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /gasolineras/datos {
      allow read, write: if request.auth != null
        && request.auth.token.email in [
          "authorized@email.com",
          "another@email.com"
        ];
    }
  }
}
```

### 3 — Enable Google Sign-In

Firebase Console → Authentication → Sign-in method → Google → Enable

### 4 — Deploy

Push to `main` — Netlify auto-deploys. Add your Netlify domain to Firebase → Authentication → Authorized domains.

## Project Structure

```
/
├── index.html          # Entire app — React + Firebase in a single file
├── README.md
└── .gitignore
```

The app intentionally lives in a single HTML file with no build step, making it deployable by drag-and-drop or by anyone without Node.js knowledge.

---

*Built for operational use at a fuel distribution company in Juticalpa, Honduras.*
