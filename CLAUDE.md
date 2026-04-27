# Lazy Weather

A web clone of the "Lazy Weather" mobile app — a minimalist forecast that compares today vs. yesterday and tomorrow vs. today in plain language ("ABOUT THE SAME as yesterday", "MUCH WARMER than today") instead of throwing numbers at you. Three-page horizontal pager: **Saved (left) → Today (default) → Tomorrow (right)**. The app loads on Today.

## Visual design

Match these constraints exactly. They're the whole point of the clone.

- **Background:** pure black (`#000`).
- **Font:** JetBrains Mono everywhere. Loaded from Google Fonts.
- **Palette:** white text, `#6b6b6b` for muted/secondary, `#1a1a1a` for the location pill, icon button background, and popover surfaces.
- **Location pill** in the top-left, with the city wrapped in literal asterisks: `*Bellevue*`. The asterisks are part of the brand voice — keep them.
- **Top-right:** a circular `…` icon button that opens a dropdown menu (units, refresh, about).
- **Headline type** is large (~26px), three lines: a normal-weight prefix, a muted ALL-CAPS emphasis line, and a normal-weight suffix.
- **Time-of-day rows** at the bottom of Today and Tomorrow: `Morning / Noon / Evening / Night`, each row showing temp, a glyph, the delta vs. the same hour yesterday/today, and an up/down arrow.
- **Page indicator:** three small dots at the bottom; active dot is `#e5e5e5`, inactive is `#3a3a3a`. Each dot is wrapped in an `h-11 px-1` button so vertical tap area is 44px while the visual spacing matches the original tight layout.
- **Tap targets:** every interactive element is ≥44×44px (iOS HIG). Inputs use `text-[16px]` so iOS Safari doesn't auto-zoom on focus. `button, a { -webkit-tap-highlight-color: transparent }` is set globally.

The aesthetic is deliberately spare — lots of empty vertical space, monospace, no rounded card stacks, no gradients. Resist any urge to "improve" it with shadows, color accents, or icons beyond the few unicode glyphs already in use.

## What the app does

### Saved page (swipe left from Today)
- Brand-voice headline: `Your saved / PLACES`.
- Inline `Search to add a place…` input that hits the same Open-Meteo geocoding endpoint as the location-pill search. Already-saved entries appear disabled with `· added` in the results.
- Each saved place renders a row showing `*City* · current temp · weather glyph`. Current weather is fetched lazily via Open-Meteo's `?current=temperature_2m,weathercode,is_day` and cached in component state, keyed by `lat/lon` (4 decimal precision).
- Tapping a row's city name **switches the active location** and smooth-scrolls back to Today.
- An **Edit** button toggles edit mode: rows reveal a `≡` drag handle on the left and a `×` on the right. **Done** exits. The toggle is hidden when the list is empty (and edit mode auto-exits if the last item is removed).
- Reorder uses **pointer events** with `setPointerCapture` (works on mouse and touch, no library). While a handle is held, the row under the pointer is found via `document.elementFromPoint` and the array is re-spliced live; the dragged row stays at 0.4 opacity for feedback.
- Persisted to `localStorage` under `lw-saved-locations` as `[{ city, lat, lon }, …]`.

### Today (default page)
- Headline compares **today's daily-average temp** to **yesterday's daily-average temp** and produces a phrase like "ABOUT THE SAME as yesterday" / "A LITTLE WARMER as yesterday" / "MUCH COOLER as yesterday".
- Four time-of-day rows (Morning 8am, Noon 12pm, Evening 6pm, Night 10pm) showing today's hourly temp at that hour, a weather glyph, and the delta vs. yesterday's same hour with an up/down arrow.
- The app opens here. A `useLayoutEffect` reads `today.offsetLeft` from a ref on the section and sets `scroller.scrollLeft` before paint, with a `requestAnimationFrame` fallback if layout wasn't measurable yet.

### Tomorrow
- Headline compares **tomorrow's daily-average temp** to **today's**. If tomorrow's weather code matches a known dramatic condition (fog, snow, rain, thunder), a "flavor" phrase overrides the temperature comparison: "HIDDEN BY FOG", "BURIED IN SNOW", "DROWNED IN RAIN", "RIPPED BY THUNDER", "MISTED OVER".
- The flavor headline is rendered with a small SVG dot-grid glyph between each word, mimicking the original's textured spacers.
- Same four time-of-day rows showing tomorrow's hourly temp at that hour, a glyph, and the delta vs. today's same hour.

