# Prusa Firmware MK3 — PT1000 fork (unofficial)

This is a **personal, unofficial fork** of [prusa3d/Prusa-Firmware](https://github.com/prusa3d/Prusa-Firmware), modified for one specific Original Prusa MK3S+ printer to support a PT1000 hotend sensor wired directly to the stock thermistor input. It is **not affiliated with, endorsed by, or supported by Prusa Research**.

**Flash this at your own risk.** It changes hotend temperature limits and sensor behavior. If you don't understand the changes below and haven't verified your own hardware, use the official firmware from [Prusa Drivers](https://www.prusa3d.com/drivers/) instead.

Licensed under [GPL-3.0](LICENSE), same as upstream. Credit to [Prusa Research](https://prusa3d.com/) and [Marlin](https://github.com/MarlinFirmware/Marlin/) (by Scott Lahteine / @thinkyhead et al.), whose work this is built on.

Branch `pt1000-410c`, based on upstream tag `v3.14.1`. See `git log` and `git diff v3.14.1` for the exact changes.

## What's changed

- **`Firmware/variants/MK3S.h`**: `PT1000_EXTRUDER` enabled → `TEMP_SENSOR_0 1047`, `HEATER_0_MAXTEMP 420`. Assumes a PT1000 wired directly to the Einsy hotend thermistor input using the stock 4.7k pullup, no amplifier board. (Upstream's PT100 options keep `HEATER_0_MAXTEMP 410`; stock thermistor stays at 305.)
- **`Firmware/thermistortables.h`**: `temptable_1047` extended up to 450C. Added a `HEATER_0_RAW_HI_TEMP 16383` / `HEATER_0_RAW_LO_TEMP 0` override for sensor 1047. PT1000 is a PTC (resistance rises with temperature) but the NTC-oriented default direction meant MINTEMP/MAXTEMP safety checks never tripped for this sensor. Upstream's PT100 options (148/247) appear to have the same latent issue.
- **Mesh bed leveling is unchanged in firmware.** Skipping it is done in PrusaSlicer, not here — see [Skipping bed leveling](#skipping-bed-leveling) below.

## Build (Linux)

Tested on a system with no working system `pip`. `./utils/bootstrap.py`'s pip step fails in that case, so a venv is used instead:

```sh
git clone https://github.com/zefangli/MK3S-PT1000-firmware.git
cd MK3S-PT1000-firmware

# download avr-gcc, cmake, ninja, prusa3dboards into .dependencies/
./utils/bootstrap.py

# work around missing system pip
python3 -m venv --without-pip .venv
curl -sS https://bootstrap.pypa.io/get-pip.py | .venv/bin/python
.venv/bin/pip install pyelftools polib regex

export PATH="$PWD/.venv/bin:$PWD/.dependencies/cmake-3.22.5/bin:$PWD/.dependencies/ninja-1.12.1:$PATH"

mkdir -p build && cd build
cmake .. -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE=../cmake/AvrGcc.cmake
ninja MK3S_MULTILANG
```

Output: `build/release/MK3S_MK3S+_FW_3.14.1_MULTILANG.hex`.

If your system `pip` already works, skip the venv workaround and follow bootstrap.py's normal flow (see [Upstream build reference](#upstream-build-reference)).

> Note: `.venv/` is **not** in `.gitignore` here — if you keep this as a git repo, either add it or avoid committing it accidentally.

## Flashing

PrusaSlicer → Configuration → Flash printer firmware → select the `.hex` above.

## Post-flash checklist

1. **Sanity-check the sensor before heating.** `M105` at room temp should read close to actual room temperature. ~450C (pegged) means open circuit or miswiring; a large negative reading means a short.
2. **Cross-check against a reference thermometer** at a mid-range temperature (~200C) before trusting the sensor at higher temps.
3. **Recalibrate the thermal model**: `M310 A F`. Don't disable thermal model protection.
4. **PID autotune** in the 380–390C range: `M303 E0 S390 C8`, then apply the results with `M301 P.. I.. D..` and save with `M500`.
   - The LCD PID-tune menu allows setting a target up to 420C — past where MAXTEMP trips. Use the `M303`/`M301` console route instead of the LCD for the actual tune.
5. **Keep nozzle targets at or below 400C.**
   - MAXTEMP trips at ~418.6C as displayed (the last ADC step below `HEATER_0_MAXTEMP` 420).
   - `M104`/`M109` do **not** clamp targets — commanding above the cutoff heats until MAXTEMP trips, which kills the print and requires a restart.
6. **Expect the stock 40W heater cartridge may not hold 400C reliably** — it can trigger `PREHEAT ERROR` or `THERMAL RUNAWAY` if it can't keep up. Consider a higher-wattage heater if you intend to run near 400C regularly.

### Hardware caveats

- **Silicone sock**: the stock one is not rated for 400C. Use a sock rated above 400C, or run without one.
- **Heatbreak**: must be all-metal. No PTFE-lined heatbreak in the hot zone at these temperatures.
- **Wiring**: check heater and sensor wire insulation is rated for the temperatures you intend to run.
- **ADC resolution**: roughly 3C per ADC step at 400C — expect coarser temperature reporting/control near the top of the range than at normal printing temps.
- **Pullup resistor**: do **not** install the 1k pullup resistor commonly bundled with PT1000 sensors. It gives no usable gain at 400C with this wiring, and the firmware here assumes the stock 4.7k pullup.

## Skipping bed leveling

Mesh bed leveling (`G80`) is **not disabled in firmware**. The MK3 doesn't persist the mesh anyway, so without `G80` in the start G-code, only Live-Z is applied.

To skip it: in PrusaSlicer, use a separate printer profile with the `G80` line removed from Printer Settings → Custom G-code → Start G-code.

---

## Upstream build reference

The rest of this section is inherited from upstream Prusa-Firmware and covers general CMake usage, testing, and Windows/VSCode builds. It has not been changed for this fork beyond the sensor/variant edits noted above.

### Detailed CMake guide

Building with cmake requires:

- cmake >= 3.22.5
- ninja >= 1.12.1 (optional, but recommended)

Python >= 3.8 is also required with the following modules:

- pyelftools (package `python3-pyelftools`)
- polib (package `python3-polib`)
- regex (package `python3-regex`)

Additionally `gettext` is required for translators.

Assuming a recent Debian/Ubuntu distribution, install the dependencies globally with:

    sudo apt-get install cmake ninja python3-pyelftools python3-polib python3-regex gettext

`./utils/bootstrap.py` downloads a pinned `avr-gcc` and the external `prusa3dboards` package into `.dependencies/`, and will also try to install `cmake`, `ninja`, and the Python packages above if missing (via pip) — though installing those through your system's package manager, or the venv workaround above if pip is unavailable, is preferable.

By default all variants are built. There are several ways to restrict the build for development. During configuration you can set:

- `cmake -DFW_VARIANTS=variant`: comma-separated list of variants to build. This is the file name as present in `Firmware/variants` without the final `.h`.
- `cmake -DMAIN_LANGUAGES=languages`: comma-separated list of ISO language codes to include as main translations.
- `cmake -DCOMMUNITY_LANGUAGES=languages`: comma-separated list of ISO language codes to include as community translations.

Available build targets:

- `ninja ALL_MULTILANG`: build all multi-language targets (default)
- `ninja ALL_ENGLISH`: build all single-language targets
- `ninja ALL_FIRMWARE`: build all single and multi-language targets
- `ninja VARIANT_ENGLISH`: build the single-language version of `VARIANT`
- `ninja VARIANT_MULTILANG`: build the multi-language version of `VARIANT`
- `ninja check_lang`: build and check all language translations
- `ninja check_lang_ISO`: build and check all variants with language `ISO`
- `ninja check_lang_VARIANT`: build and check all languages for `VARIANT`
- `ninja check_lang_VARIANT_ISO`: build and check language `ISO` for `VARIANT`

### Automated tests

Automated tests are built with cmake by configuring for the current host:

    mkdir build && cd build
    cmake .. -G Ninja
    ninja
    ctest

### PF-build (guided script)

`PF-build.sh` is upstream's more user-friendly wrapper for casual users on Debian/Ubuntu (or derivative) distributions:

    ./PF-build.sh

This fork hasn't been tested through PF-build — the manual CMake flow above is what was actually used and verified.

### Windows / Visual Studio Code

Prerequisites: [Visual Studio Code](https://code.visualstudio.com/), the [CMake Tools plugin](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools), [Python](https://www.python.org/), [Git Bash](https://git-scm.com/downloads).

    git clone https://github.com/zefangli/MK3S-PT1000-firmware.git

Open the folder in VSCode, open a terminal (Terminal → New Terminal), and run:

    python .\utils\bootstrap.py

This downloads dependencies into `.dependencies`. Reload VSCode; it should auto-configure the CMake project. If not, `Ctrl+Shift+P` → `CMake: Select a Kit` → `avr-gcc` (scan for kits first if it's not listed), or as a fallback add `.vscode/cmake-kits.json` under CMake Tools' "Additional Kits" setting, then reload.

Build via the CMake Tools sidebar icon: find the target (e.g. `MK3S_MULTILANG`) and click Build. Output lands in `build/`.

### Arduino IDE — unsupported for this fork

Upstream marks Arduino IDE builds as deprecated (single-language only, non-reproducible, manual board-definition setup). This fork adds to that: Arduino IDE builds straight from `Firmware/Firmware.ino` only pick up this fork's PT1000 changes if you manually copy `Firmware/variants/MK3S.h` to `Firmware/Configuration_prusa.h` yourself — the CMake build does this automatically, Arduino does not. Given that extra footgun on top of an already-deprecated path, treat Arduino IDE builds of this fork as unsupported. Use the CMake flow above.
