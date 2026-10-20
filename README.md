# Jarvis Agent

Jarvis is an Expo (React Native) voice assistant app for university HOD workflow. It pairs a real-time voice AI agent (ElevenLabs) with a live Google Calendar timeline, letting the department head talk to "Jarvis" to review and schedule meetings, while students get a read-only view of the HOD's daily availability.

## Features

- **Voice assistant (HOD)** — Tap the animated orb to start a real-time conversational session powered by the ElevenLabs React Native SDK. Live transcript, speaking/listening state, and a text composer fallback.
- **Live HOD schedule** — Pulls the HOD's Google Calendar public iCal feed and builds a continuous timeline of occupied events and auto-generated *Free Slot / Available* gaps.
- **Calendar sync via voice** — Calendar tool responses and spoken scheduling phrases are parsed and saved as local tasks.
- **Role-based UI** — The HOD gets the voice assistant; students see a read-only schedule portal with day navigation and pull-to-refresh.
- **Local auth & persistence** — SQLite (`expo-sqlite`) on native, `localStorage` on web.
- **Web scraper backend** — Optional Express service that scrapes a URL using axios + cheerio, falling back to Playwright for JavaScript-heavy pages.

## Tech stack

- **Expo SDK 55** / React Native 0.83 / React 19
- **expo-router** (file-based routing)
- **@elevenlabs/react-native** for real-time voice
- **@shopify/react-native-skia** + **react-native-reanimated** for the animated orb
- **expo-sqlite** for auth + task persistence
- **Express + axios + cheerio + Playwright** for the scraper backend

## Project structure

```
src/
  app/
    _layout.tsx                 Root providers (SafeArea, Auth, Conversation) + Stack
    index.tsx                   Main screen (HOD assistant / student schedule portal)
    login.tsx                   Email + password login
  components/
    jarvis-background.tsx       SVG hatch + glow background
    jarvis-orbs.tsx             Animated Skia orb + status pill
    jarvis-toolbar.tsx          Keyboard / mute / end-call controls
    transcript-panel.tsx        Conversation transcript list
    jarvis-icons.tsx            Custom SVG icons
  constants/jarvis-theme.ts     Colors, spacing, radii
  context/auth-context.tsx      Auth provider (user, login, logout)
  hooks/use-jarvis-conversation.ts  ElevenLabs session + calendar tool handling
  services/db.ts                SQLite / localStorage auth, tasks, iCal parsing
backend/
  server.js                     Express scraper (axios+cheerio -> Playwright)
plugins/
  with-safe-user-leave-hint.js  Android crash workaround config plugin
```

## Getting started

### 1. Prerequisites

- Node.js 18+ and npm
- For native builds: Android Studio (Android) and/or Xcode (iOS). This project uses native modules (WebRTC, Skia, SQLite), so **Expo Go is not supported** — use a development build.

### 2. Install dependencies

```bash
npm install
```

### 3. Configure environment

Create a `.env` file in the project root (it is gitignored):

```bash
EXPO_PUBLIC_AGENT_ID=your_elevenlabs_agent_id
EXPO_PUBLIC_GOOGLE_CALENDAR_ICAL_URL=https://calendar.google.com/calendar/ical/<calendar-id>/public/basic.ics
```

- `EXPO_PUBLIC_AGENT_ID` — required to start a voice session.
- `EXPO_PUBLIC_GOOGLE_CALENDAR_ICAL_URL` — optional; the HOD schedule falls back to a default feed if unset.

### 4. Run the app

```bash
npm run android   # Android device/emulator
npm run ios       # iOS simulator
npm run web       # Web (auth uses localStorage)
npm start         # Expo dev server, choose a target
```

Because custom native modules and config plugins are used, generate/rebuild the dev client after native changes:

```bash
npx expo prebuild --clean
npx expo run:android   # or run:ios
```

## Scraper backend

The `backend/` folder is an independent Node/Express service.

```bash
cd backend
npm install          # also runs `npx playwright install chromium`
npm start            # http://localhost:3000
```

Endpoints:

- `GET /health` — health check
- `POST /scrape` — body `{ "url": "https://..." }`
- `GET /scrape?url=https://...`

Response shape: `{ success, method, title, description, content, url }` (content is capped at 20,000 chars).

## Authentication

Authentication is local-only and intended for demo use. Credentials are stored in SQLite (`users` table) / `localStorage` on web.

- HOD account (hardcoded): **`hod@gmail.com` / `1234`** → full voice assistant.
- Any other email/password → created on first login with the `student` role → read-only schedule view.

## Calendar & task model

- Working slots: 08:30–10:30, 10:30–12:30, 12:30–14:30, 14:30–16:30.
- Occupied events come from the iCal feed (including `RRULE`/`EXDATE` recurrence) plus locally saved tasks.
- Gaps between events become *Free Slot / Available* entries, so the timeline always covers the working day.

## Builds (EAS)

`eas.json` defines `development`, `preview`, and `production` profiles. Set `EXPO_PUBLIC_AGENT_ID` (and the calendar URL) as EAS environment variables for `preview`/`production`.

```bash
npx eas build --profile development --platform android
```

## Scripts

| Script | Description |
| --- | --- |
| `npm start` | Start the Expo dev server |
| `npm run android` / `ios` / `web` | Start on a specific platform |
| `npm run lint` | Run ESLint (`expo lint`) |
| `npm run reset-project` | Move starter code to `app-example` and start fresh |

## Notes

- The app requests microphone, camera, and audio permissions (see `app.json`); microphone access is required for voice sessions.
- `plugins/with-safe-user-leave-hint.js` patches `MainActivity.kt` to avoid an Android `NullPointerException` when the app is backgrounded on RN 0.83+.
