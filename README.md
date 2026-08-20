# Keep Grinding and Nobody Dies (I Guess)

> 🏆 **1st place overall at the Horizons Arcana Hackathon.**
>
> _Built over a weekend, against a field of around 100+ people._


<img width="4096" height="2304" alt="image" src="https://github.com/user-attachments/assets/637ccfd6-f8fb-472e-940d-2a76ebcc33cb" />



<!-- [ Watch a full run](PASTE_VIDEO_LINK) -->

A physical bomb defusal party game in the spirit of _Keep Talking and Nobody Explodes_. One player (the **defuser**) stands alone at a real hardware bomb with buttons, wires and a countdown display, and sees only what's in front of them. Everyone else is the **expert**: they hold the rulebook, they never see the bomb, and they have to talk the defuser through it before the timer expires.

The game lives in the asymmetry between those two views. Neither side can act on what the other knows without saying it out loud, so the real interface is the conversation. Sabotage events run in parallel and degrade the defuser's screen while they work: it inverts, the text scrambles, a fake kernel panic covers the display.

---

## How it's built

The device is intentionally minimal. All authoritative state lives on the laptop.

```
  ESP32 (buttons, wires, 7-segment)
        │  USB serial, JSON at 115200 baud
        ▼
  Rust / Tauri backend  ──► native window   = the EXPERT's rulebook
        │
        └─ HTTP relay on :4000 ──► tablet browser = the DEFUSER's bomb screen
                                └─► /admin        = the operator panel
```

- **The firmware** (`keep_grinding_ful_game.ino`) is strictly an I/O bridge. It debounces the 4 buttons, emits press and release edges stamped with its own `millis()`, samples the 4 wires, and drives the 7-segment display on command. No game logic runs on the device, which makes the firmware effectively write once.
- **The Rust backend** owns the countdown, strikes, the four module state machines, and the sabotage scheduler.
- **Two screens, deliberately.** The laptop window exposes the timer, strike count, active module and the manual, and nothing about the bomb's actual state. The defuser's tablet loads a separate fullscreen page over the LAN. Neither view is solvable alone, which is the constraint the whole design rests on.

## Design notes

The decisions the rest of the project hangs off, written down because they're the ones that would have been expensive to change later.

**The device holds no game state.** The ESP32 reports inputs and renders a number. Rules, timer, strikes and sabotage all live on the host. The payoff shows up under time pressure: Color Memory was redesigned end to end without the firmware changing at all, and every balance number sits in `game_config.toml` rather than in a sketch that needs reflashing to tune.

**Input edges carry device timestamps.** The host polls at roughly 50ms, so timing a button hold host side would fold poll jitter straight into the measurement. Instead the ESP32 stamps every press and release with its own `millis()` and the host derives duration from those, which keeps TAP versus HOLD exact no matter what the transport does. It's also the one decision here that couldn't have been retrofitted, since it's part of the wire format.

**Snapshots merge rather than queue.** When several lines arrive between ticks, levels are last write wins and edges are concatenated in order. A slow tick never drops a press and never acts on a stale wire state.

**Neither screen is solvable alone.** The expert window deliberately withholds the serial, batteries, indicators, wire colors and alien phrase. That's enforced in the view layer instead of by agreement between players, so speech stays the only channel between them. Widening the expert view would quietly delete the game.

**One value, one source of truth.** The countdown on the tablet, the digits on the 7-segment, and The Button's release rule all resolve through the same function, so a rule can't disagree with what the player is looking at.

## The modules

Four of them, in randomised order each run. All four have to be solved before the clock reaches zero.

| Module           | The defuser sees                                             | The expert decodes                                                         |
| ---------------- | ------------------------------------------------------------ | -------------------------------------------------------------------------- |
| **Memory**       | A number and four labelled positions, reshuffled every stage | A 5 stage table keyed on the display and on positions pressed earlier      |
| **Color Memory** | Three colors flashing, then "your turn"                      | A flat color to button mapping (RED = Button 1, and so on)                 |
| **The Button**   | One lit button reading TAP or HOLD                           | Tap it quickly, or hold until READY appears and then release               |
| **Alien Wires**  | Four colored wires and an alien phrase with a CLASS          | Resolve the index word to a wire position, then apply ALPHA / BETA / OMEGA |

A mistake costs a strike and 20 seconds, but never progress inside a module. That policy is configurable via `mistake_behavior`; the default exists because forfeiting a five stage module to a single slip reliably kills a demo.

## Sabotage events

Eight of them fire at random during a run: dyslexia (scrambled text), upside down, mirror mode, a fake blue screen, static, siren lights, an "intrusion detected" shake, and a jumpscare. Blue screen and jumpscare also block input, so the defuser has to wait them out.

The interval is measured from the _end_ of the previous event rather than its start, with random jitter so the cadence stays unpredictable. Default is 45 seconds ±15, which is roughly four per four minute round.

## Operator panel

`http://localhost:4000/admin` on the host laptop. The break glass controls for keeping a live showcase on the rails:

- Force start, pause and resume, restart
- Nudge or set the timer, set the strike count
- Solve the current module, or jump to any module
- Fire any sabotage event manually, clear the active one, or disable random events entirely
- Force a DEFUSED or BOOM ending

