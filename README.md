# hode-hd

**An HD-capable, scriptable fork of [`usineur/hode`](https://github.com/usineur/hode) — Gregory Montoir's reverse-engineered reimplementation of the engine used by *Heart of Darkness* (Amazing Studio, 1998).**

The original engine is faithfully preserved: same 12.5 Hz game tick, same `.lvl` / `.sss` / `.mst` / `.paf` data path, same per-level scripted callbacks. This fork adds a configurable presentation pipeline (HD upscaling with chained xBRZ + MLAA, 16:9 widescreen, 60 Hz interpolated render, persistent disk cache, sprite + cutscene prerender), a programmable Unix-socket automation API, and several menu / input / portability fixes.

```
$ ./hode --4k --hd-wide --smooth --hd-cache=./cache --prerender
HD mode: 15x scale (3840x2880), 16:9 widescreen, disk cache, prerender
prerender level 0 (rock) [################################] 5320/5320 (100%) 6.9s
```

---

## Table of contents

- [At a glance](#at-a-glance)
- [Building](#building)
  - [macOS (Apple Silicon & Intel)](#macos-apple-silicon--intel)
  - [Linux](#linux)
  - [Windows (WSL)](#windows-wsl)
  - [Other platforms](#other-platforms)
- [Game data files](#game-data-files)
- [Running](#running)
  - [Quick examples](#quick-examples)
  - [CLI flag reference](#cli-flag-reference)
  - [`hode.ini` reference](#hodeini-reference)
- [Display & rendering modes](#display--rendering-modes)
  - [Standard mode](#standard-mode)
  - [HD upscaling](#hd-upscaling)
  - [Scale chains](#scale-chains)
  - [Widescreen 16:9 borders](#widescreen-169-borders)
  - [Smooth (60 Hz) animation](#smooth-60-hz-animation)
- [Disk cache & prerender](#disk-cache--prerender)
  - [On-disk layout](#on-disk-layout)
  - [Cache key](#cache-key)
  - [Prerender flow](#prerender-flow)
- [Automation API](#automation-api)
  - [Protocol](#protocol)
  - [Commands](#commands)
  - [Python client](#python-client)
  - [Headless testing](#headless-testing)
- [Test scripts](#test-scripts)
- [Engine architecture](#engine-architecture)
  - [High-level dataflow](#high-level-dataflow)
  - [Core types](#core-types)
  - [HD pipeline](#hd-pipeline)
  - [PAF cutscene flow](#paf-cutscene-flow)
- [Source file map](#source-file-map)
- [Game world reference](#game-world-reference)
  - [Levels](#levels)
  - [Cutscenes](#cutscenes)
  - [Default key bindings](#default-key-bindings)
  - [Settings menu](#settings-menu)
  - [Cheat flags](#cheat-flags)
  - [Debug bitmask](#debug-bitmask)
- [Data formats (brief)](#data-formats-brief)
- [Save state](#save-state)
- [Troubleshooting](#troubleshooting)
- [Differences from upstream](#differences-from-upstream)
- [Known limitations](#known-limitations)
- [Contributing](#contributing)
- [Credits](#credits)
- [License & legal](#license--legal)

---

## At a glance

| Area | Feature |
|---|---|
| 🖼️ Rendering | Chained xBRZ at 6× / 8× / 10× / 15× / arbitrary 2..16; MLAA edge smoothing; per-screen background cached once |
| 📺 Widescreen | 16:9 framebuffer with palette-derived gradient borders (no fake gameplay) |
| ⏱️ Smoothness | Decoupled 60 Hz render with 12.5 Hz game tick, sprite-position interpolation |
| 🎬 Cutscenes | PAF frames upscaled and cached to disk; `--prerender` warms the cache fast-forward |
| 🚀 Prerender | Single unified console progress bar covers every sprite (incl. per-screen background animations) and the in-gameplay PAF clips for the level |
| 🤖 Automation | Unix-domain JSON socket: `get_state`, `input`, `step`, `screenshot`, `set_level` — perfect for AI bots and replay harnesses |
| 🧰 Menu/input | Bind any letter, digit, modifier or **Space / Enter / Tab / Backspace / Win**; OK / Cancel / Test row works; one key = one action; defaults stay live alongside custom binds |
| 🍎 macOS | Builds out of the box on Apple Silicon and Intel Macs (added `<libkern/OSByteOrder.h>` shim) |
| 💾 Cache | Content-hashed key (FNV-1a over SPR bytes + dims + flags + palette hash) — survives across runs *and* palette changes |

---

## Building

The project uses a hand-written GNU `Makefile`. There is one external dependency: **SDL2**. C++11 is the language standard; warnings are turned up (`-Wall -Wextra -Wpedantic`) but the build is warning-free on macOS clang and Linux gcc.

Generic build, any POSIX:

```bash
make -j"$(nproc 2>/dev/null || sysctl -n hw.ncpu)"
```

Outputs `./hode` (~800 KB, statically links no extras besides SDL2).

`make clean` removes objects (`*.o`) and dependency files (`*.d`).

For an optimised build:

```bash
make -j"$(nproc 2>/dev/null || sysctl -n hw.ncpu)" \
  CPPFLAGS='-O2 -std=c++11 -Wall -Wextra -Wno-unused-parameter -Wpedantic -MMD'
```

### macOS (Apple Silicon & Intel)

This fork includes the changes required to build cleanly on macOS — upstream `hode` fails to compile on macOS because clang's libc doesn't provide a glibc-style `<endian.h>`. The fix lives in [`intern.h`](intern.h):

```cpp
#elif defined(__APPLE__)
  #include <libkern/OSByteOrder.h>
  #define le16toh(x) OSSwapLittleToHostInt16(x)
  #define le32toh(x) OSSwapLittleToHostInt32(x)
  #define htole16(x) OSSwapHostToLittleInt16(x)
  #define htole32(x) OSSwapHostToLittleInt32(x)
  static const bool kByteSwapData = (__BYTE_ORDER__ == __ORDER_BIG_ENDIAN__);
```

Steps for a fresh macOS install:

1. **Install Xcode Command Line Tools** (provides clang/git):
   ```bash
   xcode-select --install
   ```
2. **Install SDL2** via Homebrew:
   ```bash
   brew install sdl2
   ```
   This places `sdl2-config` on `PATH` and `SDL2.framework` / `libSDL2*.dylib` where the linker can find them. The Makefile uses `sdl2-config --cflags --libs`, so no extra plumbing is needed.
3. **Build**:
   ```bash
   make -j"$(sysctl -n hw.ncpu)"
   ```
4. **Run** (data files in the same directory):
   ```bash
   ./hode --hd
   ```

If `brew` is in a non-standard prefix (e.g. `/opt/homebrew` on Apple Silicon), make sure `/opt/homebrew/bin` is on your `PATH` so `sdl2-config` is found:

```bash
echo 'export PATH="/opt/homebrew/bin:$PATH"' >> ~/.zprofile
source ~/.zprofile
```

For a debug build with `lldb`:

```bash
make clean && make -j"$(sysctl -n hw.ncpu)" CPPFLAGS='-g -O0 -std=c++11 -Wall -Wextra -Wno-unused-parameter -Wpedantic -MMD'
lldb -- ./hode --hd
```

#### macOS-specific notes

| Concern | Notes |
|---|---|
| Window display backend | SDL2 uses Metal via SDL_RENDERER_ACCELERATED; nothing extra required. |
| Endianness | `<libkern/OSByteOrder.h>` is the macOS-blessed equivalent of `<endian.h>`. The fork's `intern.h` selects the right header per platform. |
| Hardened runtime / Gatekeeper | The Makefile produces an unsigned binary. The first launch from Finder may show a "cannot be opened" dialog — right-click → Open, or run from Terminal. No entitlements are required. |
| Audio | Audio uses SDL's CoreAudio backend by default. Set `SDL_AUDIODRIVER=dummy` to disable for headless / CI use. |
| Headless / CI | macOS has no `Xvfb` equivalent that's worth fighting. Use `SDL_VIDEODRIVER=dummy SDL_AUDIODRIVER=dummy` on macOS for non-interactive runs (still requires a display server unless you patch out `SDL_INIT_VIDEO`). For real headless CI, run the Linux build inside a container with `xvfb-run`. |
| Apple Silicon | The build is native arm64 with clang's auto-vectorisation; xBRZ scalers benefit. No Rosetta required. |
| Bundle / `.app` | This fork ships a CLI binary, not a `.app` bundle. If you want a real app bundle (icon in dock, drag-to-Applications), wrap with [`appify`](https://gist.github.com/mathiasbynens/674099) or build one with `Info.plist`; the engine itself doesn't care. |

A clean build on macOS is normally a few seconds:

```text
$ make clean && make -j$(sysctl -n hw.ncpu) 2>&1 | tail -3
c++  -g -std=c++11 -Wall -Wextra -Wno-unused-parameter -Wpedantic `sdl2-config --cflags`  -MMD  -c -o video.o video.cpp
c++  -o hode andy.o automation_api.o ... `sdl2-config --libs`
```

### Linux

```bash
# Debian / Ubuntu / WSL
sudo apt install build-essential libsdl2-dev pkg-config

# Fedora / RHEL
sudo dnf install gcc-c++ SDL2-devel

# Arch
sudo pacman -S base-devel sdl2

make -j"$(nproc)"
```

For headless CI:

```bash
SDL_AUDIODRIVER=dummy xvfb-run -a ./hode --automation=/tmp/hode.sock --hd
```

### Windows (WSL)

Use WSL2 + an X server (e.g. WSLg, included in Windows 11 by default):

```bash
sudo apt install build-essential libsdl2-dev pkg-config
make -j"$(nproc)"
```

Native Windows builds (MinGW / MSYS2) are not currently maintained in this fork; the Makefile assumes POSIX, but the original code is portable — patches welcome.

### Other platforms

The upstream tree ships `system_psp.cpp` and `system_wii.cpp` (PSP / Wii Homebrew) but they are not wired into this `Makefile` — only `system_sdl2.cpp` is built. The HD compositor and disk-cache changes target SDL2 by design and would need backend work to run on PSP / Wii.

---

## Game data files

You must own the original *Heart of Darkness* PC release (full or demo). Place these next to the binary or pass `--datapath=PATH`:

| File | Description | Approx. size |
|---|---|---:|
| `HOD.PAF` | All cutscenes (Packed Animation File container) | ~411 MB |
| `SETUP.DAT` | Strings, fonts, hint images, loading-screen art, sound bank metadata | ~5.5 MB |
| `ROCK_HOD.LVL` … `DARK_HOD.LVL` | Per-level geometry, sprite tables, screen layout | 0.7 – 5 MB ×9 |
| `ROCK_HOD.SSS` … `DARK_HOD.SSS` | Sound script (SssBank/SssSample) for each level | 3 – 7 MB ×9 |
| `ROCK_HOD.MST` … `DARK_HOD.MST` | Monster + scripting tables | 10 KB – 150 KB ×9 |

PSX disc data (`*.dax`, MDEC backgrounds, SPU ADPCM sounds) is also supported by the original engine. The HD upscaler works for both PC and PSX paths.

`RELEASES.yaml` (carried over from upstream) lists SHA-1 hashes of every game version the engine has been validated against (French / German / English Win32, demos, PSX, etc.). If you suspect bad data files, compare them against the entries there.

> ⚠️ The data files are **not redistributed** — bring your own from your CD / store purchase.

---

## Running

### Quick examples

```bash
# Vanilla 256×192 windowed
./hode

# HD (1536×1152) at default 6× scale
./hode --hd

# 4K with 16:9 borders, 60 Hz interpolation, persistent cache, prerender first run
./hode --4k --hd-wide --smooth --hd-cache=./cache --prerender

# Skip menu, jump to Island level, checkpoint 2, in HD
./hode --level=isld --checkpoint=2 --hd

# Custom 9× scale (2304×1728)
./hode --hd-scale=9

# Scriptable mode — opens a Unix socket and disables menu/loading/cutscenes
./hode --automation=/tmp/hode.sock --hd

# Read data files from elsewhere
./hode --datapath=/Volumes/HOD-CD --savepath=~/Library/Application\ Support/hode
```

### CLI flag reference

All flags use the GNU long form (`--name` or `--name=value`); there are no short flags.

| Flag | Argument | Default | Effect |
|---|---|---|---|
| `--datapath=PATH` | path | `.` | Directory containing `HOD.PAF`, `SETUP.DAT`, level files |
| `--savepath=PATH` | path | `.` | Directory for `setup.cfg` and screenshots |
| `--level=NUM` | int 0–8 or name (`rock`, `fort`, `pwr1`, `isld`, `lava`, `pwr2`, `lar1`, `lar2`, `dark`) | — | Skip menu and start at given level |
| `--checkpoint=NUM` | int | 0 | Checkpoint within `--level` |
| `--debug=MASK` | int | 0 | Bitmask OR'd into `g_debugMask` (see [Debug bitmask](#debug-bitmask)) |
| `--cheats=MASK` | int | 0 | Bitmask OR'd into cheat flags (see [Cheat flags](#cheat-flags)) |
| `--hd` | — | off | Enable HD compositor at default 6× scale |
| `--hd-scale=N` | int 2–16 | — | HD compositor at custom scale; implies `--hd` |
| `--fullhd` | — | off | Shortcut: HD at 8× (2048×1536) |
| `--4k` | — | off | Shortcut: HD at 15× (3840×2880; internally 16× cropped) |
| `--hd-wide` | — | off | 16:9 framebuffer with palette-gradient borders; also forces SDL window to 16:9 aspect |
| `--hd-cache=PATH` | path | — | Read/write upscaled sprites + PAF frames under `PATH/<scale>x/` |
| `--prerender` | — | off | Pre-fill sprite + relevant PAF caches per level (slow first run, instant after) |
| `--smooth` | — | off | 60 Hz render with sprite-position interpolation, 12.5 Hz game logic |
| `--automation=PATH` | unix-socket | — | Open a JSON automation socket; also disables menu, loading screen and cutscenes for fast scripted boot |

> CLI flags **override** matching `hode.ini` keys.

### `hode.ini` reference

Place a `hode.ini` next to the binary (or in `--datapath`) for persistent configuration:

```ini
[engine]
disable_paf       = false   ; skip cutscenes (auto-true if HOD.PAF missing)
disable_mst       = false   ; skip monster scripting (debugging only)
disable_sss       = false   ; skip sound script (debugging only)
disable_menu      = false   ; boot straight into the game
max_active_sounds = 16      ; SSS object pool cap
difficulty        = 1       ; 0=easy, 1=normal, 2=hard
frame_duration    = 80      ; ms per game tick (default 80 = 12.5 Hz)
loading_screen    = true    ; show "please wait" image during level load

[display]
scale_factor      = 3       ; integer upscaler factor (legacy non-HD path)
scale_algorithm   = nearest ; nearest | linear | xbr | nearest+blur | …
gamma             = 1.0     ; output gamma (1.0 = neutral)
fullscreen        = false
widescreen        = false   ; legacy 16:9 path (blur-stretch). Prefer hd_widescreen.
hd_mode           = false   ; same as --hd
hd_scale          = 0       ; same as --hd-scale=N (0 = default 6)
hd_widescreen     = false   ; same as --hd-wide
hd_cache          = ./cache ; same as --hd-cache
smooth_anim       = false   ; same as --smooth
automation_socket = /tmp/hode.sock ; same as --automation
```

Legend:

- Booleans accept `true`/`false`, `t`/`f`, `1`/`0` (case-insensitive).
- `disable_paf` is auto-set to `true` if the engine can't open `HOD.PAF`.
- Lines starting with `#` are comments. Blank lines are tolerated.

---

## Display & rendering modes

### Standard mode

`./hode` (no flags) gives you the upstream behaviour: 256×192 indexed-colour framebuffer, integer-upscaled `scale_factor` times by SDL through the chosen software scaler. Behaviour and visuals match upstream `hode` exactly.

### HD upscaling

`--hd`, `--fullhd`, `--4k`, `--hd-scale=N` route every visible pixel through `HdCompositor` (in `hd_compositor.{h,cpp}`):

- Background bitmap is upscaled **once per screen change** and reused.
- Each sprite is upscaled **once per (content + dimensions + flip + palette) tuple** and cached.
- The HD compositor framebuffer is RGBA32; SDL2 streams it to a `SDL_Texture` per frame.

### Scale chains

xBRZ scales by 2×, 3×, 4× or 5× per pass; arbitrary scales are decomposed into two passes:

| `--hd-scale` / preset | First pass | Second pass | Output (256×192 →) |
|---:|:--|:--|:--|
| 2 | 2× | — | 512×384 |
| 3 | 3× | — | 768×576 |
| 4 | 4× (= 2×·2×) | — | 1024×768 |
| 5 | 5× (nearest) | — | 1280×960 |
| **6** (`--hd`) | 3× | 2× | 1536×1152 |
| 7 | 4× | 2× → crop | 1792×1344 |
| **8** (`--fullhd`) | 4× | 2× | 2048×1536 |
| 9 | 3× | 3× | 2304×1728 |
| **10** | 4× | 3× → crop | 2560×1920 |
| 11 | 4× | 3× → crop | 2816×2112 |
| 12 | 4× | 3× | 3072×2304 |
| 13 | 4× | 4× → crop | 3328×2496 |
| 14 | 4× | 4× → crop | 3584×2688 |
| **15** (`--4k`) | 4× | 4× → SDL downscale | 3840×2880 |
| 16 | 4× | 4× | 4096×3072 |

After the chained scale, [`mlaa_smooth()`](edge_smooth.cpp) runs a morphological-edge-AA pass that softens stair-stepping artefacts on diagonal edges (cape, hair, plasma traces).

### Widescreen 16:9 borders

`--hd-wide` switches the HD framebuffer from 4:3 to 16:9. The 4:3 game stays centred; left and right margins are filled with a vertical gradient sampled from the top and bottom rows of the **current screen's palette**, then darkened to ~⅓ brightness so it reads as ambient atmosphere instead of fake gameplay. The borders re-sample whenever the screen changes, so they track lighting/mood per area.

This fork additionally **resizes the SDL window itself to 16:9** when `--hd-wide` is on. Without that change (upstream behaviour), the wide framebuffer is squished into a 4:3 window and the colored borders are clipped to invisibility.

### Smooth (60 Hz) animation

`--smooth` decouples logic from rendering:

```text
                ┌───── 12.5 Hz ─────┐       ┌───── ~60 Hz ─────┐
                │  game tick        │       │   render frame   │
                │  ‒ AI / physics   │       │   ‒ interpolate  │
                │  ‒ sprite list    │  ──►  │     prev → curr  │
                │  ‒ snapshot pos   │       │   ‒ blit to HD   │
                └───────────────────┘       └──────────────────┘
```

- Game tick stays at the original 80 ms (12.5 Hz). All collision, AI, sound, scripted callbacks behave exactly as upstream.
- `Game::saveInterpolationState()` snapshots `(prev, curr)` for Andy and every visible sprite at each tick.
- `Game::renderInterpolatedFrame(t)` — called at the render rate — blends each sprite's previous and current position with `t ∈ [0,1]`.

The result is fluid on-screen motion with zero impact on gameplay timing.

---

## Disk cache & prerender

`--hd-cache=PATH` makes every upscaled artifact persistent.

### On-disk layout

```
PATH/
├── 6x/                                  ← sprites at the active scale
│   └── spr_<key:016x>_<w>x<h>.raw      ← header (int32 w, int32 h) + w*h*4 RGBA bytes
├── 8x/
└── paf/
    ├── 6x/
    │   ├── v00/                          ← cutscene 0 (intro)
    │   │   ├── f0000.raw
    │   │   ├── f0001.raw
    │   │   └── …
    │   ├── v22/                          ← Canyon Andy-falling-with-cannon
    │   └── v24/                          ← Island Andy-falling
    └── 8x/
```

Each `<scale>x/` directory is independent — switching `--hd-scale=10` populates `cache/10x/` without touching `cache/6x/`.

### Cache key

```
key = FNV-1a-64(
    decoded indexed bytes (w×h)
  ⊕ w little-endian uint16
  ⊕ h little-endian uint16
  ⊕ flags & 3                  ← horizontal-flip bit
  ⊕ palette FNV-1a-64
)
```

Including the **palette hash** is the reason the same sprite content rendered with different on-screen palettes (between screens, cross-fade, cutscenes) gets distinct cache entries. Without that, low scales (HD/FullHD) would occasionally show "wrong-coloured" sprite frames as a stale entry was reused after a palette transition.

> Note: this changes the on-disk format compared to upstream and earlier builds of this fork. **Delete any pre-existing `cache/` directory** before the first run with palette-hashed keys.

### Prerender flow

`--prerender` populates the cache up-front, **once per level** (tracked via a 32-bit bitmask `Game::_hdPrerenderedMask`). After `Game::restartLevel()` finishes loading the level and before the gameplay loop starts:

1. **Sprite phase** — walk every `(sprite-type, frame, flip)` tuple across:
   - the main level table `_resLevelData0x2988PtrTable[0..31]`
   - every screen's `backgroundLvlObjectDataTable[0..7]` (the per-screen background animations: trees, water, ambient effects). Without this, those animations only upscaled the first time they appeared on-screen.
2. **PAF phase** — fast-forward decode the small in-gameplay clips (Canyon falling with cannon, Canyon falling, Island falling) plus the level's intro cutscene (`_cutscenes[level]`). Each not-yet-cached cutscene is decoded full-speed without audio/display/sleep so the existing HD frame callback writes every frame to disk.
3. Both phases share **one progress bar** sized by the total work (sprite frames + PAF frames):
   ```
   prerender level 0 (rock) [############################....] 4720/5320 (88%) 5.2s eta 0.7s
   ```

The `_playedMask` tracking "watched cutscenes" is snapshotted/restored across each PAF prerender so your save file isn't polluted with movies you didn't actually watch. After the bar completes, the engine pumps SDL once and re-baselines `inp.prevMask = inp.mask` so any keys held during prerender don't fire phantom press/release events on the first gameplay tick.

---

## Automation API

`--automation=/tmp/hode.sock` opens a non-blocking `AF_UNIX` `SOCK_STREAM` listening for one client at a time. Newline-delimited JSON in both directions.

`--automation` implies:

- `disable_menu = true` — boot straight into gameplay
- `loading_screen = false` — no "please wait"
- `disable_paf = true` — cutscenes skipped

### Protocol

- One JSON object per line, terminated with `\n`.
- The server only responds to `get_state` and `screenshot` (a JSON header line, then for screenshot the raw RGB bytes). Other commands are fire-and-forget.
- Backpressure: writes are blocking; the client should drain replies promptly.
- Connection drops are detected on next `read()`; the engine continues running and re-`accept()`s.

### Commands

| `cmd` | Body fields | Reply | Effect |
|---|---|---|---|
| `get_state` | – | JSON object (see below) | Snapshot Andy + monsters + level/checkpoint/screen |
| `input` | `dir` (UDLR mask), `act` (RUN/JUMP/SHOOT mask), `frames` (int), `raw` (override for raw `SYS_INP_*` mask) | – | Inject input for `frames` ticks |
| `step` | `count` (int) | – | Enable step mode and step `count` ticks; gameplay blocks between batches |
| `screenshot` | – | header line then `width*height*3` raw RGB bytes | Capture the 256×192 framebuffer |
| `set_level` | `level` (0–8), `checkpoint` (int) | – | Break out of the current level loop and restart at the given level/checkpoint |

`get_state` reply schema (truncated for clarity):

```json
{
  "andy": {
    "x": 128, "y": 96, "screen": 0,
    "anim": 0, "frame": 0, "sprite": 0,
    "hasCannon": true, "dying": false
  },
  "level": 0, "checkpoint": 0, "screen": 0,
  "endLevel": false,
  "monsters": [
    {"x": 200, "y": 100, "type": 1, "i": 0},
    {"x": 220, "y": 100, "type": 2, "i": 0}
  ],
  "monsterCount": 2
}
```

Input bit layout:

| Bit | `dir` | `act` | Raw `SYS_INP_*` |
|---:|:--|:--|:--|
| 0x01 | UP | RUN | `SYS_INP_UP=0x01`, `SYS_INP_RUN=0x10` |
| 0x02 | RIGHT | JUMP | `SYS_INP_RIGHT=0x02`, `SYS_INP_JUMP=0x20` |
| 0x04 | DOWN | SHOOT | `SYS_INP_DOWN=0x04`, `SYS_INP_SHOOT=0x40` |
| 0x08 | LEFT | – | `SYS_INP_LEFT=0x08` |
| 0x80 | – | – | `SYS_INP_ESC=0x80` (raw only) |

### Python client

```python
import socket, json, time

s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect("/tmp/hode.sock")
f = s.makefile("rwb", buffering=0)

def send(j): f.write((json.dumps(j) + "\n").encode())
def recv_line(): return json.loads(f.readline().decode())

# State snapshot
send({"cmd": "get_state"})
state = recv_line()
print(state["andy"], "monsters:", state["monsterCount"])

# Walk right + run for 30 ticks
send({"cmd": "input", "dir": 2, "act": 1, "frames": 30})

# Frame-by-frame step for 10 ticks
send({"cmd": "step", "count": 10})

# Screenshot — header + raw RGB
send({"cmd": "screenshot"})
hdr = recv_line()           # {"width":256,"height":192,"format":"rgb","size":N}
img = b""
while len(img) < hdr["size"]:
    img += s.recv(4096)
open("frame.rgb", "wb").write(img)

# Jump to a specific level
send({"cmd": "set_level", "level": 3, "checkpoint": 0})
```

### Headless testing

```bash
# Linux
SDL_AUDIODRIVER=dummy xvfb-run -a ./hode --automation=/tmp/hode.sock --hd

# macOS (still needs a window server; for true headless use the Linux build in CI)
SDL_AUDIODRIVER=dummy ./hode --automation=/tmp/hode.sock
```

---

## Test scripts

Drop-in Python drivers for the automation API. All require a running `./hode --automation=/tmp/hode.sock`.

| Script | Purpose |
|---|---|
| [`test_walkthrough.py`](test_walkthrough.py) | Drive Andy through level 1 end-to-end with scripted inputs. Useful as a smoke test after engine changes. |
| [`test_all_levels.py`](test_all_levels.py) | Boot each of the 9 levels, verify gameplay starts, optionally take a screenshot. |
| [`test_combat_bot.py`](test_combat_bot.py) | Reactive AI: query monsters via `get_state`, fire when in range. Demonstrates `input` + `get_state` polling. |
| [`test_hd_replay.py`](test_hd_replay.py) | Compare HD vs SD rendering frame-by-frame. |

---

## Engine architecture

### High-level dataflow

```text
┌──────────────────── main.cpp ────────────────────┐
│  parse CLI / hode.ini → readConfigIni()         │
│  open SETUP.DAT       → Resource::loadSetupDat │
│  init SDL2/audio      → System::init / setupAudio│
│  if --hd : new HdCompositor                      │
│  if --automation : new AutomationApi             │
│  loop:                                           │
│    Game::loadSetupCfg                            │
│    apply keymap                                  │
│    Menu::mainLoop  (unless --automation)         │
│    Game::mainLoop(level, checkpoint)             │
└──────────────────────────────────────────────────┘
                  │
                  ▼
┌──────────────────── Game ────────────────────────┐
│  Resource (.lvl/.sss/.mst/.paf parsers)          │
│  Video (256×192 indexed framebuffer + palette)   │
│  PafPlayer (cutscene streaming)                  │
│  Mixer (software audio mix)                      │
│  HdCompositor* (optional HD path)                │
│  AutomationApi* (optional socket server)         │
│  per-level callbacks (level1_rock.cpp .. level9) │
└──────────────────────────────────────────────────┘
                  │
                  ▼
        ┌─────────────────────┐
        │   System (abstract) │  ← System_SDL2 (this build)
        │                     │     System_PSP, System_Wii (legacy)
        └─────────────────────┘
                  │
                  ▼
              SDL2 (Metal/OpenGL/D3D)
```

### Core types

| Type | File | Responsibility |
|---|---|---|
| `Game` | [`game.h`](game.h) / [`game.cpp`](game.cpp) | Orchestrator. Holds Video / Resource / PafPlayer / HdCompositor / AutomationApi pointers; ticks the level main loop; per-level callbacks live in `level1_rock.cpp` .. `level9_dark.cpp`. |
| `Resource` | [`resource.h`](resource.h) | Owns LVL / SSS / MST parsers, sprite tables (`_resLevelData0x2988PtrTable[32]`, `_resLvlScreenBackgroundDataTable[40]`), font/loading-screen images, palettes. |
| `Video` | [`video.h`](video.h) / [`video.cpp`](video.cpp) | Legacy 256-colour framebuffer with `_frontLayer`, `_backgroundLayer`, `_shadowLayer`. Owns the palette + font. |
| `PafPlayer` | [`paf.h`](paf.h) / [`paf.cpp`](paf.cpp) | Streams the proprietary Packed Animation File (per-frame indexed bitmap + delta encoding + ADPCM audio). |
| `HdCompositor` | [`hd_compositor.h`](hd_compositor.h) / [`hd_compositor.cpp`](hd_compositor.cpp) | New: HD framebuffer, multi-resolution presets, 16:9 borders, prerender driver. |
| `SpriteUpscaler` | [`sprite_upscaler.h`](sprite_upscaler.h) / [`sprite_upscaler.cpp`](sprite_upscaler.cpp) | New: chained xBRZ scalers + content-hashed RAM/disk cache. |
| `AutomationApi` | [`automation_api.h`](automation_api.h) / [`automation_api.cpp`](automation_api.cpp) | New: Unix socket JSON server. |
| `System` | [`system.h`](system.h) | Abstract platform — input, audio, screen, key mapping. `System_SDL2` is the active backend. |

### HD pipeline

```text
Original logic (256×192, 8-bit indexed)
        │
        ▼  drawScreen() collects sprite list (z-sorted)
        │
        ▼  HdCompositor::beginFrame(bgLayer, palette, screenNum, bgId)
        │   ‒ expand palette 6→8-bit
        │   ‒ if screen changed: SpriteUpscaler::upscaleBackground()
        │   ‒ memcpy _hdBackground → _hdFramebuffer
        │
        ▼  for each sprite:
        │     SpriteUpscaler::getOrUpscale(sprData, w, h, flags, palette)
        │       ‒ decodeSprToTemp(): RLE → indexed
        │       ‒ key = FNV-1a(indexed + dims + flags + palette hash)
        │       ‒ RAM lookup → disk lookup → upscale chain → MLAA → cache
        │     blitHdSprite(): alpha-test composite into _hdFramebuffer
        │
        ▼  HdCompositor::endFrame()
        │   ‒ if widescreen: compositeWidescreen() draws gradient borders
        │
        ▼  System_SDL2::copyRectRGBA(): SDL_Texture stream → SDL_RenderCopy → SDL_RenderPresent
```

### PAF cutscene flow

```text
HOD.PAF
  │   one container, k cutscenes, indexed by uint32 LE offset
  ▼
PafPlayer::preload(num)
  │   read sub-header, allocate 4 page buffers + demux blocks
  ▼
PafPlayer::mainLoop()                          ← _prerenderMode skips audio/display/sleep
  │   per frame:
  │     ‒ read N file blocks into demux buffers (video or ADPCM audio)
  │     ‒ decodeVideoFrame() → write current page buffer
  │     ‒ callback(frame) → gamePafFrameCallback (game.cpp)
  │           ├── (legacy) g_system->copyRect to 256×192 layer
  │           └── (HD) cache hit? → present
  │                   else upscale → save to disk → present
  │     ‒ if !prerender: setPalette + updateScreen + sleep frameMs
  ▼
PafPlayer::unload()
```

`PafPlayer::prerender(num)` = `_prerenderMode = true; play(num); _prerenderMode = false;` with `_playedMask` snapshot/restore so prerender doesn't pollute the save's "watched cutscenes" record.

---

## Source file map

### Original engine (preserved)

| File(s) | Responsibility |
|---|---|
| `andy.cpp` | Player state machine / animation FSM |
| `benchmark.cpp` | CPU benchmark used during the loading screen |
| `defs.h` | On-disk struct layouts (`Lvl*`, `Sss*`, `Mst*`, `SetupConfig`, …) |
| `fileio.cpp/h` | Buffered binary file reader |
| `fs.h`, `fs_posix.cpp`, `fs_android.cpp` | Filesystem abstraction |
| `intern.h` | LE byte-swap helpers; this fork adds a macOS branch |
| `level.h`, `level1_rock.cpp` … `level9_dark.cpp` | Per-level scripted callbacks (`callLevel_initialize`, `callLevel_postScreenUpdate`, …) |
| `lzw.cpp` | LZW decompressor used for some assets in v11 data |
| `mdec.cpp/h`, `mdec_coeffs.h` | PSX MDEC video decoder for cutscene + background overlays |
| `menu.cpp/h` | Settings menus, cutscene replay screen, controls bind |
| `mixer.cpp/h` | Software audio mixer |
| `monsters.cpp` | Monster AI / animation tables |
| `paf.cpp/h` | PAF cutscene format |
| `random.cpp/h` | Game logic PRNG |
| `resource.cpp/h` | LVL / SSS / MST loader, sprite tables, font loader |
| `scaler.h`, `scaler_xbr.cpp` | Legacy software scalers (the small-window path) |
| `screenshot.cpp/h` | PNG dump (BMP fallback) |
| `sound.cpp` | SSS interpreter (script-driven sound triggers) |
| `staticres.cpp` | Baked-in tables (cutscene mappings, font character indices) |
| `system.h` | Abstract platform interface |
| `system_sdl2.cpp` | Active SDL2 backend (input, audio, screen, key mapping) |
| `system_psp.cpp`, `system_wii.cpp` | Legacy backends, **not built** by this Makefile |
| `util.cpp/h` | Debug print, error helpers |
| `video.cpp/h` | Legacy framebuffer + palette |

### Added by this fork

| File(s) | Responsibility |
|---|---|
| `hd_compositor.cpp/h` | HD framebuffer; multi-resolution presets; 16:9 widescreen with palette borders; prerender driver; unified progress bar |
| `sprite_upscaler.cpp/h` | xBRZ chained scalers (`2x`, `3x`, `4x = 2x·2x`, `5x ≈ nearest`, `Nx via decompose`); RAM LRU + disk cache with FNV-1a content key |
| `edge_smooth.cpp/h` | MLAA edge smoothing |
| `automation_api.cpp/h` | Unix-socket JSON server; input injection; step mode; screenshot |
| `test_walkthrough.py` | Level 1 walkthrough using the automation API |
| `test_all_levels.py` | Per-level boot smoke test |
| `test_combat_bot.py` | Reactive combat AI |
| `test_hd_replay.py` | HD vs SD comparison harness |

### Modified vs upstream

| File | Why |
|---|---|
| `Makefile` | Add the four new `.cpp` sources to `SRCS` |
| `intern.h` | macOS `<libkern/OSByteOrder.h>` shim |
| `main.cpp` | New CLI flags + INI keys; wire `_video->_font` after `loadSetupDat()` |
| `game.cpp/h` | HD compositor begin/end-frame hooks; smooth-anim 60 Hz interpolated render loop; per-level sprite + PAF prerender driver with shared progress bar |
| `paf.cpp/h` | HD frame callback path; `_prerenderMode` skips audio/display/sleep; `peekFramesCount(num)` for sizing the unified progress bar |
| `menu.cpp/h` | Keyboard binding screen overhaul (OK/Cancel/Test, one-key-one-action, Space/Enter/Tab/Backspace/Win bindable, fallback labels); font init guard |
| `system.h` | Expose `applyKeyboardControls` and `waitForKeyPress` on the abstract interface |
| `system_sdl2.cpp` | Default mappings stay live alongside user keys; `waitForKeyPress` clears edge state on return; `copyRectRGBA` for HD presents |
| `video.cpp` | Zero-init `_font` so the menu can't dereference garbage before `Game::mainLoop` wires it |

---

## Game world reference

### Levels

| # | Code | Display name | Notes |
|---:|:--|:--|:--|
| 0 | `rock` | Canyon | Tutorial / first level |
| 1 | `fort` | Fort | |
| 2 | `pwr1` | Power 1 | The Swamp |
| 3 | `isld` | Island | |
| 4 | `lava` | Lava | |
| 5 | `pwr2` | Power 2 | Underwater |
| 6 | `lar1` | Lair 1 | |
| 7 | `lar2` | Lair 2 | |
| 8 | `dark` | Dark | Final level (single checkpoint) |

Level callbacks dispatch from `Game::callLevel_*` into the `levelN_<code>.cpp` matching the player's current level (`_currentLevel`).

### Cutscenes

The 25 PAF videos in `HOD.PAF`:

| # | Symbol | Purpose |
|---:|:--|:--|
| 0 | `kPafAnimation_intro` | Game intro |
| 1 | `kPafAnimation_cine14l` | Cinematic transition |
| 2 | `kPafAnimation_rapt` | Cinematic |
| 3 | `kPafAnimation_glisse` | Slide |
| 4 | `kPafAnimation_meeting` | Meeting cinematic |
| 5 | `kPafAnimation_island` | Island intro |
| 6 | `kPafAnimation_islefall` | Falling on Island |
| 7 | `kPafAnimation_vicious` | |
| 8 | `kPafAnimation_together` | |
| 9 | `kPafAnimation_power` | |
| 10 | `kPafAnimation_back` | |
| 11 | `kPafAnimation_dogfree1` | |
| 12 | `kPafAnimation_dogfree2` | |
| 13 | `kPafAnimation_meteor` | |
| 14 | `kPafAnimation_cookie` | |
| 15 | `kPafAnimation_plot` | |
| 16 | `kPafAnimation_puzzle` | |
| 17 | `kPafAnimation_lstpiece` | |
| 18 | `kPafAnimation_dogfall` | |
| 19 | `kPafAnimation_lastfall` | |
| 20 | `kPafAnimation_end` | |
| 21 | `kPafAnimation_cinema` | |
| 22 | `kPafAnimation_CanyonAndyFallingCannon` | In-gameplay: Canyon, falls with cannon + helmet |
| 23 | `kPafAnimation_CanyonAndyFalling` | In-gameplay: Canyon, falls without cannon |
| 24 | `kPafAnimation_IslandAndyFalling` | In-gameplay: Island falling |

The per-level intro cutscene is taken from the table `_cutscenes[] = { 0, 2, 4, 5, 6, 8, 10, 14, 19 }` (one entry per level 0..8).

### Default key bindings

| Key | Action |
|---|---|
| Arrow keys | Move (Up = climb, Down = crouch) |
| Left Ctrl, F | Run |
| Left Alt, G, Enter | Jump |
| Left Shift, H | Shoot |
| D, Space | Special (Run + Shoot — fires plasma cannon if equipped) |
| Esc | Pause / quit menu |
| S | Screenshot |

### Settings menu

`Menu → Settings → Keyboard Controls` lets you bind any of Run / Jump / Shoot / Special:

- **Letters, digits, Shift, Ctrl, Alt** – have icon glyphs in the engine's bitmap font.
- **Space, Enter, Tab, Backspace, Cmd/Win** – also bindable; menu shows short text labels (`SP`, `EN`, `TB`, `BS`, `WN`) since the bitmap font has no glyphs for them.
- **Two slots per action** – the first bind goes into slot 1, the second into slot 2.
- **One key, one action** – binding a key already used by another action automatically clears the prior binding.
- **Defaults stay live** – `LCtrl`, `F`, `LAlt`, `G`, `LShift`, `H`, `D`, `Space` remain mapped to their defaults alongside any custom keys, so the in-menu Select handler always works while you're rebinding the rest.
- **OK / Cancel / Test row** – navigable via ←/→. Select on **OK** keeps changes, **Cancel** reverts to the controls snapshot taken when you entered the screen, **Test** enters a live-key visualisation where action icons light up as you press their keys (any arrow key exits Test).
- Esc inside the bind prompt cancels the bind only — it doesn't propagate to the outer menu and won't quit the game.

### Cheat flags

`--cheats=N` (or runtime `cheats` integer) is a bitmask:

| Bit | Symbol | Effect |
|---:|:--|:--|
| `1 << 0` | `kCheatSpectreFireballNoHit` | Spectre fireballs don't hit Andy |
| `1 << 1` | `kCheatOneHitPlasmaCannon` | One-shot kill with plasma cannon |
| `1 << 2` | `kCheatOneHitSpecialPowers` | One-shot kill with special powers |
| `1 << 3` | `kCheatWalkOnLava` | Walk on lava without dying |
| `1 << 4` | `kCheatGateNoCrush` | Gates won't crush Andy |
| `1 << 5` | `kCheatLavaNoHit` | Lava doesn't damage |
| `1 << 6` | `kCheatRockShadowNoHit` | Shadow monsters in rock don't damage |

### Debug bitmask

`--debug=N` is a bitmask, OR'd into the global `g_debugMask`:

| Bit | Symbol | Output |
|---:|:--|:--|
| `1 << 0` | `kDebug_GAME` | Game / level state transitions |
| `1 << 1` | `kDebug_RESOURCE` | Resource loader |
| `1 << 2` | `kDebug_ANDY` | Andy state machine |
| `1 << 3` | `kDebug_SOUND` | SSS interpreter |
| `1 << 4` | `kDebug_PAF` | PAF cutscene player |
| `1 << 5` | `kDebug_MONSTER` | Monster AI |
| `1 << 6` | `kDebug_SWITCHES` | `lar1` and `lar2` switches |
| `1 << 7` | `kDebug_MENU` | Menu state machine |

`--debug=255` enables everything.

---

## Data formats (brief)

The original `Heart of Darkness` data files are well-defined binary formats; full reverse-engineering notes live in the upstream `hode` source as struct comments. A summary:

### `*_HOD.LVL`

- 4-byte tag `0x4D5A4448` ('HDZM') at offset 0.
- 4 bytes header counts: `screensCount`, `staticLvlObjectsCount`, `otherLvlObjectsCount`, `spritesCount`.
- `_screensGrid[N][4]` at 0x08: per-screen up/right/down/left neighbour table.
- `_screensBasePos[N]` at 0xA8: per-screen world coordinate.
- `_screensState[N]` at 0x1E8: per-screen flags (0..3).
- `_resLvlScreenObjectDataTable[]` at 0x288: 96-byte `LvlObject` entries.
- Sprite type table at `_lvlSpritesOffset = 0x288 + 96 * (96 or 104)`, 32 × 16-byte entries (each describes a `LvlObjectData` data island).
- Background table at `_lvlBackgroundsOffset = _lvlSpritesOffset + 32*16`, 40 × 16-byte entries.

### `*_HOD.SSS`

- Sound-script bytecode interpreted by `sound.cpp`. `SssBank`, `SssSample`, `SssPcm`, `SssPreloadList`, `SssPreloadInfoData` structures defined in `resource.h`.
- ADPCM sample data (PSX) or PCM samples (PC) referenced by offset.

### `*_HOD.MST`

- Monster + scripting tables. Op codes interpreted by `executeMstCode()` in `monsters.cpp`.

### `HOD.PAF`

- Container of N cutscenes, each indexed by a uint32 LE offset at the start.
- Each cutscene starts with a fixed signature `Packed Animation File V1.0\n(c) 1992-96 Amazing Studio\n`.
- Header at offset `+0x84..0xAC`: `framesCount`, `frameDuration`, `startOffset`, `preloadFrameBlocksCount`, `readBufferSize`, `maxVideoFrameBlocksCount`, `maxAudioFrameBlocksCount`, `frameBlocksCount`.
- Followed by `frameBlocksCountTable[framesCount]`, `framesOffsetTable[framesCount]`, `frameBlocksOffsetTable[frameBlocksCount]`.
- Each frame is delta-encoded against the previous; 4-page rotation buffer; 4 op codes (0..3) for partial vs full blits and palette updates.
- Audio (when present) is 22 kHz ADPCM in interleaved blocks.

### `SETUP.DAT`

- Versioned (10 or 11). Header has counts (`iconsCount`, `menusCount`, `cutscenesCount`, `levelsCount`, `levelCheckpointsCount[8]`, `yesNoQuitImage`, `soundDataSize`, `loadingImageSize`).
- Followed by aligned blocks: loading image (with palette), font (1024 × 16-byte glyphs), menu/options bitmaps, hint images, sound metadata.

### `setup.cfg`

- 212-byte `SetupConfig` struct (`defs.h:55`) — 4 player slots × 52 bytes (progress, level, checkpoint, cutscenes mask, controls, difficulty, stereo, volume, last-level), plus `currentPlayer` and a checksum.

---

## Save state

`setup.cfg` is the engine's binary save file. It tracks per player slot:

- Per-level progress (highest checkpoint reached)
- Last-played level + checkpoint
- Watched-cutscenes bitmask
- 32 bytes of controls (16 bytes joystick, 8 bytes keyboard scancodes, 8 unused)
- Difficulty, stereo, volume, last-level

Up to 4 player slots. Selected via the menu.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `fatal error: 'endian.h' file not found` (macOS) | Use this fork's `intern.h` (it has the macOS `__APPLE__` branch). `git pull && make clean && make`. |
| First-run 4K is sluggish | Expected: every sprite/cutscene frame upscales once. Use `--hd-cache=./cache --prerender`; the second run is instant. |
| Widescreen "borders are still black" | Use `--hd-wide` (not `--widescreen`). `--widescreen` is the legacy blur-stretch path. |
| Cutscenes display on screen during `--prerender` | Should not happen with this fork. If it does, you're running an older build — `make clean && make`. |
| Bound a key in the menu but in-game it does nothing | Custom keys are *additive*; defaults still work. Verify that nothing else captured the key (e.g. macOS Cmd+Space, OS-level hotkeys). |
| Game freezes on the loading screen with `--prerender` | Should not happen post-fix; the engine pumps SDL after prerender. If it does happen, build with `make clean && make` and rerun. |
| `setup.cfg` is corrupt or has a weird default keymap | Delete `setup.cfg` — the engine recreates a default one. |
| Audio is silent / distorted | Set `SDL_AUDIODRIVER=dummy` to disable, or `coreaudio` (macOS) / `pulseaudio` (Linux) to force a backend. |
| `Repository not found` when pushing to a fork | Make sure the GitHub fork exists (https://github.com/&lt;user&gt;/hode) before `git push`. |

---

## Differences from upstream

A high-level diff vs `usineur/hode` master:

- **New modules** — `hd_compositor`, `sprite_upscaler`, `edge_smooth`, `automation_api`, plus four Python automation drivers.
- **New CLI flags** — `--hd`, `--hd-scale`, `--hd-wide`, `--hd-cache`, `--fullhd`, `--4k`, `--smooth`, `--automation`, `--prerender`.
- **New INI keys** — `hd_mode`, `hd_scale`, `hd_widescreen`, `hd_cache`, `smooth_anim`, `automation_socket`.
- **Sprite cache key** changed from heap pointer to FNV-1a(content + dims + flags + palette hash). The upstream pointer-keyed disk cache effectively never produced cross-run hits.
- **Widescreen window sizing** — `--hd-wide` now also sizes the SDL window 16:9 (otherwise the wide framebuffer was squashed back into a 4:3 window and gradient borders were invisible).
- **Palette-correct beginFrame** — `HdCompositor::beginFrame()` now expands the engine's 6-bit palette to 8-bit instead of treating it as already 8-bit, so border colours and HD sprite colours are not ~4× too dim.
- **PAF HD path** with disk cache and a fast-forward `prerender(num)` that skips audio / display / sleep.
- **Smooth animation** — 60 Hz interpolated render with 12.5 Hz logic.
- **Menu / input fixes** — OK / Cancel / Test sub-buttons, one-key-one-action enforcement, Space / Enter / Tab / Backspace / Cmd-Win bindable, default keys stay live alongside custom keys, font-pointer init hardened, edge state cleared on `waitForKeyPress`.
- **macOS portability** — `intern.h` selects `<libkern/OSByteOrder.h>` on `__APPLE__`.

---

## Known limitations

- The HD compositor only supports the SDL2 backend. PSP / Wii are not HD-capable in this fork.
- `--4k` is internally 16× cropped to 15× to fit the 4K UHD frame; visible artifacts are below 1 px.
- Cutscene prerender at 4K can use multiple GB of disk per cutscene. For the full PAF set, prefer `--hd` or `--fullhd`.
- The legacy `--widescreen` (blur-stretch) is preserved for parity but `--hd-wide` is preferred when the HD compositor is on.
- The automation API supports one client at a time.

---

## Contributing

1. Fork the repo on GitHub.
2. Branch off `master`:
   ```bash
   git checkout -b feature/<short-name>
   ```
3. Make changes; follow the existing code style:
   - Tabs for indentation, K&R braces
   - `snake_case` for free functions, `lowerCamelCase` for methods, `_member` for fields
   - Plain C++11 — no STL containers in hot paths; raw arrays + `malloc`/`free` are fine
4. Build cleanly with `-Wall -Wextra -Wpedantic` (no new warnings).
5. Commit with a descriptive message and open a PR against this fork or `usineur/hode`.

---

## Credits

- **Original engine** — reverse-engineered by **Gregory Montoir** (cyx@users.sourceforge.net). See [`usineur/hode`](https://github.com/usineur/hode), upstream `README.txt`, and `CHANGES.txt` (preserved in this tree).
- **Original game** — *Heart of Darkness* by **Amazing Studio** (1998), published by Infogrames / Ocean.
- **xBRZ-style scaling** — based on Zenju's xBRZ algorithm (https://sourceforge.net/projects/xbrz/).
- **MLAA edge smoothing** — classic anti-aliasing technique (Reshetov 2009 / Jimenez et al.).

External links:

- [MobyGames: Heart of Darkness](https://www.mobygames.com/game/heart-of-darkness)
- [heartofdarkness.ca](http://heartofdarkness.ca/) — fan resource
- [usineur/hode upstream](https://github.com/usineur/hode)

---

## License & legal

The engine source is provided under the same terms as upstream `hode` (no explicit LICENSE file in upstream — treat it as "use at your own risk; please credit the original author"). The HD additions in this fork inherit the same terms.

The *Heart of Darkness* game data (`HOD.PAF`, `SETUP.DAT`, `*_HOD.*`) is **copyrighted by Amazing Studio / Infogrames** and is **not redistributed** here. You must own a legitimate copy of the original game to play.
