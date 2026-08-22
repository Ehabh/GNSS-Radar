GNSS-Radar -- Preview Available Here: https://ehabh.github.io/GNSS-Radar/
===============================================================================

Author (original project)
-------------------------------------------------------------------------------
Taro Suzuki
E-Mail: <gnsssdrlib@gmail.com>
HP: <https://www.taroz.net>
Original repo: <https://github.com/taroz/GNSS-Radar>

Overview
-------------------------------------------------------------------------------
"GNSS-Radar" is a web application to show the current GNSS constellation at a
specified location.

This fork/branch modernizes the original 2014 codebase, which had stopped
working entirely by 2026, and adds a few new features on top. Everything
below is organized so it's easy to diff against the original repo: what was
broken and got fixed, what regressed along the way and got caught, and what's
genuinely new.

-------------------------------------------------------------------------------
## 1. Fixes - the app was completely broken, now it isn't

| # | Problem in the original | What was wrong | The fix |
|---|---|---|---|
| 1 | Stale TLE data | `gnssradar/celestrak.txt` was a static file frozen in **2014** - every satellite position was 12 years out of date | Page now does an async `fetch()` of CelesTrak's **live** GNSS group on every load (`https://celestrak.org/NORAD/elements/gp.php?GROUP=gnss&FORMAT=tle`). If that fetch fails for any reason (offline, CelesTrak down, CORS), it automatically falls back to the bundled 2014 snapshot **and says so on-screen** via the status bar - it never silently shows stale data as if it were live. |
| 2 | Broken map | Google Maps script had no API key (`sensor=false`, a parameter Google retired years ago) - the map never loaded | Replaced with **Leaflet.js + OpenStreetMap** tiles. No API key, no billing account needed. |
| 3 | Mixed content | Page is served over `https://` but pulled some resources over plain `http://` - blocked/warned by modern browsers | Every remaining `http://` reference changed to `https://`. *(The Google Maps and jQuery `http://` lines from the original are not just "fixed" here - they no longer exist in the app at all, superseded by fix #2 and the cleanup below. The Twitter meta image tag is the one `http://→https://` change still actually present in the code today.)* |
| 4 | Synchronous XHR | Used a blocking, synchronous `XMLHttpRequest` to read local files - most browsers now refuse this outright, and it only worked from a real web server anyway | Rewritten as `async`/`await` `fetch()`, with the whole init sequence restructured to run after data actually loads. |
| 5 | Missing PRN 9 | `gnssradar/satlist.txt` was a hand-maintained PRN -> NORAD ID lookup table that had gaps (PRN 9 was simply never added) and would only ever get more stale | **`satlist.txt` is no longer used at all.** CelesTrak embeds the PRN/slot directly in the satellite name for GPS, QZSS, SBAS and BeiDou (e.g. `GPS BIII-1 (PRN 04)`, `QZS-2 (QZSS/PRN 194)`, `BEIDOU-2 IGSO-3 (C08)`), so the PRN is now parsed straight out of live TLE text on every load. Whatever CelesTrak publishes just shows up - including PRN 9, and anything added after this project's shelf life. `satlist.txt` is kept in the repo only for historical reference. |

**Known limitation carried over from fix #5:** CelesTrak's feed does *not*
embed an official slot/PRN for GLONASS (`COSMOS ####`) or Galileo
(`GSAT0###`). Rather than ship a mapping that will silently drift out of date
again, those two constellations get a **stable display index** instead
(sorted by NORAD catalog number, labeled "display index, see readme" in the
marker tooltip). If you need the true official GLONASS slot or Galileo
E-number, cross-reference the NORAD ID shown in the tooltip against a current
source such as the [IGS MGEX constellation table](https://igs.org/mgex/constellations/)
and hardcode it - the same "tedious manual bookkeeping" the old `satlist.txt`
had, just narrowed from 6 constellations down to 2.

Also cleaned up along the way, found while fixing the above:
* Removed an unused jQuery `<script>` tag - the app's own code never called `$()` anywhere (confirmed by search), so it was dead weight.
* Removed a dead "help" link pointing to `gnssradar_e.html`, a file that was never committed to this repo.

-------------------------------------------------------------------------------
## 2. A regression caught during testing (and fixed)

Removing jQuery in fix-cleanup above (Section 1) broke something non-obvious:
the bundled `highcharts.js` (v4.0.1, from 2014) silently depends on jQuery at
load time to build its internal `HighchartsAdapter` (used for its own event
handling). Without it, chart creation threw `TypeError: K is not a function`
the moment the sky plot tried to render - `K` being Highcharts' own minified
name for its `addEvent` utility, which was `undefined` with no adapter
present.

**Fix:** load `gnssradar/lib/highchartsv4/adapters/standalone-framework.js`
(already present in the repo, just never wired up) *before* `highcharts.js`.
It builds the same adapter Highcharts needs, without requiring jQuery at all.
Reproduced the crash and confirmed the fix in a headless DOM before shipping it.

-------------------------------------------------------------------------------
## 3. New features (not in the original app at all)

* **Small colored PRN badge markers.** The original's map markers were large
  illustrated PNG icons (up to 110x85px) - fine for a handful of satellites,
  unreadable once 100+ are on screen at once. Replaced with small (24x18px)
  colored badges showing the PRN number directly, colored by constellation.
* **Satellite orbit ground tracks.** Hover (or tap) a satellite to preview its
  orbit; click to "pin" it (click again, or pick a different satellite, to
  switch). The track is centered on the satellite's current displayed
  position - half an orbital period before it, half after - so it passes
  through the marker and spans the visible range on both sides, rather than
  only showing where the satellite is *about* to go. Only one satellite's
  orbit is shown at a time by design: drawing all 100+ simultaneously was
  tried and is unreadable clutter.
* **Constellation toggle panel.** A checkbox panel in the map's top-right
  corner lets you show/hide each constellation directly, with a live
  satellite count per system. It stays in sync with the existing sky-plot
  legend (in the bottom-right chart) - toggling either one updates the other.

-------------------------------------------------------------------------------
## 4. Security hardening

A review turned up one real issue and one gap, both now fixed:

* **Stored XSS via satellite names (fixed).** Satellite marker tooltips were
  built by concatenating `sat.name` - text that comes from the **live
  CelesTrak feed**, an external, unauthenticated source - into a string
  passed to Leaflet's `bindTooltip()`. Leaflet renders string tooltip
  content as raw HTML by design (this is documented Leaflet behavior, not a
  bug in Leaflet - see their reference docs and the recent CVE-2025-69993
  advisory for the identical `bindPopup()` case). In practice CelesTrak is a
  reputable, curated source, so the realistic risk was low, but the app was
  rendering third-party network data as HTML with no sanitization - the
  textbook setup for this bug class. **Fix:** tooltip content is now built as
  real DOM nodes with `.textContent` (`satTooltipContent()` in
  `GNSS-Radar.html`), which Leaflet's own docs recommend for exactly this
  situation - whatever CelesTrak sends can never be interpreted as markup.
* **Content-Security-Policy (added).** The page previously had no CSP at
  all. Added one as a `<meta>` tag, allow-listing only the origins the page
  actually uses (`unpkg.com`, `celestrak.org`, OpenStreetMap tiles).
  Two things worth knowing about it:
    * `script-src` includes `'unsafe-inline'` because the whole app is one
      inline `<script>` block rather than an external `.js` file - so this
      CSP does **not** block inline event-handler-based script execution. A
      stricter policy would need the script moved to its own file with a
      nonce/hash, which hasn't been done.
    * `script-src` also includes `'unsafe-eval'`, required because
      `gnssradar/lib/sylvester/sylvester.js` ships in the old "packer"
      format and self-decompresses via `eval()` at load time - confirmed
      it's the only bundled library that needs this. That's vendored code
      we ship, not attacker-influenced data, so it's a different trust
      boundary from the XSS fix above and doesn't reopen it.
    * `frame-ancestors` was deliberately **left out**: it's silently
      ignored when a CSP is delivered via `<meta>` (only an HTTP response
      header enforces it), so including it would just be dead weight that
      looks like protection it isn't. Real clickjacking protection needs a
      server-level header - not possible with plain
      `python -m http.server`, and out of scope for a static-file repo.

-------------------------------------------------------------------------------
Running it
-------------------------------------------------------------------------------
Like the original, this must be served over HTTP(S), not opened as a
`file://` URL (browsers block `fetch()` from `file://`). From this folder:

```
python3 -m http.server 8000
```

then open <http://localhost:8000/GNSS-Radar.html>.

Options
-------------------------------------------------------------------------------
* Set the observer location by latitude and longitude (the unit is degree).
    * URL?lat=xxx&lon=xxx (default: lat=35.7&lon=139.8 (Tokyo))
    * e.g. `GNSS-Radar.html?lat=-37.8&lon=145`

* Set the elevation mask angle when computing the sky plot (the unit is degree).
    * URL?elemask=xxx (default: elemask=10)
    * e.g. `GNSS-Radar.html?elemask=45`

* Set the time offset when computing the sky plot (the unit is hour).
    * URL?offhr=xxx (default: offhr=0)
    * e.g. `GNSS-Radar.html?offhr=12`

* Set the time interval when computing the sky plot (the unit is minutes).
    * URL?tint=xxx (default: tint=30)
    * e.g. `GNSS-Radar.html?tint=5`

* Set the number of times when computing the sky plot.
    * URL?ntimes=xxx (default: ntimes=24, 24*30min=12hour)
    * e.g. `GNSS-Radar.html?ntimes=48`

* These options start with "?" and can be combined with "&".

Acknowledgments
-------------------------------------------------------------------------------
"GNSS-Radar" uses the following libraries:

* satellite.js: https://github.com/shashwatak/satellite-js
* highcharts.js (+ standalone-framework.js adapter): https://www.highcharts.com/
* Sylvester.js: https://sylvester.jcoglan.com/
* Leaflet.js: https://leafletjs.com/
* Map tiles: © [OpenStreetMap](https://www.openstreetmap.org/copyright) contributors
* TLE (Two Line Element) is downloaded live from [CelesTrak](https://celestrak.org/).