It sits behind a random per run token printed to the terminal at startup. Opened on the host laptop, the token is injected automatically. From a phone, use the `?token=...` URL from that same output.

## Optional extras

- **Stimulation corner.** Muted, auto advancing clips in the corner of the defuser's screen, TikTok style, for audiences that drift. Paths go in `[stimulation]`.
- **Jumpscare videos.** Your own clips under `[events.jumpscare]`. Left empty, it falls back to a CSS monster face and a synthesized scream.

## The hardware

Every input is a plain GPIO read. No I²C, no analog, no shields.

### Parts

| Part                                     | Qty | Notes                                                                                                           |
| ---------------------------------------- | --- | --------------------------------------------------------------------------------------------------------------- |
| ESP32 dev board                          | 1   | Any board with a USB serial bridge (CP210x, CH340/CH9102, FTDI). The backend detects the port itself            |
| Momentary push buttons                   | 4   | Button 1 doubles as the game start trigger                                                                      |
| Removable "wires"                        | 4   | Blue, white, black, yellow. Any connector that separates cleanly: banana plugs, jumper pins, alligator clips    |
| 3 digit 7-segment display                | 1   | Driven with active low segment bytes, so common anode                                                           |
| Serial in latching shift register, 8 bit | 3   | 74HC595 or equivalent, chained. The firmware shifts three bytes MSB first, one per digit, then pulses the latch |
| USB cable                                | 1   | Data capable. It's the only link between bomb and laptop                                                        |

No external pull ups required; the buttons and wires use the ESP32's internal ones.

### Wiring

Every button and wire runs from its GPIO, through the component, to GND. Pressed or connected reads LOW.

| Signal            | GPIO | Notes                          |
| ----------------- | ---- | ------------------------------ |
| Button 1          | 14   | Also starts the game from idle |
| Button 2          | 27   |                                |
| Button 3          | 26   |                                |
| Button 4          | 33   |                                |
| Wire (blue)       | 25   | Wire 1 in the manual           |
| Wire (white)      | 13   | Wire 2                         |
| Wire (black)      | 16   | Wire 3                         |
| Wire (yellow)     | 17   | Wire 4                         |
| Display SDI / DIN | 32   |                                |
| Display SCLK      | 18   |                                |
| Display LOAD / CS | 19   |                                |

Flash `keep_grinding_ful_game.ino` once from the Arduino IDE with ESP32 board support installed. It shouldn't need reflashing after that, since all the game logic lives on the laptop.

## Running it

The full walkthrough (toolchain, flashing, connecting the tablet, firewall rules) is in **[guide.md](guide.md)**. Short version:

```bash
pnpm install
```

```bash
pnpm tauri dev
```

Plug the ESP32 in over USB and press physical Button 1 to start. The tablet joins at `http://<laptop-ip>:4000/` on the same Wi-Fi.

Tests:

```bash
cd src-tauri && cargo test
```

## Tuning

Every balance number lives in **`game_config.toml`**: timer length, allowed strikes, mistake penalty, tap versus hold threshold, mistake behaviour, event frequency, and per event durations. All of it is commented inline. Edit, save, restart, since the config is only read at startup. A malformed file won't crash the app; it falls back to defaults and reports it in the terminal.

## Project layout

| Path                         | What's in it                                            |
| ---------------------------- | ------------------------------------------------------- |
| `keep_grinding_ful_game.ino` | ESP32 firmware, I/O bridge only                         |
| `src-tauri/src/engine/`      | Game engine: modules, sabotage events, operator actions |
| `src-tauri/src/esp_io.rs`    | Serial link, port detection, reconnection               |
| `src-tauri/src/relay.rs`     | LAN HTTP server: defuser page, `/state`, `/admin`       |
| `src-tauri/src/tablet.html`  | The defuser's fullscreen bomb screen                    |
| `src-tauri/src/admin.html`   | The operator panel                                      |
| `src/`                       | The expert's React app (the Tauri window)               |
| `game_config.toml`           | All game balance settings                               |

## Tech stack

[Tauri](https://tauri.app/) (Rust backend plus a native window) with a [React](https://react.dev/) and [Vite](https://vite.dev/) frontend, talking to an ESP32 over USB serial.

## Team

Four of us built this over a weekend at the Horizons Arcana Hackathon.

|                                                         | Role                                                                                                                                                                                                                                                                                                                                                                                             |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Firdavs** ([@firancly](https://github.com/firancly))  | System architecture and the serial protocol. Set the split the project runs on: the ESP32 stays a dumb I/O bridge, the host holds all authoritative state, and edge events carry device timestamps so hold timing survives the poll interval. Designed the two screen information split and built the defuser, admin and expert pages. Code review, the presentation, and marketing with Daniel. |
| **Daniel** ([@keqw](https://github.com/ZTDaniel72))     | Firmware and hardware. Wrote the ESP32 code, did much of the breadboard work, and ran marketing with Firdavs.                                                                                                                                                                                                                                                                                    |
| **Kuzey** ([@CSDC-K](https://github.com/CSDC-K))        | Backend. Wrote most of the Rust: engine, modules and serial layer.                                                                                                                                                                                                                                                                                                                               |
| **Alimzhan** ([@zzertexx](https://github.com/zzertexx)) | Hardware. Wiring, the breadboard build and its styling, and validating the rig.                                                                                                                                                                                                                                                                                                                  |
