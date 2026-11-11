# ParkLog

![Tests](https://github.com/caetano14/parklog/actions/workflows/test.yml/badge.svg)
![Expo SDK 54](https://img.shields.io/badge/Expo-SDK%2054-000020?logo=expo)
![React Native](https://img.shields.io/badge/React%20Native-0.81-61DAFB?logo=react&logoColor=white)

**A pocket album for cars.** Spot a car on the street, take a photo, and ParkLog suggests the brand, model and year with a vision model, scores its rarity, pins where you saw it and unlocks achievements as your collection grows. Think of it as a Pokédex for cars.

Built with React Native and Expo in about a week as a portfolio project. Dark UI, works offline, no account needed, and your data stays on your phone.

https://github.com/user-attachments/assets/fcad3476-4699-434d-bef7-75e430a923fa

**[▶ Full demo on YouTube](https://youtu.be/4zAzQh7ZVGI)** · **[⬇ Download the Android APK](https://github.com/caetano14/parklog/releases/latest/download/parklog.apk)** · [Source code](https://github.com/caetano14/parklog)

> The demo videos show the app in English. Portuguese is also built in.

## What this project shows

- **Secure AI integration.** The model is called through a small Cloudflare Worker, so the API key never ships inside the app. The app validates the response again before using it.
- **Product thinking.** Recognition is opt-in, never saves on its own, can be undone, and is offered at the moment it makes sense instead of at install time.
- **Testable logic.** The rarity engine, response validation, backup sanitizing, search and achievements are pure functions covered by 173 unit tests, run on every push by GitHub Actions.
- **Data safety.** Providers only write after loading, imported backups are treated as untrusted input, and importing never deletes or overwrites anything.
- **Offline-first.** Everything lives on the device. There is no backend to keep running except the optional recognition proxy.

## Screenshots

| Home | Add a car | Car detail |
| :---: | :---: | :---: |
| <img src="docs/media/home.png" width="220"> | <img src="docs/media/add.png" width="220"> | <img src="docs/media/detail.png" width="220"> |

| Collection | Achievements | Profile |
| :---: | :---: | :---: |
| <img src="docs/media/collection.png" width="220"> | <img src="docs/media/achievements.png" width="220"> | <img src="docs/media/profile.png" width="220"> |

## Features

- **Add a car** from the camera or the gallery. The photo is resized (1080 px wide, JPEG 75%) and stored on the device.
- **Optional AI recognition** suggests brand, model and year from the photo. It is off by default. High confidence fills the empty fields (including the year when the model only existed in one), medium or low confidence only shows a suggestion with an *Apply* button, and everything can be undone.
- **Rarity engine**: Common, Rare or Exotic, calculated automatically from a brand baseline plus model rules. It cannot be edited by hand, so the same car always gets the same rarity. The country also comes from the brand; it is only asked for when the brand is not in the list.
- **116 brands** (112 with model suggestions, about 1,600 models in total) and country filters.
- **Collection and search**: grouped by brand with sticky headers, searchable by brand and model, filterable by rarity, year and country. A toggle switches between large cards (2 per row) and small cards (4 per row), and the choice is remembered.
- **Edit a car** after saving it (brand, model and year). Country and rarity are recalculated.
- **Location and time**: cars taken with the camera are pinned on a map. Each car shows the date and time it was registered and a Pokédex-style number.
- **50 achievements** across quantity, brands, rarity, time and style.
- **Export and import** the whole garage as a `.zip` with the photos. Profile shows when you last exported, and the share sheet lets you save the copy to iCloud Drive or Google Drive.
- **Portuguese and English**, following the phone language by default.
- **First-launch onboarding** with language and permissions, and a contextual invitation to turn recognition on.

## Try it

### Android

1. Download [`parklog.apk`](https://github.com/caetano14/parklog/releases/latest/download/parklog.apk) on your phone (all versions are on the [Releases page](https://github.com/caetano14/parklog/releases)).
2. Open it. If Android asks, allow installing apps from this source (Settings → Install unknown apps).
3. Google Play Protect may warn that the app is from an unknown developer. That is expected: this is a personal project that is not on the Play Store.

**The APK does not include automatic car recognition.** It is built without the server address and token on purpose, so nobody can run up the author's API bill. The option shows as unavailable in Profile; adding cars, rarity, collection, achievements, map link, and backup all work offline. Recognition is shown in the demo videos, and you can enable it in your own build by deploying the Worker ([`server/README.md`](server/README.md)).

### iOS

There is no public iOS build: installing on an iPhone outside the App Store needs a paid Apple developer account. You can build it yourself (see below) or watch the demo video.

## Tech stack

| Area | Choice |
| --- | --- |
| App | React Native 0.81, Expo SDK 54 (development build, new architecture) |
| Navigation | React Navigation 7 (native stack and bottom tabs) |
| State | React Context (settings, cars, achievements) and a small custom i18n context |
| Storage | AsyncStorage for data, the document directory for photos |
| Lists and images | FlashList, expo-image |
| Backup | jszip, expo-sharing, expo-document-picker |
| AI proxy | Cloudflare Worker calling the Anthropic Messages API |
| Tests and CI | Jest (jest-expo), ESLint (eslint-config-expo), GitHub Actions |

## How it works

```
Screens and navigation  →  Providers (I18n → Settings → Cars → Achievements)
                                   ↓
              Services and pure logic (recognition, rarity, backup, photos)
                       ↓                                 ↓
        AsyncStorage + photo files              Cloudflare Worker  →  Anthropic API
```

### AI recognition without exposing the API key

An API key shipped inside a mobile app can be extracted, so the app never talks to the model directly. It sends the resized photo to a small Cloudflare Worker ([`server/recognize-worker.js`](server/recognize-worker.js), under 80 lines), which holds the key as a secret, validates the request (token, image type, size), calls the model and returns a small JSON object. The app then validates that response again before it touches the form.

- The token in the app bundle is **not a strong secret**. It keeps casual abuse away and can be rotated; the real protection is a monthly spending limit on the API account.
- The model is told to ignore any text inside the image and never to read license plates.
- The Worker does not store or log photos. The location never leaves the phone.
- Only [`src/services/recognition.js`](src/services/recognition.js) knows about the server, so it could be replaced by an on-device model without touching the screens.

### Rarity engine

`calculateRarity(brand, model)` works in two layers: the brand sets a baseline, and per-brand model rules can only raise it, never lower it. Rules match whole words or phrases so that "M3" counts but "M Sport" does not. It is a plain function with no dependencies, which makes it easy to test. Rarity here is a curated opinion, not production data. It is deliberately not editable: it is always derived from the brand and model, so the collection stays consistent.

### Data safety

- Providers only write to storage after they have loaded, so an empty first render never overwrites real data. If the stored cars cannot be read, a copy of the raw text is kept under another key and nothing is written over it.
- Photos are stored by file name, and importing a backup treats the file as untrusted: fields are sanitized and file names cleaned.
- Native modules that are only needed in one place are loaded lazily, so a missing module disables one feature instead of crashing the app.

## Tests

```bash
npm test
```

173 unit tests cover the rarity engine (including the false positives its rules avoid), recognition response validation, backup sanitizing, form rules, reading stored data, brand and model search, filters, achievements and PT/EN translation parity. GitHub Actions runs ESLint and the tests on every push and pull request. The screens and native modules are not covered by automated tests.

## Run it locally

Requirements: Node 20+, Xcode (for iOS) and an iPhone or a simulator.

```bash
npm install
cd ios && pod install && cd ..      # after installing any native package
npx expo run:ios --device           # builds the development build
npx expo start --dev-client         # day-to-day development
```

The app runs without the recognition server. To enable it, deploy the Worker and create a `.env` as described in [`server/README.md`](server/README.md):

```
EXPO_PUBLIC_RECOGNITION_URL=https://parklog-recognize.<your-account>.workers.dev
EXPO_PUBLIC_RECOGNITION_TOKEN=<the APP_TOKEN you set on the Worker>
```

Never commit `.env`. It is listed in `.gitignore`.

## Project structure

```
App.js                     providers and splash handling
__tests__/                 Jest unit tests
locales/                   translated iOS permission texts (PT/EN)
server/                    Cloudflare Worker (recognition proxy)
src/
  achievements/            50 achievement definitions and evaluation
  components/              reusable UI (cards, sheets, banners, error boundary)
  context/                 cars, settings and achievements state
  data/                    brand and model lists
  i18n/                    PT/EN dictionaries and the t() function
  navigation/              AppRoot, stack and tabs
  screens/                 Home, Add, Edit, Collection, Achievements, Profile, Detail, Onboarding
  services/                recognition client and response validation
  utils/                   rarity engine, backup, photos, dates, filters
```

## Limitations and next steps

- Tested on both Android and iPhone, but mostly on iPhone. On Android the car map is replaced by the coordinates and a link to the maps app (the native map would need a Google Maps API key).
- Screens and native modules have no automated tests. Next step: component tests and an end-to-end run.
- All cars are saved as one JSON list. That is fine for hundreds of cars; thousands would call for SQLite.
- The Worker has no per-user rate limiting. A production version would add authentication, rate limits and app attestation.
- Recognition can be wrong (for example a close model variant), which is why it suggests instead of deciding. That is why only high-confidence answers fill the form on their own.
- Backups are not encrypted and only restore cars. Settings and achievements are rebuilt from the cars.

## Privacy

Everything is stored on the device. Photos are only sent to the recognition server if you turn the option on, and only the photo is sent. Location, collection and settings stay on the phone.

## Author

Made by **Miguel Caetano** ([@caetano14](https://github.com/caetano14)).
