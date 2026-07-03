# Kitchen Knob — CrowPanel 1.28" Music + Weather Controller

An ESPHome firmware for the **Elecrow CrowPanel 1.28" round ESP32-S3 rotary display** that turns it into a local, always-on control puck for Home Assistant:

- **Music** — Spotify via Music Assistant to a cast speaker: volume, play/pause, next / restart-or-previous, album art, on-device playlist browser.
- **Weather** — a five-view panel: current conditions (with MDI condition icon), wind, air, a lightning direction compass, and live NWS radar.
- **Ambient** — the WS2812 ring reacts to nearby lightning; the screen sleeps when idle and wakes on interaction.

All UI and input handling runs on-device (LVGL); Home Assistant only feeds data and executes service calls. Nothing round-trips per interaction, so volume/scroll stay instant.

> Built for a specific kitchen setup — treat entity IDs, units, and the weather source as things you'll repoint. Everything tunable lives in `substitutions:` at the top of `TheKnob.yaml`.

---

## Hardware

**Elecrow CrowPanel 1.28" HMI ESP32-S3 Rotary Display** (ESP32-S3-N16R8: 16 MB flash, 8 MB octal PSRAM, GC9A01A 240×240 round LCD, CST816 touch, rotary encoder + push, 5× WS2812).

### Pinout (as used here)

| Function | GPIO | Notes |
|---|---|---|
| LCD SCLK / MOSI | 10 / 11 | SPI |
| LCD DC / CS / RST | 3 / 9 / 14 | |
| **LCD power rails** | 1, 2 | driven HIGH in `on_boot` |
| **Power gate** | 40 | driven **LOW** in `on_boot` — miss this and the panel stays black |
| Backlight | 46 | LEDC PWM |
| Touch SDA / SCL | 6 / 7 | CST816 @ `0x15` |
| Touch INT / RST | 5 / 13 | |
| WS2812 ring | 48 | 5 LEDs |
| Encoder A / B / SW | 45 / 42 / 41 | SW active-low |

---

## Controls

| Input | Now Playing | Playlist Browser | Weather Panel |
|---|---|---|---|
| **Rotate** | Volume | Scroll playlists | Cycle views |
| **Press** | Play/pause (or start default playlist if idle) | Select + play | Exit |
| **Double-click** | Open playlist browser | — | — |
| **Swipe L→R** | Next track | — | — |
| **Swipe R→L** | Restart (or previous if <10 s in) | — | — |
| **Double-tap screen** | Open weather panel | — | — |

Weather views cycle **Now → Wind → Air → Lightning → Radar** and auto-exit after 10 s (10 min on Lightning/Radar). The screen dark-sleeps after 2 min idle unless music is playing or a weather view is up.

---

## Prerequisites (Home Assistant)

- **Music Assistant** with a player entity (e.g. a Google Nest Mini via cast) and Spotify configured (Premium required for playback control).
- **"Allow the device to perform Home Assistant actions"** enabled for this device: *Settings → Devices & Services → ESPHome → (device) → Configure*. Off by default; nothing outbound works without it.
- A **weather entity** (this build uses `weather.pirateweather`) for feels-like, dew point, pressure, condition, and forecast high/low.
- *(Optional)* **local weather-station sensors** for temperature, humidity, wind speed/gust/direction — repoint or fall back to the weather entity's attributes.
- **Blitzortung** integration for lightning azimuth + distance.
- **Template sensors** for the playlist list and forecast high/low — see [`ha-knob-playlists.yaml`](ha-knob-playlists.yaml). Grab your Music Assistant `config_entry_id` from *Developer Tools → Actions → `music_assistant.get_library`*.
- **Radar camera** — a **Generic Camera** whose Still Image URL is your NWS station GIF wrapped through an image proxy so HA holds a JPEG (the on-device decoder can't do GIF, and `camera_proxy` won't transcode it):
  ```
  https://images.weserv.nl/?url=radar.weather.gov/ridge/standard/KIWX_0.gif&output=jpg&w=480
  ```
  (`KIWX` = your NEXRAD station.)

## Prerequisites (device)

