# Corne (crkbd/rev1) QMK Local Build + Home-Row Mods (WSL + Vim)

This README documents the full setup I used to move from **config.qmk.fm** (JSON-only) to a **local QMK build** so I can:
- generate a `keymap.c` from my JSON,
- add firmware-level settings (e.g., tapping/hold behavior),
- compile to a `.hex` locally,
- flash it (usually via QMK Toolbox on Windows).

---

## 0) Context / Why local build

The QMK Configurator website can compile a firmware from a JSON layout, but it **cannot** apply:
- `TAPPING_TERM`, `CHORDAL_HOLD`, `PERMISSIVE_HOLD`, etc. (these are `config.h` options),
- custom C code like `get_tapping_term()`,
- per-key and advanced mod-tap behavior tuning.

So for home-row mods that behave well at high WPM, a local QMK build is the right workflow.

---

## 1) Install QMK CLI in WSL (Ubuntu)

> Open Ubuntu/WSL terminal.

### Install dependencies + Python tools
```bash
sudo apt update
sudo apt install -y python3 python3-pip git make
```

### Install QMK CLI
```bash
python3 -m pip install --user qmk
```

### Ensure QMK is on PATH (zsh)
```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### Verify
```bash
qmk --version
qmk doctor
```

---

## 2) Get QMK firmware sources

This clones and prepares the `qmk_firmware` repo:

```bash
qmk setup
```

Your repo will be here:
```bash
~/qmk_firmware
```

---

## 3) Install AVR toolchain (Corne rev1 is AVR)

If you compile `crkbd/rev1` and see `avr-gcc: not found`, install:

```bash
sudo apt install -y gcc-avr avr-libc binutils-avr avrdude
```

---

## 4) Create my keymap folder

```bash
cd ~/qmk_firmware
mkdir -p keyboards/crkbd/keymaps/mxvlshn
```

Expected structure:
```text
keyboards/crkbd/keymaps/mxvlshn/
  keymap.c
  config.h
  (optional) rules.mk
```

---

## 5) Convert my JSON -> keymap.c (with vim)

### Save JSON
Example location:
```bash
vim ~/mxvlshn.json
```

Paste the JSON export from config.qmk.fm and save:
- `:wq`

### Generate `keymap.c` from JSON
```bash
qmk json2c ~/mxvlshn.json -o keyboards/crkbd/keymaps/mxvlshn/keymap.c
```

This step converts the JSON layout/layers into a C keymap source file.

---

## 6) Add firmware settings via `config.h` (Home-row mods tuning)

Create/edit:
```bash
vim keyboards/crkbd/keymaps/mxvlshn/config.h
```

Recommended starting setup (good for fast typing; avoids “mods all the time”):
```c
#pragma once

// Base tapping term (adjust after testing)
#define TAPPING_TERM 175
#define TAPPING_TERM_PER_KEY

// Home-row mods: prevents same-hand rolls from turning into holds
#define CHORDAL_HOLD

// Allows intended opposite-hand chords to still register as holds
#define PERMISSIVE_HOLD

// Optional: can reduce accidental holds during fast typing
#define FLOW_TAP_TERM 150

// IMPORTANT: do NOT enable this for fast typing HRMs, it causes constant accidental holds
// #define HOLD_ON_OTHER_KEY_PRESS
```

Notes:
- In newer QMK, `IGNORE_MOD_TAP_INTERRUPT` is no longer needed and will fail compilation if defined.
- If you still get accidental mods: increase `TAPPING_TERM` to 190–220.
- If holds feel hard to activate intentionally: decrease `TAPPING_TERM` slightly (165–175) or tweak per-key terms.

---

## 7) Update `keymap.c` for Chordal Hold (layout map)

Chordal Hold needs a handedness map to decide what “same-hand” vs “opposite-hand” means.

Open:
```bash
vim keyboards/crkbd/keymaps/mxvlshn/keymap.c
```

Add this **outside of any function** (top or bottom is fine):

```c
const char chordal_hold_layout[MATRIX_ROWS][MATRIX_COLS] PROGMEM =
    LAYOUT_split_3x6_3(
        'L','L','L','L','L','L',  'R','R','R','R','R','R',
        'L','L','L','L','L','L',  'R','R','R','R','R','R',
        'L','L','L','L','L','L',  'R','R','R','R','R','R',
                     '*','*','*',  '*','*','*'
    );
```

`'*'` exempts thumbs from the “opposite hands” rule so thumb chords don’t get weird.

---

## 8) (Optional) Per-key tapping terms for home-row mods

If you want tighter control, add to the bottom of `keymap.c`:

```c
uint16_t get_tapping_term(uint16_t keycode, keyrecord_t *record) {
    switch (keycode) {
        // left home row
        case LGUI_T(KC_A):
        case LALT_T(KC_S):
        case LCTL_T(KC_D):
        case LSFT_T(KC_F):
        // right home row
        case RSFT_T(KC_J):
        case RCTL_T(KC_K):
        case RALT_T(KC_L):
        case RGUI_T(KC_SCLN):
            return 170;   // tighter home-row
        default:
            return TAPPING_TERM;
    }
}
```

Tuning tips:
- If accidental mods still happen: try 165.
- If holds don’t activate when you want: try 175–185.

---

## 9) Compile to `.hex` locally

From QMK repo root:

```bash
cd ~/qmk_firmware
qmk compile -kb crkbd/rev1 -km mxvlshn
```

Output file will be copied to repo root as:
```text
crkbd_rev1_mxvlshn.hex
```

That is your “JSON → HEX” pipeline:

1. JSON exported from Configurator  
2. `qmk json2c` generates `keymap.c`  
3. `config.h` + `keymap.c` contain tuning/code  
4. `qmk compile` produces `.hex`

---

## 10) Flashing

### Recommended: QMK Toolbox on Windows
Copy hex to Windows (replace username):
```bash
cp ~/qmk_firmware/crkbd_rev1_mxvlshn.hex /mnt/c/Users/<YOUR_WINDOWS_USER>/Downloads/
```

Then:
1. Open **QMK Toolbox**
2. Select the `.hex`
3. Put keyboard half into bootloader (Reset button)
4. Flash

Repeat for the other half if needed.

### Alternative: flash via QMK CLI (if USB works in WSL)
```bash
qmk flash -kb crkbd/rev1 -km mxvlshn
```

(WSL2 sometimes needs USB pass-through; Windows flashing is usually simpler.)

---

## 11) Quick troubleshooting

### `avr-gcc: not found`
Install AVR toolchain:
```bash
sudo apt install -y gcc-avr avr-libc binutils-avr avrdude
```

### QMK error about `IGNORE_MOD_TAP_INTERRUPT`
Remove it from `config.h`. Newer QMK makes that behavior default.

### Home-row mods triggering constantly
Make sure you **did not** enable:
```c
#define HOLD_ON_OTHER_KEY_PRESS
```
Use `CHORDAL_HOLD` + `PERMISSIVE_HOLD` instead, and tune `TAPPING_TERM`.

---

## 12) Useful commands recap

```bash
# setup
qmk setup
qmk doctor

# convert JSON -> keymap.c
qmk json2c ~/mxvlshn.json -o keyboards/crkbd/keymaps/mxvlshn/keymap.c

# compile -> hex
qmk compile -kb crkbd/rev1 -km mxvlshn
```

---

## Notes for future me
- If I switch to Corne rev4, compile/flash target changes to `crkbd/rev4`.
- Keep HRM tuning in `keyboards/crkbd/keymaps/mxvlshn/config.h` + `keymap.c` so it’s version-controlled.