### Menu (… icon)
- **Units:** Celsius / Fahrenheit. Component state, not persisted (still — see TODOs).
- **Refresh:** re-fetches forecast data for the active location.
- **About:** opens a modal crediting Taylor Brazelton (https://taylorbrazelton.com), naming the data sources (Open-Meteo for forecasts and city search, BigDataCloud for reverse geocoding), and linking the open-source repo (https://github.com/Ryanb58/lazyweather/).

`Use my location` lives **only** in the city-selector popover now; it was duplicated in the menu before and removed.

### Location pill (search)
The `*City*` pill is also a button. Tapping it opens a small search popover that hits Open-Meteo's geocoding endpoint (`https://geocoding-api.open-meteo.com/v1/search?name=...&count=5`). Picking a result sets `coords` + `city` directly (no reverse geocode needed — the geocoding payload includes the canonical name) and the weather effect refetches. Search debounces at 250ms and requires ≥2 characters.

When the input is empty, the popover shows a `+ Save *currentCity*` / `✓ Saved *currentCity*` toggle so the active location can be pinned without leaving Today. Below that is `Use my location`.

### Popover dismissal
Both the city-selector popover and the `…` menu auto-close when:
- The user swipes to a different page (handled inside `onScroll` once the page index changes), or
- A `mousedown`/`touchstart` lands outside the wrapper. A document listener checks `e.target.closest('[data-lw-search]')` / `[data-lw-menu]` and closes whichever popover the click missed.

## APIs

No keys required for any of these.

### Open-Meteo (forecast)
```
https://api.open-meteo.com/v1/forecast
  ?latitude={lat}&longitude={lon}
  &hourly=temperature_2m,weathercode
  &daily=weathercode,temperature_2m_max,temperature_2m_min
  &past_days=1&forecast_days=2
  &timezone=auto
```
With `past_days=1&forecast_days=2`, `daily.time` returns `[yesterday, today, tomorrow]` and `hourly.time` returns 72 hourly entries in the format `YYYY-MM-DDTHH:00`. The hourly index lookup uses exact string match on that format.

### Open-Meteo (current, for Saved-page rows)
```
https://api.open-meteo.com/v1/forecast
  ?latitude={lat}&longitude={lon}
  &current=temperature_2m,weathercode,is_day
  &timezone=auto
```
Lighter payload — used for each saved place to fill in temp and glyph.

### Open-Meteo (geocoding)
```
https://geocoding-api.open-meteo.com/v1/search?name={q}&count=5&language=en
```
Used by both the location-pill search and the Saved page's add-input.

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

- `docs/index.html` — single self-contained React 18 + Tailwind + Babel app. The deployed/canonical entry point (GitHub Pages serves from `/docs`). All component code lives inside one `<script type="text/babel">` block.
- `docs/manifest.webmanifest`, `docs/favicon.svg` — PWA assets. The install banner is shown on iOS/Android (and via `beforeinstallprompt` on Chromium) and is dismissable; dismissal is persisted in `localStorage` under `lw-install-dismissed`.

The earlier `lazy-weather.html` / `lazy-weather.jsx` files at the repo root are gone — `docs/index.html` is the only build target.

## Persistence (localStorage)

| Key | Shape | What writes it |
|---|---|---|
| `lw-saved-locations` | `[{ city, lat, lon }, …]` | the Saved page (add / remove / reorder) |
| `lw-install-dismissed` | `"1"` | the × on the install banner |

Saved-place current-weather is **not** persisted — it's re-fetched per session (and only for places not already cached in component state).

## Known issues / TODOs

1. **Geolocation falling back to Bellevue.** When opened from `file://`, browsers block geolocation. When opened from a non-localhost `http://`, browsers also block it. Fix is to serve over `localhost` or `https://`. The fallback hint surfaces the real reason (`permission denied`, `file:// — needs http(s)`, `needs https`, `timeout`).
2. **Unit toggle doesn't persist.** Easy `localStorage` add — we already use it for saved places and banner dismissal.
3. **Phrasing thresholds are Celsius-only.** Whichever unit the user picks, the comparison still bins on Celsius deltas. Probably fine — the *feel* of "warmer" is similar across units — but worth knowing.
4. **No caching.** Every reload hits Open-Meteo for the active location and once per saved place. Generous free tier but for a deployed app cache responses for ~30 minutes.
5. **The horizontal pager** uses native CSS `scroll-snap-type: x mandatory`. Works on mobile and desktop trackpads. No swipe library needed.
6. **Active picked-city isn't persisted across reloads.** Refresh reverts to geolocation / Bellevue. The Saved page partially papers over this (one tap to re-select), but the implicit "last viewed" location could be remembered too.
7. **Saved-place reorder uses pointer capture.** Pointer capture follows the DOM node by React key as the array reorders. If the dragged row ever feels like it loses capture mid-drag (e.g. on a browser that detaches the node during reconciliation), move the move/up handlers to the document instead of the handle.

## What was deliberately left out

- The "Lazy Weather is a self funded project … Support us!" panel from the original — removed at the user's request. Don't add it back.
- The tram / "2 stops" pill at the top of the screenshots — that's the user's iOS Live Activity from a separate app, not part of Lazy Weather.

## Running locally

```bash
npx serve docs
# → open http://localhost:3000
```

Geolocation needs `localhost` or `https://`. On `file://` it falls back to Bellevue and surfaces the reason in the muted hint under the location pill.
