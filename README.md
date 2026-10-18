# Quiet Spots

This repository contains the code base for the **Design and Implementation of Mobile Applications** course project, academic year **2026/27**, at **Politecnico di Milano**.

## About the project

Finding a good place to study is harder than it should be: libraries fill up, cafés get loud, and you only find out once you arrive.

**Quiet Spots** is a cross-platform Flutter app that maps study places (libraries, cafés and study rooms) and scores them by how quiet and bright they are. The data is crowdsourced: users run a short measurement session at a spot, and the phone's sensors measure the noise level and ambient light. Only decibel and light values are recorded; no audio is ever stored.

Combining everyone's measurements, the app shows:

- **Live conditions** for each spot, based on recent measurements.
- **Typical hour-by-hour patterns**, so you can see when a place is usually quiet.
- **Recommendations** for the best nearby spot right now, based on your preferences.

The app supports both phones and tablets, on Android and iOS.

## Tech stack

- **Flutter** (Dart), with Riverpod for state management
- **Supabase** (Postgres + PostGIS) for the backend
- **OpenStreetMap** data for maps and study place import
- **Open-Meteo** for weather information
- **Firebase Cloud Messaging** for push notifications

## Getting started

```bash
git clone https://github.com/YOUR_USERNAME/quiet-spots.git
cd quiet-spots
flutter pub get
flutter run
```

API keys are not committed to the repository. See `env.example.json` for the required keys.

## Documentation

- [Project roadmap](docs/ROADMAP.md)
- Design document: `docs/design/`
- Test campaign: `docs/testing/`

## Team

- Federica Currò Dossi
- Cristina Drinciu
- [Team member 3]