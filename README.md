# ✈️ ESP8266 Plane Spotter

A desktop gadget with a **tactical / military-style** interface that shows the
**aircraft currently flying closest to your home** on a 0.96" OLED, plus live
weather, an NTP clock and a bunch of nerdy stats. Live ADS-B data comes from
the [OpenSky Network](https://opensky-network.org/) API; weather from
[Open-Meteo](https://open-meteo.com/); routes/airlines from
[hexdb.io](https://hexdb.io/). 3D-printable desktop case included in
[`hardware/`](hardware/).

Default home location is the **centre of Pinerolo (TO), Italy** — change it in
`config.h` to point at your own roof.

---

## What it shows

Every screen has a tactical header: title + page index on the left, a live NTP
clock on the right. It cycles through five screens every 7 seconds:

1. **TARGET** — nearest aircraft: callsign, a type icon, distance + compass
   bearing, altitude (m and flight level), a heading arrow, ground speed and a
   vertical altitude gauge.
2. **INTEL** — callsign, **airline**, **departure → arrival** route and an
   **ETA** estimate (looked up from hexdb.io), plus ICAO24 and heading.
3. **RADAR** — North-up PPI: home at the centre, range rings (outer = 120 km),
   a rotating sweep and a blip per aircraft. Blips are **dead-reckoned** from
   track + speed so they creep in real time between refreshes, with radar
   **persistence** (bright as the sweep passes, then a faint dot). The nearest
   is boxed; a tick marks the wall direction. Side panel: type, callsign, RNG,
   BRG, contact count.
4. **WX** — current weather (temperature, condition icon, humidity, wind) and a
   minimal next-hours forecast strip.
5. **SYSTEM** — big `HH:MM:SS` clock with live milliseconds, date, uptime,
   target count, WiFi signal bars, free RAM and request counters.

The **type icon** uses the OpenSky emitter category (`extended=1`) when present;
since that is often unset, the firmware otherwise estimates the type from
altitude + speed and flags it with a leading `~`.

Aircraft data refreshes every 60 s, weather every 10 min (both configurable —
see the rate-limit note below).

---

## Hardware

| Part | Notes |
|------|-------|
| ESP8266 board | NodeMCU v2/v3, Wemos D1 mini, or similar |
| 0.96" OLED, **SSD1306**, **4-wire SPI** | 7-pin module: `GND VCC SCK SDA RES DC CS` |

### Wiring

Hardware SPI is used, so `SCK` and `SDA` are fixed; the other three pins are
configurable in `config.h`.

| OLED pin | ESP8266 (NodeMCU label / GPIO) | Role |
|----------|-------------------------------|------|
| GND | GND | Ground |
| VCC | 3V3 | Power |
| SCK | **D5 / GPIO14** | HW SPI clock (fixed) |
| SDA | **D7 / GPIO13** | HW SPI data / MOSI (fixed) |
| RES | D0 / GPIO16 | Reset |
| DC  | D2 / GPIO4  | Data/Command |
| CS  | D1 / GPIO5  | Chip select |

> The default pins avoid the ESP8266 boot-strapping pins (GPIO0/2/15), so the
> board flashes and boots reliably.

### 3D-printed case

A simple two-part desktop case lives in [`hardware/`](hardware/):

| File | Part |
|------|------|
| [`hardware/Radar1.stl`](hardware/Radar1.stl) | Case part 1 |
| [`hardware/radar2.stl`](hardware/radar2.stl) | Case part 2 |

Print both, slot the board + OLED inside, and stand it on your desk. It is
designed to hang on a wall facing a known compass heading — set that heading in
`WALL_HEADING_DEG` so the radar's wall tick lines up.

```
        ESP8266 (NodeMCU)                OLED SSD1306 SPI
      ┌───────────────────┐            ┌──────────────────┐
      │ 3V3 ──────────────┼────────────┤ VCC              │
      │ GND ──────────────┼────────────┤ GND              │
      │ D5/GPIO14 ────────┼────────────┤ SCK              │
      │ D7/GPIO13 ────────┼────────────┤ SDA (MOSI)       │
      │ D0/GPIO16 ────────┼────────────┤ RES              │
      │ D2/GPIO4  ────────┼────────────┤ DC               │
      │ D1/GPIO5  ────────┼────────────┤ CS               │
      └───────────────────┘            └──────────────────┘
```

---

## Build & flash

### Option A — Arduino IDE

1. Install the **ESP8266 board package** (Boards Manager → "esp8266").
2. Install libraries via **Library Manager**:
   - `U8g2` by olikraus
   - `ArduinoJson` by Benoit Blanchon (v7.x)
3. Copy `firmware/plane_spotter/config.example.h` → `config.h` and fill in your
   WiFi credentials (and your coordinates, if not Pinerolo).
4. Open `firmware/plane_spotter/plane_spotter.ino`, select your board, and
   upload.

### Option B — PlatformIO

```bash
cd firmware
# copy and edit the config first:
cp plane_spotter/config.example.h plane_spotter/config.h
pio run -t upload
pio device monitor
```

`platformio.ini` already pins the board, libraries, and 115200 monitor speed.

---

## Configuration

All settings live in `config.h` (git-ignored so your WiFi password stays
private). Key options:

| Define | Meaning |
|--------|---------|
| `WIFI_SSID` / `WIFI_PASS` | Your network |
| `HOME_LAT` / `HOME_LON` | Your home coordinates (default: Pinerolo) |
| `SEARCH_RADIUS_DEG` | Half-size of the sky box to query (~1.0° ≈ 111 km) |
| `UPDATE_INTERVAL_MS` | Poll period for OpenSky |
| `WALL_HEADING_DEG` | Compass heading the wall/device faces (used by the radar wall tick) |
| `TIMEZONE` | POSIX timezone string for the NTP clock (default: Europe/Rome) |
| `WEATHER_INTERVAL_MS` | Weather refresh period (default: 10 min) |
| `OPENSKY_CLIENT_ID` / `OPENSKY_CLIENT_SECRET` | Optional OpenSky OAuth2 client for a much bigger budget |

### Data sources

| What | Service | Key needed |
|------|---------|------------|
| Aircraft (ADS-B) | OpenSky Network | optional (OAuth2 client → bigger quota) |
| Weather + forecast | Open-Meteo | no |
| Airline / route / ETA | hexdb.io | no |
| Clock | NTP (`pool.ntp.org`) | no |

> Route/ETA come from a community database, so they aren't available for every
> flight — `INTEL` shows `unknown` / `ETA --` when a route isn't found.

---

## Notes on the OpenSky API

- Anonymous access has a **small daily request budget (~400 calls)**, shared
  per public IP. Polling every 60 s is ~1440 calls/day, so anonymously you
  *will* hit `HTTP 429 Too many requests` and the display freezes on the last
  data (the firmware keeps showing the last known aircraft when rate-limited).
- **Recommended: use an OpenSky account.** Create one, go to *Account → API
  clients*, create a client, and put the `clientId`/`clientSecret` into
  `OPENSKY_CLIENT_ID`/`OPENSKY_CLIENT_SECRET`. The firmware then does the OAuth2
  `client_credentials` flow (token from `auth.opensky-network.org`, sent as a
  `Bearer` header, auto-refreshed) and gets a far larger quota — 60 s polling
  works comfortably. Anonymous (no client) still works if you raise
  `UPDATE_INTERVAL_MS` to ~300000 (5 min).
- **Aircraft-type icons:** OpenSky's emitter category (state index 17, via
  `extended=1`) is frequently `0` (unknown) in practice, so when it is missing
  the firmware estimates the type from altitude + speed and flags it with a
  leading `~` (e.g. `~Small`). Real category data, when present, is used as-is.
- Coverage is community ADS-B, so very low / very local traffic may not always
  appear. A larger `SEARCH_RADIUS_DEG` finds more planes but uses more RAM to
  parse (the ESP8266 only has ~40 KB free heap, so don't go wild).
- TLS certificate validation is skipped (`setInsecure()`) to keep memory low;
  fine for read-only public data.

---

## License

MIT — see [LICENSE](LICENSE).