- **ESPHome 2026.6+**.
- `materialdesignicons-webfont.ttf` in your ESPHome `config/fonts/` directory (weather condition glyphs). Grab it from the [Pictogrammers MDI](https://github.com/Templarian/MaterialDesign-Webfont) repo.
- `secrets.yaml`: `wifi_ssid`, `wifi_password`, `api_key`.
- Montserrat is pulled automatically via `gfonts://` (needs internet at compile).

---

## Setup

1. Add the template sensors from `ha-knob-playlists.yaml` to HA (set your `config_entry_id` and radar/weather entities). Restart HA; confirm `sensor.knob_playlists` has a `names` attribute and `sensor.knob_weather_hilo` a `hilo` attribute.
2. Create the Generic Camera (above) and note its **entity_id** (HA may prefix `camera_`).
3. Drop the MDI TTF into `config/fonts/`.
4. Fill in the `substitutions:` block in `TheKnob.yaml`.
5. First flash over USB (the S3 native-USB re-enumerates on bootloader entry — if a serial flash fails, hold BOOT / tap RESET / release BOOT and retry). OTA thereafter.

### Substitutions

| Key | Meaning |
|---|---|
| `ma_player` | Music Assistant player entity |
| `ha_base` | HA base URL, no trailing slash |
| `spotify_default_uri` | Playlist **name** to play when idle (MA takes a library name, not a Spotify URI) |
| `weather_entity` | Weather entity for feels/dew/pressure/condition/hi-lo |
| `radar_camera` | Generic Camera entity id |
| `ltng_near_mi` / `ltng_mid_mi` | Ring turns red / orange within these miles |
| `ltng_stale_s` | Strike considered stale after N seconds (compass clears, ring reverts) |

Local weather-station entities are hardcoded in the `sensor:` block (`sensor.crystal_lake_weather_*`) — repoint to yours or swap back to `weather_entity` attributes.

---

## Gotchas & lessons (CrowPanel 1.28" specific)

The board is finicky to bring up. What actually works:

- **Display driver: `ili9xxx` (model `GC9A01A`), not `mipi_spi`.** `mipi_spi` mangles color on this panel (byte/channel ordering); `ili9xxx` + `invert_colors: true` renders correctly. Yes, `ili9xxx` is deprecation-warned — ignore it, it's the one that works.
- **Power sequence is the black-screen fix.** In `on_boot`: backlight on → **GPIO40 LOW** → GPIO1/GPIO2 HIGH. GPIO40 is the non-obvious one.
- **`update_interval: 10ms`** on the display (LVGL drives it) — not `never`, which starves the panel here.
- **CST816 touch: `skip_probe: true`.** The chip sleeps after ~2 s and won't answer a chip-ID read, so ESPHome marks it failed on boot. `skip_probe` skips the read.
- **Never call `lv_*` / `lv_refr_now()` from encoder/button callbacks.** LVGL reentrancy → reboot loop. Mutate a global in the callback; let a timer/`lvgl.*.update` action repaint.
- **LVGL meter `rotation` is broken** in current ESPHome (renders wrong) — leave `rotation: 0` and pre-rotate the value instead (compass needle = `(azimuth + 270) % 360`).
- **`music_assistant.play_media` wants a library name**, not `spotify:playlist:...`. Test the exact string in Developer Tools first.
- **HA sensor state caps at 255 chars; attributes don't.** The playlist list rides in an attribute, not state.
- **`camera_proxy` won't transcode a GIF** — hence the weserv proxy so HA stores a JPEG.
- **Color/number literals in LVGL** (`bg_color`, `seek_position`, etc.) must be quoted strings.
- Cast/Spotify-Connect: if you start playback by casting *from the phone*, the phone owns the session and the knob can't set volume. Start via the knob (MA owns the stream) and control works.

---

## Files

| File | Purpose |
|---|---|
| `TheKnob.yaml` | ESPHome device config |
| `ha-knob-playlists.yaml` | HA template sensors (playlist list + forecast hi/lo) |

---

## Credits

- [tobsch/nanopod](https://github.com/tobsch/nanopod) — reference for the same board driving Music Assistant; the crash-free encoder pattern and `skip_probe` came from here.
- Elecrow's LVGL reference config for the 1.28" board (power sequence, `ili9xxx`+invert).
- [ESPHome LVGL cookbook](https://esphome.io/cookbook/lvgl/) — MDI font + meter needle.

## License

MIT — do what you like.
