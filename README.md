# EQMap

Animated, time-lapse display of Japan Meteorological Agency (JMA) epicenter data on a map of Japan.

The authoritative database is SQLite on a Linux server, which **automatically ingests and accumulates JMA data once a day**.
The browser app reads from that server. It also works on its own without a server.

```
  JMA list.json ──once a day (systemd timer)──▶ Linux server
                                                 SQLite + HTTP API + app hosting
                                                        │
                                       Browser ◀────────┘  http://<server>:8787/
                                       (localStorage is an offline cache)
```

## Two ways to run it

### A. Server mode (recommended)

Just open `http://<server>:8787/` in a browser. The same server delivers both the app and the API,
so no CORS setup is needed. Data accumulates on the server, so **it keeps growing even when your PC is off**.

For setup, see [the steps below](#installing-on-a-linux-server) (they also serve as the server README).

### B. Standalone mode

Double-click `index.html` to open it. Without a server, the app reads JMA directly
and stores data in the browser's localStorage. No build or server required.

To view an existing server while opening the page via `file://`, add `?api=http://<server>:8787` to the URL
(it is remembered after the first time; start the server with `--cors`).

## Controls

| Control | Action |
|---|---|
| ▶ / ⏸ (Space key) | Play / Pause |
| Seek bar | Jump to any time |
| Speed | How many hours advance per real-time second |
| Range | Playback range as start/end dates (YYMMDD). Defaults to 3 months back from the latest stored date |
| Magnitude | Filter by minimum magnitude |
| Mouse wheel | Zoom (centered on the cursor) |
| Drag | Pan the map |
| Hover an epicenter | Location name, time, M, depth, max intensity |
| Fit all | Reset the map view |
| Refresh | Manually re-fetch from JMA |
| Export / Import | Save / restore stored data as JSON |

## Behavior

### On startup

1. Draws the map immediately from the localStorage cache (without waiting for the server or network)
2. Queries the data source in the background
   - Server mode: asks the server to check for updates with `POST /api/refresh`, then reloads with `GET /api/events`
   - Standalone mode: reads JMA directly and merges into localStorage
3. Playback does not start automatically. Press ▶ to start

The "Source" field in the top-left always shows whether you are looking at **Server / Cache / JMA direct**.
If the source can't be reached, the app keeps displaying the cached data.

### Data accumulation

JMA's public list only covers **roughly the last 30 days**. By continuously ingesting and merging it,
older periods remain available.

- **Server mode** — the server's SQLite is authoritative. A systemd timer updates it once a day.
  There is effectively no size limit; years of data can accumulate. With `Persistent=true`, any runs missed
  while the server was down are caught up after boot (the list's 30-day window means a few days of downtime loses nothing)
- **Standalone mode** — stored in localStorage. Limited to 30,000 events; the oldest are removed beyond that

> **Backup**
> In server mode, just copy `/var/lib/eqmap/eqmap.db`.
> In standalone mode, localStorage is wiped by the browser's "Clear browsing data",
> so occasionally save a JSON file with "Export".

> **Note on opening via `file://`**
> localStorage works in Chrome / Edge / Firefox, but it is **shared by all `file://` pages**.

## Data source

[JMA Earthquake Information (multilingual)](https://www.data.jma.go.jp/multi/quake/index.html?lang=jp)

The app fetches the following JSON, which that page uses internally, directly.
It is served with `Access-Control-Allow-Origin: *`, so browsers can read it directly.

```
https://www.jma.go.jp/bosai/quake/data/list.json
```

### Ingestion processing

- **Deduplication** — the same earthquake (`eid`) receives multiple reports. Among reports with epicenter
  coordinates, the one with the latest `ctt` (creation time) is used
- **Seismic intensity bulletins excluded** — their `cod` (epicenter coordinates) is empty, so they can't be plotted
  and are dropped automatically
- **Two coordinate formats supported** — normally decimal degrees `+32.5+130.5-10000/`. Very rarely, degree-minute
  format `+3237.5+13040.7-16000/` (= 32°37.5′N / 130°40.7′E) appears, so out-of-range values are
  converted as degree-minutes
- **Missing depth** — some records lack the third component. Treated as `null` (unknown)
- **Distant earthquakes excluded** — the list also includes earthquakes in Indonesia, South America, etc.
  Anything outside longitude 120–156° / latitude 20–50° is not shown

## Display

- **Circle size = magnitude** (`1.6 × 1.45^(M-2)` px)
- **Flashes on occurrence and leaves an afterglow** — a ripple expands, the core fades, and a faint trail remains
- **Trails cover only 3 months back from the displayed time** — older epicenters disappear (`TRAIL_DAYS = 90`).
  Keeping the entire period fills the screen and makes recent activity unreadable. Tooltip
  hit-testing uses the same range
- Effect durations are defined in real time (ripple 0.9 s, afterglow 5 s), so **changing playback speed
  doesn't change how they look**

## Files

```
index.html                     The app (authoritative). Single file containing HTML / CSS / JS / map data
server/
  eqmap.py                     Server. Fetching + SQLite + HTTP API + app hosting in one file
  install.sh                   Linux install script (idempotent)
  web/index.html               Copy of index.html for serving
  systemd/eqmap.service         Resident HTTP server unit
  systemd/eqmap-update.service  Oneshot unit that runs a single fetch
  systemd/eqmap-update.timer    Timer that triggers the above once a day
tools/make_map_data.py         Script that generates the bundled map data
tools/japan.geo.js             Generated map data (inlined into index.html)
```

After editing `index.html`, copy it to `server/web/index.html` before deploying.

## Installing on a Linux server

Requires Python 3.8 or later. No pip installs needed.

```sh
scp -r server/ user@server:/tmp/eqmap-server
ssh user@server 'sudo sh /tmp/eqmap-server/install.sh'
```

What `install.sh` does:

1. Creates the system user `eqmap`
2. Puts the code in `/opt/eqmap` and the DB in `/var/lib/eqmap`
3. Performs the initial fetch from JMA to create the DB
4. Installs the systemd units and enables `eqmap.service` (HTTP) and `eqmap-update.timer` (once a day)
   (without systemd, it installs `/etc/cron.d/eqmap` instead)

It is idempotent and can be run any number of times. The DB is preserved.

### Operations

```sh
sudo systemctl status eqmap                     # server status
sudo systemctl list-timers eqmap-update.timer   # when the next automatic update runs
sudo journalctl -u eqmap-update -n 50           # update history
sudo systemctl start eqmap-update               # run one update now

sudo -u eqmap python3 /opt/eqmap/eqmap.py --db /var/lib/eqmap/eqmap.db stats
sudo -u eqmap python3 /opt/eqmap/eqmap.py --db /var/lib/eqmap/eqmap.db import export.json
```

### Changing the update schedule

Edit `OnCalendar` in `/etc/systemd/system/eqmap-update.timer`, then run
`sudo systemctl daemon-reload && sudo systemctl restart eqmap-update.timer`.
Times are interpreted in the server's local time zone (check with `timedatectl`).

### API

| Endpoint | Description |
|---|---|
| `GET /` | The app |
| `GET /api/events` | All events. Filter with `from` / `to` (epoch ms), `min_mag`, `limit` |
| `GET /api/status` | Count, span, last fetch, recent fetch log |
| `POST /api/refresh` | Fetch from JMA now. Repeated requests within 60 seconds get 429 |

`/api/events` returns arrays of `[eid, t, lat, lon, depth, mag, maxi, name]`. Supports gzip;
710 events are about 11 KB.

### Regenerating the map data

Japan is extracted from [Natural Earth](https://www.naturalearthdata.com/) (Public Domain) 10m admin_0
and simplified with Douglas–Peucker. Runs with only the Python standard library.

```sh
python tools/make_map_data.py --tol 0.005
```

The generated `tools/japan.geo.js` is a single line: `const JAPAN_GEO=[...];`.
Replace the same line in `index.html` with its contents.

Raising `--tol` makes it coarser and smaller (default 0.01; the current bundled data is 0.005 = 61 KB / 3721 points).

The source data (13 MB) is cached at `tools/_ne_10m_admin_0_countries.geojson`.
If deleted, it is re-downloaded on the next run, so it doesn't need to be in the repository (already in `.gitignore`).

## Limitations

- JMA's JSON is not an officially documented API, so its format may change
- Earthquakes for which only a seismic intensity bulletin was issued have no confirmed epicenter and are not shown
- Lakes are cut out as holes with even-odd filling, but small islands and inlets are omitted due to simplification
- Epicenter location names are shown in Japanese, as provided by JMA
