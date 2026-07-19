GNSS-Radar
===============================================================================

Author
-------------------------------------------------------------------------------
Taro Suzuki
E-Mail: <gnsssdrlib@gmail.com>
HP: <https://www.taroz.net>

Overview
-------------------------------------------------------------------------------
"GNSS-Radar" is a web application to show the current GNSS constellation at a specified location.

2026 modernization pass
-------------------------------------------------------------------------------
The original 2014 version had stopped working (dead Google Maps script, a
static/stale 2014 TLE snapshot, HTTP-only asset URLs on an HTTPS page, and
synchronous XHR that most browsers now refuse to run). This version fixes all
of that:

* **Map**: Google Maps (needed an API key + billing, and the old `sensor=false`
  param is long dead) replaced with **Leaflet.js + OpenStreetMap** tiles — no
  API key required.
* **Live data**: on load, the page now does an async `fetch()` of CelesTrak's
  live GNSS group (`https://celestrak.org/NORAD/elements/gp.php?GROUP=gnss&FORMAT=tle`)
  instead of reading a bundled static file. If that fetch fails for any reason
  (offline, CORS, CelesTrak down), it automatically falls back to the bundled
  `gnssradar/celestrak.txt` snapshot **and says so on-screen** — it will never
  silently show stale data as if it were live.
* **PRN mapping is now dynamic, not a hand-maintained file.** CelesTrak embeds
  the PRN/slot directly in the satellite name for GPS, QZSS, SBAS and BeiDou
  (e.g. `GPS BIII-1 (PRN 04)`, `QZS-2 (QZSS/PRN 194)`, `BEIDOU-2 IGSO-3 (C08)`),
  so the page parses it straight out of the TLE text every load. This is what
  used to be `gnssradar/satlist.txt` — that file is no longer read and is kept
  in the repo only for historical reference. It also incidentally fixes the
  old "missing PRN 9" bug, since whatever CelesTrak publishes now just shows up.
* **Known limitation — GLONASS and Galileo**: CelesTrak's feed does *not*
  embed an official slot/PRN for these two (GLONASS shows up as bare
  `COSMOS ####`, Galileo as `GSAT0###`). Rather than ship a mapping that will
  silently drift out of date, the app assigns them a **stable display index**
  (sorted by NORAD catalog number, labelled "(display index)" in the marker
  tooltip) instead of an official slot number. If you need the true official
  GLONASS slot or Galileo E-number, cross-reference the NORAD ID shown in the
  tooltip against a current source such as the [IGS MGEX constellation
  table](https://igs.org/mgex/constellations/) and hardcode it — this is the
  same "tedious manual bookkeeping" tradeoff the old `satlist.txt` had, just
  narrowed down to 2 of 6 constellations instead of all 6.
* **HTTPS everywhere**, synchronous `XMLHttpRequest` replaced with
  `async`/`await` `fetch()`, and an unused jQuery `<script>` tag (it wasn't
  actually called anywhere in the code) was removed.
* Removed a dead `help` link that pointed to a `gnssradar_e.html` file that
  isn't in this repo.

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
* highcharts.js: https://www.highcharts.com/
* Sylvester.js: https://sylvester.jcoglan.com/
* Leaflet.js: https://leafletjs.com/
* Map tiles: © [OpenStreetMap](https://www.openstreetmap.org/copyright) contributors
* TLE (Two Line Element) is downloaded live from [CelesTrak](https://celestrak.org/).
