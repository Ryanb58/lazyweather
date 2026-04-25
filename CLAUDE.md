# Lazy Weather

A web clone of the "Lazy Weather" mobile app — a minimalist forecast that compares today vs. yesterday and tomorrow vs. today in plain language ("ABOUT THE SAME as yesterday", "MUCH WARMER than today") instead of throwing numbers at you. Two-page horizontal pager: page 1 is today, page 2 is tomorrow.

## Visual design

Match these constraints exactly. They're the whole point of the clone.

- **Background:** pure black (`#000`).
- **Font:** JetBrains Mono everywhere. Loaded from Google Fonts.
- **Palette:** white text, `#6b6b6b` for muted/secondary, `#1a1a1a` for the location pill and icon button background.
- **Location pill** in the top-left, with the city wrapped in literal asterisks: `*Bellevue*`. The asterisks are part of the brand voice — keep them.
- **Top-right:** a circular `…` icon button that opens a dropdown menu (units, "use my location").
- **Headline type** is large (~26px), three lines: a normal-weight prefix, a muted ALL-CAPS emphasis line, and a normal-weight suffix.
- **Time-of-day rows** at the bottom of page 1: `Morning / Noon / Evening / Night`, each row showing temp, a glyph, the delta vs. the same hour yesterday, and an up/down arrow.
- **Page indicator:** two small dots at the bottom; active dot is `#e5e5e5`, inactive is `#3a3a3a`.

The aesthetic is deliberately spare — lots of empty vertical space, monospace, no rounded card stacks, no gradients. Resist any urge to "improve" it with shadows, color accents, or icons beyond the few unicode glyphs already in use.

## What the app does

### Page 1 — Today
- Headline compares **today's daily-average temp** to **yesterday's daily-average temp** and produces a phrase like "ABOUT THE SAME as yesterday" / "A LITTLE WARMER as yesterday" / "MUCH COOLER as yesterday".
- Four time-of-day rows (Morning 8am, Noon 12pm, Evening 6pm, Night 10pm) showing today's hourly temp at that hour, a weather glyph, and the delta vs. yesterday's same hour with an up/down arrow.

### Page 2 — Tomorrow
- Headline compares **tomorrow's daily-average temp** to **today's**. If tomorrow's weather code matches a known dramatic condition (fog, snow, rain, thunder), a "flavor" phrase overrides the temperature comparison: "HIDDEN BY FOG", "BURIED IN SNOW", "DROWNED IN RAIN", "RIPPED BY THUNDER", "MISTED OVER".
- The flavor headline is rendered with a small SVG dot-grid glyph between each word, mimicking the original's textured spacers.
- Same four time-of-day rows (Morning / Noon / Evening / Night) showing tomorrow's hourly temp at that hour, a glyph, and the delta vs. today's same hour.

### Menu
- **Units:** Celsius / Fahrenheit. Stored in component state, no persistence yet.
- **Use my location:** retries `navigator.geolocation.getCurrentPosition()`. On failure, falls back to Bellevue (47.6101, -122.2015) with a small "fallback · {reason}" hint under the location pill.

### Location pill (search)
The `*City*` pill is also a button. Tapping it opens a small search popover that hits Open-Meteo's geocoding endpoint (`https://geocoding-api.open-meteo.com/v1/search?name=...&count=5`). Picking a result sets `coords` + `city` directly (no reverse geocode needed — the geocoding payload includes the canonical name) and the weather effect refetches. Search debounces at 250ms and requires ≥2 characters.

## APIs

No keys required for either.

### Open-Meteo (weather)
```
https://api.open-meteo.com/v1/forecast
  ?latitude={lat}&longitude={lon}
  &hourly=temperature_2m,weathercode
  &daily=weathercode,temperature_2m_max,temperature_2m_min
  &past_days=1&forecast_days=2
  &timezone=auto
```
With `past_days=1&forecast_days=2`, `daily.time` returns `[yesterday, today, tomorrow]` and `hourly.time` returns 72 hourly entries in the format `YYYY-MM-DDTHH:00`. The hourly index lookup uses exact string match on that format.

