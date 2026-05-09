# hode-hd

Enhanced fork of [Gregory Montoir's `hode`](https://github.com/usineur/hode), a
reverse-engineered reimplementation of the engine used by *Heart of Darkness*
(Amazing Studio, 1998).

This fork keeps the original engine intact and layers on:

- **HD rendering** — xBRZ sprite/background upscaling at 6x / 8x / 10x / 15x with
  on-the-fly chained scalers and MLAA edge smoothing
- **16:9 widescreen** with palette-derived gradient borders
- **Animation interpolation** — optional 60 Hz render with 12.5 Hz logic tick
- **PAF cutscene HD upscaling** with persistent disk cache
- **Sprite & cutscene prerender** — populate the cache up-front so gameplay is
  smooth from the first run
- **Programmatic automation API** — drive the game via a Unix-domain JSON
  socket (input injection, save state, screenshots, frame stepping)
- **Various menu / input fixes** (keyboard rebinding, OK/Cancel/Test buttons,
  default-key fallback so binding one action doesn't kill the others)

The original is built and tested on Linux/Windows; this fork additionally
builds on macOS (`<endian.h>` shim → `<libkern/OSByteOrder.h>`).

---

## Building

### Prerequisites

- SDL2 development libraries
- A C++11 compiler (gcc/clang)

```bash
# macOS
brew install sdl2

# Debian/Ubuntu
sudo apt install libsdl2-dev
```

### Compile

```bash
make -j$(nproc 2>/dev/null || sysctl -n hw.ncpu)
```

### Game data files

Place the following in the working directory (originals from the *Heart of
Darkness* CD/Demo):

- `HOD.PAF` (~411 MB cutscene video)
- `SETUP.DAT`
- `*_HOD.LVL`, `*_HOD.SSS`, `*_HOD.MST` for each of 9 levels

See `RELEASES.yaml` for the upstream-tested data versions.

---

## Running

### Display modes

| Flag | Resolution | Notes |
|------|-----------:|-------|
| (none) | 256×192 (×3 default) | Original engine path |
| `--hd` | 1536×1152 (6x) | xBRZ chain 3×2 |
| `--fullhd` | 2048×1536 (8x) | xBRZ chain 4×2 |
| `--4k` | 3840×2880 (15x) | xBRZ chain 4×4 then crop |
| `--hd-scale=N` | 256N × 192N | Custom scale (clamped to 2–16) |
| `--hd-wide` | 16:9 | Palette-derived gradient borders |
| `--smooth` | — | 60 Hz interpolated render, 12.5 Hz logic |

Combined example:

```bash
./hode --4k --hd-wide --smooth --hd-cache=./cache --prerender
```

### Disk cache

```bash
./hode --hd --hd-cache=./cache
```

Stores upscaled sprites and PAF cutscene frames under
`cache/<scale>x/`. First run is slow (everything upscales on demand), later
runs are instant cache hits.

### Prerender

```bash
./hode --hd --hd-cache=./cache --prerender
```

After each level loads, walk every sprite frame (main + per-screen background
animations) **and** the relevant in-gameplay PAF clips (Canyon-with-cannon
falling, Canyon falling, Island falling, plus the level intro cutscene). All
frames go into the disk cache with a single unified console progress bar:

```
prerender level 0 (rock) [############################...] 4380/4720 (92%) 3.7s eta 0.3s
```

Subsequent runs skip everything that's already cached.

### Start at a specific level

```bash
./hode --level=3          # Island (numeric index)
./hode --level=rock       # Canyon (by name)
./hode --checkpoint=2     # Combine with --level for resume points
```

Level names: `rock`, `fort`, `pwr1`, `isld`, `lava`, `pwr2`, `lar1`, `lar2`,
`dark` (corresponding to indices 0..8).

### Configuration file (`hode.ini`)

```ini
[engine]
disable_paf = false
disable_menu = false
difficulty = 1
frame_duration = 80

[display]
scale_factor = 3
fullscreen = false
widescreen = false
hd_mode = true
hd_scale = 6
hd_widescreen = true
hd_cache = ./cache
smooth_anim = true
automation_socket = /tmp/hode.sock
```

CLI flags override `hode.ini` settings.

---

## Automation API

```bash
./hode --automation=/tmp/hode.sock
```

Programmatic control via a Unix-domain socket with a small JSON protocol.
Useful for headless testing, AI bots, and replay capture.

### Python example

```python
import socket, json

s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.connect("/tmp/hode.sock")

s.sendall(b'{"cmd":"get_state"}\n')
state = json.loads(s.recv(4096).decode())

# Inject input — direction + action bits, optional frame count
s.sendall(b'{"cmd":"input","dir":2,"act":1,"frames":10}\n')  # walk right + run

# Step a single frame
s.sendall(b'{"cmd":"step","count":1}\n')

# Capture a screenshot
s.sendall(b'{"cmd":"screenshot"}\n')

# Jump to a level/checkpoint
s.sendall(b'{"cmd":"set_level","level":3,"checkpoint":0}\n')
```

### Input bit constants

```
Direction: UP=1, RIGHT=2, DOWN=4, LEFT=8
Action:    RUN=1, JUMP=2, SHOOT=4
Raw SYS_INP bits: UP=0x01, RIGHT=0x02, DOWN=0x04, LEFT=0x08,
                  RUN=0x10, JUMP=0x20, SHOOT=0x40, ESC=0x80
```

### Headless testing

```bash
SDL_AUDIODRIVER=dummy xvfb-run -a ./hode --automation=/tmp/hode.sock --hd
```

### Bundled test scripts

| Script | Purpose |
|--------|---------|
| `test_walkthrough.py` | Automated level-1 walkthrough |
| `test_all_levels.py`  | Per-level smoke loop |
| `test_combat_bot.py`  | AI bot that fights monsters |
| `test_hd_replay.py`   | HD vs normal-mode comparison |

---

## HD rendering pipeline

```
Game logic (256×192, 8-bit indexed)
  │
  ▼  drawScreen() emits sprite list
  │
  ▼  HdCompositor intercepts each sprite
  │   • Decode SPR -> indexed
  │   • Convert to RGBA via palette
  │   • Upscale via chained xBRZ (e.g. 3x then 2x = 6x)
  │   • Apply MLAA edge smoothing
  │   • Cache (RAM LRU + disk, content-hash key)
  │
  ▼  Composite into HD framebuffer
  │   • Background upscaled once per screen change
  │   • Sprites blitted at scaled positions
  │   • Optional 16:9 borders (palette-derived gradient)
  │
  ▼  SDL2 renders RGBA to display
```

### xBRZ scale chains

```
6x  = 3x · 2x        12x = 4x · 3x
8x  = 4x · 2x        15x = 4x · 4x → crop
9x  = 3x · 3x        16x = 4x · 4x
```

### Disk cache layout

```
cache/
  6x/                          sprites at 6x
    spr_<key>_<w>x<h>.raw      header + RGBA pixels
  paf/
    6x/
      v00/                     cutscene 0
        f0000.raw
        f0001.raw
        ...
```

The sprite cache key is a content hash (FNV-1a over decoded indexed bytes +
sprite dimensions + flip flags + palette hash) so cached entries persist
correctly across runs and palette changes.

---

## Levels

| Index | Code   | Name        |
|------:|:-------|:------------|
| 0     | rock   | Canyon      |
| 1     | fort   | Fort        |
| 2     | pwr1   | Power 1 (Swamp)   |
| 3     | isld   | Island      |
| 4     | lava   | Lava        |
| 5     | pwr2   | Power 2 (Underwater) |
| 6     | lar1   | Lair 1      |
| 7     | lar2   | Lair 2      |
| 8     | dark   | Dark (Final)|

## Default key bindings

| Key         | Action |
|-------------|--------|
| Arrow keys  | Move |
| Left Ctrl / F | Run |
| Left Alt / G / Enter | Jump |
| Left Shift / H | Shoot |
| D / Space   | Special (Run + Shoot) |
| Escape      | Menu / quit |
| S           | Screenshot |

The defaults stay active even after you bind custom keys in the settings menu
— custom keys are *additive* so the menu's action shortcuts still work while
you're rebinding the rest.

---

## Files added by this fork

| File | Purpose |
|------|---------|
| `automation_api.h/cpp` | Unix-socket server, JSON commands, monster data exposure |
| `hd_compositor.h/cpp`  | HD framebuffer, 16:9 borders, resolution presets, prerender driver |
| `sprite_upscaler.h/cpp`| xBRZ upscaling + content-hash RAM/disk cache |
| `edge_smooth.h/cpp`    | MLAA edge smoothing for upscaled sprites |
| `test_*.py`            | Automation API regression scripts |

## Files modified vs upstream

`Makefile`, `game.cpp/h`, `intern.h`, `main.cpp`, `menu.cpp/h`, `paf.cpp/h`,
`system.h`, `system_sdl2.cpp`, `video.cpp`.

---

## Credits

- Original engine reverse-engineered by **Gregory Montoir** (cyx@users.sourceforge.net),
  see [`usineur/hode`](https://github.com/usineur/hode).
- Original game *Heart of Darkness* by **Amazing Studio** (1998).

## Links

- [MobyGames page](https://www.mobygames.com/game/heart-of-darkness)
- [heartofdarkness.ca](http://heartofdarkness.ca/)
