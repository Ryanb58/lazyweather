# Lazy Weather

A web clone of the Lazy Weather mobile app. Plain-language forecast — today vs. yesterday, tomorrow vs. today — instead of throwing numbers at you.

Pure black, JetBrains Mono, two-page horizontal pager. No API keys: weather from [Open-Meteo](https://open-meteo.com), reverse geocoding from [BigDataCloud](https://bigdatacloud.com).

## Files

- `docs/index.html` — deployable build (point GitHub Pages at `/docs`)
- `lazy-weather.html` — single-file dev version
- `lazy-weather.jsx` — drop into a Vite + React + Tailwind project as `App.jsx`

## Run locally

```bash
npx serve .
# open http://localhost:3000/lazy-weather.html
```

Geolocation needs `localhost` or `https://`. On `file://` it falls back to Bellevue.