### BigDataCloud (reverse geocoding)
```
https://api.bigdatacloud.net/data/reverse-geocode-client
  ?latitude={lat}&longitude={lon}&localityLanguage=en
```
Returns `{ locality, city, principalSubdivision, countryName, ... }`. We try `locality` first (most specific — returned "Bellevue" for the test coords), then fall back through the others. Open-Meteo's own reverse-geocoding endpoint does **not** exist (404s) — don't try to use it.

## Weather code mapping

WMO weather codes → glyph and flavor are in two functions: `codeGlyph(code, isNight)` and `conditionFlavor(code)`. Codes used:

| Range | Glyph | Flavor |
|---|---|---|
| 0–2 (clear/partly) | ☀ / ☾ | none |
| 3 (overcast) | ☁ | none |
| 45, 48 (fog) | ≋ | HIDDEN BY FOG |
| 51–57 (drizzle) | ‧ | MISTED OVER |
| 61–67, 80–82 (rain) | ☂ | DROWNED IN RAIN |
| 71–77, 85–86 (snow) | ❄ | BURIED IN SNOW |
| 95+ (thunder) | ⚡ | RIPPED BY THUNDER |

## Comparison thresholds

In `comparePhrase(deltaC, mode)`:

- `|Δ| < 1.5°C` → ABOUT THE SAME
- `< 4°C` → A LITTLE WARMER / A LITTLE COOLER
- `< 8°C` → WARMER / COOLER
- `≥ 8°C` → MUCH WARMER / MUCH COOLER

These are tuned for Celsius; if you ever want them to reflect Fahrenheit feel, the thresholds need a separate path. Right now the unit toggle only affects display, not phrasing.

Grammar fix: "ABOUT THE SAME than today" reads wrong, so for the tomorrow page when delta is small the suffix is rewritten to "as today".

## Files

- `lazy-weather.html` — single self-contained file using React 18 + Tailwind + Babel from CDNs. Open with `npx serve .` (NOT `file://` — geolocation will fail). This is what to use for casual local testing.
- `lazy-weather.jsx` — same component as a proper React module. Drop into a Vite + React + Tailwind project as `App.jsx`. This is the form to build on going forward.

## Known issues / TODOs

1. **Geolocation falling back to Bellevue.** When opened from `file://`, browsers block geolocation. When opened from a non-localhost `http://`, browsers also block it. Fix is to serve over `localhost` or `https://`. The fallback hint surfaces the real reason (`permission denied`, `file:// — needs http(s)`, `needs https`, `timeout`).
2. **Unit toggle doesn't persist.** Add `localStorage` once you move to a real Vite build. (Note: `localStorage` is not available in Anthropic-hosted artifacts but is fine in any normal deployment.)
3. **Phrasing thresholds are Celsius-only.** Whichever unit the user picks, the comparison still bins on Celsius deltas. Probably fine — the *feel* of "warmer" is similar across units — but worth knowing.
4. **No caching.** Every reload hits both APIs. Open-Meteo is generous but for a deployed app cache the response for ~30 minutes.
5. **The two-page swipe** uses native CSS `scroll-snap-type: x mandatory`. Works on mobile and desktop trackpads. No swipe library needed.
6. **Picked-city is not persisted.** Refreshing reverts to geolocation/Bellevue. `localStorage` would fix it.

## What was deliberately left out

- The "Lazy Weather is a self funded project … Support us!" panel from the original — removed at the user's request. Don't add it back.
- The tram / "2 stops" pill at the top of the screenshots — that's the user's iOS Live Activity from a separate app, not part of Lazy Weather.

## Running locally

```bash
# Quickest — single HTML file
npx serve .
# → open http://localhost:3000/lazy-weather.html

# Or in a Vite project
npm create vite@latest lazy-weather -- --template react
cd lazy-weather
npm install
# replace src/App.jsx with lazy-weather.jsx
# add Tailwind: https://tailwindcss.com/docs/guides/vite
npm run dev
```