# Lazy Weather

A web clone of the Lazy Weather mobile app. Plain-language forecast — today vs. yesterday, tomorrow vs. today — instead of throwing numbers at you.

Pure black, JetBrains Mono, three-page horizontal pager: **Saved | Today | Tomorrow** (opens on Today). No API keys: forecast and city search from [Open-Meteo](https://open-meteo.com), reverse geocoding from [BigDataCloud](https://bigdatacloud.com).

## Features

- **Today** vs. yesterday, **Tomorrow** vs. today, in plain language ("ABOUT THE SAME", "MUCH WARMER", etc.) — Celsius or Fahrenheit
- **Saved places** page (swipe left from Today): inline city search to add, tap a row to switch active location, tap **Edit** for drag-to-reorder + delete. Persists in `localStorage`
- **City selector pill** with live search; **Use my location** with a graceful Bellevue fallback when geolocation is denied
- Installable as a PWA, ≥44px tap targets throughout

## Files

- `docs/index.html` — single-file React app (deployable; GitHub Pages serves from `/docs`)
- `docs/manifest.webmanifest`, `docs/favicon.svg` — PWA assets

## Run locally

```bash
npx serve docs
# open http://localhost:3000

# or, using Python's built-in server:
python3 -m http.server 8000 -d docs
# open http://localhost:8000
```

Geolocation needs `localhost` or `https://`. On `file://` it falls back to Bellevue.

## Contributing

Issues and PRs welcome at <https://github.com/Ryanb58/lazyweather>.
