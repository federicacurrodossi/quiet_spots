# Quiet Spots — Project Roadmap and Plan

**Course:** Design and Implementation of Mobile Applications (Prof. Baresi), Politecnico di Milano
**Team:** 3 members · **Start:** 19 Oct 2026 · **Target:** January 2027 exam session

Quiet Spots is a Flutter app: a crowdsourced map of study places (libraries, cafés, study rooms) scored by noise level (microphone, decibels only, no audio stored) and ambient light. It shows live conditions and typical hour-by-hour patterns for each spot.

---

## Table of contents

1. [Key decisions](#key-decisions)
2. [Week-by-week roadmap](#1-week-by-week-roadmap)
3. [Architecture](#2-architecture)
4. [Screens and flows](#3-screens-and-flows)
5. [Test plan](#4-test-plan)
6. [Design document outline](#5-design-document-outline)
7. [Presentation outline](#6-presentation-outline-about-1015-minutes)
8. [Repository conventions](#7-repository-conventions)
9. [Risks and mitigations](#8-risks-and-mitigations)

---

## Key decisions

- **Ramp-up.** Only one team member knows Flutter. The first two weeks include a structured ramp-up for the other two, and the Flutter lead owns the architecture skeleton.
- **Devices.** We have a Mac, an iPhone, an Android phone, a tablet and a smartwatch, so we can test properly on both platforms. The watch is a **stretch goal**: Flutter can't build Apple Watch apps (it can build Wear OS ones).
- **Backend: Supabase.**
  - **Location queries.** It runs on Postgres with PostGIS, so "spots within 1 km, sorted by distance" is a single query.
  - **Hour-by-hour patterns.** These are a plain SQL view, with no client-side aggregation to write.
  - **It isn't Firebase**, which directly answers the "not just Firebase" criterion. Firebase Cloud Messaging is used for push notifications only.

---

## 1. Week-by-week roadmap

### Schedule

  - Weekly meetings to check on current situation and decide on next steps.
      - 18/10
        - Creation of repository and setup.
        - Decision on project content and goals
        - Next steps:
          - Research on where to find usable data
          - Research on backend    

| Week | Dates | Goals | Milestone |
|---|---|---|---|
| 1 | 19–25 Oct | B and C do the Flutter codelabs and the Riverpod basics. All three search both app stores for competitors. C sketches wireframes. A sets up the repo, CI and app skeleton. B creates the Supabase project. | |
| 2 | 26 Oct–1 Nov | Technical spikes on iOS and Android: microphone decibels, light sensor, map package. A writes the interfaces and fakes. B drafts the data model. C makes a clickable Figma prototype (phone + tablet). | Architecture skeleton merged |
| 3 | 2–8 Nov | B writes a script to import study places from OpenStreetMap and sets up auth. A builds the measurement pipeline on a background isolate. C builds the map and spot detail screens on fake data. | |
| 4 | 9–15 Nov | Write the one-page proposal (problem, competitor gap, features, criteria coverage). First end-to-end slice: open a spot, measure, submit. | **M1: proposal ready** |
| 5 | 16–22 Nov | Negotiate with the professor and adjust scope. B builds the hourly statistics view and offline cache. C adds the charts. | |
| 6 | 23–29 Nov | Complete flow working on both platforms: map → detail → measure → live status updates. Tablet layouts start. | **M2: MVP** |
| 7 | 30 Nov–6 Dec | "Find me a spot" recommender, favorites, push alerts, weather, add a new spot. Decide yes or no on the watch app. | |
| 8 | 7–13 Dec | Tablet layouts finished, light-level fallback for iOS, settings and privacy screens, visual polish. Design document chapters 1–4 drafted. | |
| 9 | 14–20 Dec | Integration and golden tests. Usability test with 5 students (before they leave for the holidays). | **M3: feature freeze (20 Dec)** |
| 10 | 21–27 Dec | Light week, used as buffer and for bug fixing. | |
| 11 | 28 Dec–3 Jan | Test campaign on the full device matrix. Write up test results. Build the demo seed data. | |
| 12 | 4–10 Jan | Design document finished, slides done, first rehearsal on the real devices. | **M4: document done** |
| 13 | 11–17 Jan | Second rehearsal with timing, final builds installed on the demo devices, check exam registration. | **Exam** |

> ⚠️ Check the official exam dates on webeep as soon as they're published, then shift weeks 12–13 to match.

---

## 2. Architecture

### Layers

Organize the code by feature, and split each feature into clean layers:

- **Presentation:** widgets and Riverpod notifiers.
- **Domain:** entities, use cases (e.g., `SubmitMeasurement`, `RecommendSpots`) and repository interfaces.
- **Data:** the repository implementations, with three kinds of data source: remote (Supabase), local (a SQLite cache using Drift) and sensors.

### State management: Riverpod

- It needs no `BuildContext`.
- Its provider overrides make tests trivial: fakes are swapped in with one line, and the same fakes power the demo mode.
- It's easier to justify architecturally in the design document than GetX.

Navigation uses **go_router**; data models use **freezed**.

### Backend: Supabase

- Postgres with PostGIS, using a `geography` column on spots.
- Row-level security policies so users can only write their own measurements.
- A SQL view for **live status** (the median of measurements from the last 60 minutes).
- A materialized view for **weekday × hour patterns**, refreshed hourly with pg_cron.
- Auth with anonymous sign-in plus email.
- Edge Functions to validate submitted measurements and reject outliers.

### External services (beyond Firebase)

| Service | Purpose |
|---|---|
| Supabase | Database, auth, edge functions |
| OpenStreetMap Overpass API | Initial import of libraries, cafés and study rooms around Milan |
| flutter_map with OSM or MapTiler tiles | Map display (MapTiler's free key avoids OSM's usage limits) |
| Open-Meteo | Weather on the spot detail screen and as a recommender signal (rainy days fill indoor spots). Free, no API key needed. |
| Firebase Cloud Messaging | "Your favorite spot is quiet right now" alerts |

### Threading model

This covers the "multiple threads" criterion:

- The `noise_meter` plugin streams microphone amplitude on the main isolate.
- Raw samples go to a **long-lived background isolate** (communicating via `SendPort`). It converts them to decibels, smooths them, rejects spikes, and computes the average and 90th percentile.
- iOS has no ambient light API, so the iOS fallback analyzes **camera frames for brightness**. That also runs on an isolate.
- Large JSON parsing uses `compute()`.

A diagram of this goes in the design document.

### Offline support

Drift caches nearby spots and their patterns. Measurements taken offline go into a **pending queue**, which syncs when connectivity_plus reports the network is back. This also protects the demo if the exam room Wi-Fi is bad.

### Microphone privacy

- Audio frames are processed in memory and immediately discarded. Only decibel summaries leave the device.
- Measurements happen only during an explicit 30–60 second session the user starts, never in the background.
- A screen explains why before the system permission prompt appears.
- An anonymous user ID is stored with each measurement.
- A "delete my data" button is provided.

All of this is stated in the design document under a GDPR heading.

### Calibration

Phone microphones differ by several decibels. The app shows **categories** (Silent, Quiet, Moderate, Loud) rather than raw decibel values, stores the device model with each measurement, and offers an optional calibration offset in settings.

---

## 3. Screens and flows

### Screen list

1. **Onboarding** with permission explanations (microphone, location, notifications).
2. **Sign in:** anonymous, or email to keep contributions across devices.
3. **Map (home):** markers colored by live noise level, with filters for type, open now, power outlets and Wi-Fi.
4. **List and search:** sorted by distance or quietness.
5. **Spot detail:** live status, weekday × hour chart (fl_chart), amenities, weather, favorite button.
6. **Measure session:** live decibel gauge, light level, countdown, summary, submit.
7. **Add a spot:** pin on the map, type, amenities.
8. **Find me a spot now:** pick preferences (quiet, bright, maximum walking minutes) and get a ranked list.
9. **Favorites and alerts:** per-spot quiet notifications.
10. **Profile:** contribution count and how many spots you've measured.
11. **Settings:** calibration, theme, privacy, data deletion.

### Main flows

- **Find a place:** Find me a spot → result → detail → directions.
- **Contribute:** Map → detail → Measure → submit → live status updates.
- **Stay informed:** Favorite a spot → receive a "quiet now" push.

### Tablet layouts

Breakpoints follow Material 3's window size classes (600 and 840 dp), implemented with `LayoutBuilder`:

- **Map:** map on the left two-thirds, spot detail panel on the right.
- **List:** list and detail side by side.
- **Measure:** gauge on the left, the spot's hourly chart on the right, so live levels can be compared against the usual pattern.
- **Find me a spot:** preferences on the left, results and a mini-map on the right.
- **Settings:** two panes (categories and content).
- **Landscape:** supported throughout.

The Pronto Treno iPad app shown in the course slides is a good reference for the tablet map screen.

### Watch (stretch goal, decide in week 7)

This only makes sense if the watch runs **Wear OS**. A single Flutter screen could show the nearest quiet spot plus a "start measuring" button. If it's an Apple Watch, skip it and put that effort into tablet polish instead.

---

## 4. Test plan

| Type | What | Tools |
|---|---|---|
| Unit | Decibel math, spike filtering, scoring, recommender ranking, repositories (with mocked data sources), sync queue | `test`, mocktail |
| Widget | Every screen in its loading, empty, error and loaded states, using Riverpod overrides | `flutter_test` |
| Golden | 5 key screens × (phone, tablet) × (light, dark) | alchemist |
| Integration | The three main flows end to end on emulators, with fake sensors and a local Supabase (`supabase start` in Docker) | `integration_test` |
| Backend | Row-level security policies, aggregation views, edge function validation | pgTAP or Deno tests |
| Manual / exploratory | Device matrix, permission denial, airplane mode, rotation, background and resume | Checklist |
| Usability | 5 students, 4 tasks each, measuring time on task and errors, plus a short questionnaire | Script + form |

### Sensor mocking

Define a `NoiseSource` and a `LightSource` interface. In tests, `FakeNoiseSource` emits scripted streams (a quiet room, a loud café, sudden spikes). The same fakes power a hidden **demo mode**.

### Coverage targets

- At least **80%** on the domain and data layers.
- At least **60%** overall.
- CI generates an lcov report on every PR.

### Device matrix

- iPhone and Android phone (real devices).
- iPad or Android tablet (real device).
- Emulators for a small phone (around 5.4 inches) and a large tablet.
- Android minimum SDK 24 or 26. According to the version chart in the course slides, that covers 98–99% of devices.
- iOS minimum 15.

### Reporting

The design document includes:

- A table of test counts by type with pass rates.
- The coverage figure.
- Device matrix results.
- The bugs the tests found and how they were fixed.
- Usability findings and the changes made because of them.

---

## 5. Design document outline

1. **Introduction:** the problem, a competitor gap analysis (what we searched for on the stores), scope, definitions.
2. **Requirements:** user stories, functional and non-functional requirements (privacy, battery, offline), assumptions.
3. **Architecture:** overview diagram, layers, component diagram, state management rationale, threading model diagram, backend architecture, deployment.
4. **Data model:** entity-relationship diagram (Spot, Measurement, HourlyStats, Favorite, User), Supabase tables, row-level security policies, local cache schema.
5. **External services:** one subsection per service, covering why we chose it, which endpoints we use, how we handle failures, and quotas.
6. **UI design:** principles taken from the course slides ("I should always know where I am", "pictures are faster than words", avoiding "oceans of buttons"), the screen flow graph, phone and tablet mockups, theme, accessibility.
7. **Implementation details:** the measurement algorithm, calibration, offline sync, the packages used and why.
8. **Testing:** strategy, test plan, results, coverage, device matrix, usability study.
9. **Limitations and future work.**
10. **Appendix:** setup and run instructions.

---

## 6. Presentation outline (about 10–15 minutes)

1. **The problem**, in one image: a packed library on a Tuesday afternoon.
2. **The gap:** what's on the stores today and what's missing.
3. **Live demo, the heart of the talk** (about 5 minutes). Phone and tablet side by side. Find a spot, then measure live in the exam room itself. Show the chart updating and the tablet layout.
4. **Architecture** diagram.
5. **Threading and sensor pipeline.**
6. **External services.**
7. **Multi-device support.**
8. **Test campaign** numbers.
9. **Lessons learned and future work.**

Keep a recorded backup video of the demo, but use it only if both devices fail.

---

## 7. Repository conventions

### Folder structure

```
lib/
  core/              # theme, router, DI, utils, env
  features/
    spots/ {data, domain, presentation}
    measurement/ {data, domain, presentation}
    recommend/ ...
    auth/ ...
test/                # mirrors lib/
integration_test/
supabase/            # migrations/, seed.sql, functions/
tool/                # OpenStreetMap import script
docs/                # design/, testing/, presentation/
```

### Branching and PR rules

- Follow **GitHub Flow**: `main` is protected, and work happens on branches named like `feat/12-measure-screen` or `fix/...`.
- Squash-merge every PR, with **one approving review** required.
- Use **Conventional Commits** messages (`feat:`, `fix:`, `test:`, `docs:`, `refactor:`, `chore:`).
- Link every PR to an issue on the GitHub Projects board.

### CI

`.github/workflows/ci.yml`:

```yaml
name: CI
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with: { channel: stable, cache: true }
      - run: flutter pub get
      - run: dart format --set-exit-if-changed .
      - run: flutter analyze
      - run: flutter test --coverage
```

### Keeping keys out of git

- Keys live in `env.json` (listed in `.gitignore`), with a committed `env.example.json`. Run with:
  ```bash
  flutter run --dart-define-from-file=env.json
  ```
- In CI, keys are stored in **GitHub Secrets**.
- The Supabase anon key is designed to be public and is protected by row-level security. The **service-role key must never appear in the app**.
- Map and other API keys are restricted to the app's bundle ID.

---

## 8. Risks and mitigations

| Risk | Mitigation |
|---|---|
| A similar app already exists | Search both stores thoroughly in week 1, before writing the proposal |
| iOS has no ambient light sensor API | Estimate brightness from the camera on an isolate, or let the user rate it manually. Document this as a deliberate platform-difference decision. |
| iOS limits background microphone use | By design, measurement only happens in foreground sessions |
| Microphone readings vary across phones | Show categories instead of raw values, store the device model, offer calibration |
| Too little crowd data for the demo | Seed realistic hourly patterns for about 30 Milan spots (Polimi libraries and nearby cafés), plus a demo mode with scripted sensors |
| Flutter learning curve for two members | Week 1 ramp-up, pair programming, and A reviewing every early PR |
| Bad Wi-Fi in the exam room | Offline cache plus a phone hotspot as backup |
| Holidays reduce productivity | Feature freeze on 20 Dec, with week 10 as buffer |
| Scope creep | Anything not in the proposal goes into "future work" after M2 |
| Flutter can't build Apple Watch apps | The watch is a stretch goal, built only if the watch runs Wear OS |

---

## First action this week

Search both app stores for existing quiet-place apps. If something similar exists, we need to know **before** writing the proposal, not after.
