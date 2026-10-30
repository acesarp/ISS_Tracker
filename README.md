# Mission Control: ISS Live Tracker

A real-time dashboard that tracks the International Space Station and shows current space weather. You can follow the ISS on a 3D globe or a flat world map, see where it has been and where it's going, and check solar flares and geomagnetic activity reported by NOAA.

**Live:** https://guto.ca/mission-control

## Features

- **Live ISS position**, updated every 3 seconds: latitude, longitude, altitude, speed, and whether the station is in daylight or in Earth's shadow
- **Orbit prediction:** the ground track for the last 45 minutes and the next 92 minutes (about one full orbit), calculated from the station's orbital elements (TLE) with SGP4
- **Two views:** a draggable 3D globe and a flat world map
- **Space weather from NOAA:**
  - solar flare level from the GOES X-ray flux, with a 6-hour chart
  - geomagnetic activity (Kp index) with storm levels
  - solar wind
  - NOAA R/S/G scales for radio blackouts, solar radiation storms and geomagnetic storms
- **Shared server-side caching**, so the public APIs get the same small number of requests no matter how many people are watching

## How it works

```
Browser (React components)
   |  polls every few seconds
Next.js route handlers (server)
   |-- /api/iss            current position   <- wheretheiss.at        (cached ~1 second)
   |-- /api/iss/track      past/future orbit  <- TLE from CelesTrak     (cached for hours,
   |                                              computed with satellite.js  wheretheiss.at as fallback)
   '-- /api/space-weather  flares, Kp, wind   <- NOAA SWPC feeds         (cached ~1 minute)
```

- **The browser never calls the external APIs directly.** Next.js route handlers fetch the data and cache it with `"use cache"` and `cacheLife`, so every visitor shares the same copy. This respects the limits of the free APIs (wheretheiss.at allows about one request per second).
- **The orbit is calculated on the server.** The TLE is downloaded from CelesTrak, with wheretheiss.at as a fallback, and cached for hours. satellite.js then calculates the ground track. Failed downloads aren't cached, so the next request tries again.
- **The maps are drawn with d3-geo** using real country borders from world-atlas: an orthographic projection for the globe and an equirectangular one for the flat map.

## Tech stack

| Area | Technologies |
|---|---|
| Framework | Next.js 16 (App Router, route handlers, cache components), React 19 |
| Language | TypeScript |
| Styling | Tailwind CSS 4 |
| Maps and orbits | d3-geo, topojson-client, world-atlas, satellite.js (SGP4) |
| Data sources | wheretheiss.at, CelesTrak, NOAA Space Weather Prediction Center |
| DevOps | Docker (multi-stage build, standalone output, non-root user), Docker Compose, GitHub Actions, self-hosted runner, IIS reverse proxy |

## Project structure

```
mission-control/
  app/
    page.tsx                 Dashboard page
    api/iss/route.ts         Current ISS position
    api/iss/track/route.ts   Past and future ground track
    api/space-weather/route.ts  NOAA space weather data
  components/
    IssTracker.tsx           Position panel and view switcher
    WorldMap.tsx             Globe and flat map (d3-geo)
    SpaceWeatherPanel.tsx    Flares, Kp, solar wind, NOAA scales
    Sparkline.tsx            Small charts
  lib/
    orbit.ts                 TLE download and orbit calculation
    iss.ts, space-weather.ts Types and helpers
  Dockerfile
  docker-compose.yml
.github/workflows/deploy.yml CI/CD pipeline
```

## Getting started

No API keys are needed. All data sources are public.

### Run locally

```bash
cd mission-control
npm install
npm run dev
```

Open http://localhost:3000/mission-control

### Run with Docker Compose

```bash
cd mission-control
docker compose up -d --build
```

## CI/CD

The GitHub Actions workflow (`.github/workflows/deploy.yml`) runs on every push and pull request to `main`:

1. **Check:** installs the dependencies, runs ESLint, generates the route types and type-checks the project with `tsc`.
2. **Deploy** (pushes to `main` only): a self-hosted runner on the home server builds the Docker image, restarts the container and runs a smoke test that waits for the app to respond, showing the container logs if it doesn't.

The deploy job never runs on pull requests, so code from a fork can't reach the self-hosted runner, and only one deploy runs at a time.

## Data sources

- ISS position: [Where the ISS at?](https://wheretheiss.at/)
- Orbital elements (TLE): [CelesTrak](https://celestrak.org/)
- Space weather: [NOAA Space Weather Prediction Center](https://www.swpc.noaa.gov/)
