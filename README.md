# USB Rubber Ducky (Raspberry Pi Pico)

A DIY **USB Rubber Ducky** — a keystroke-injection (BadUSB) device built on a **Raspberry Pi Pico / Pico W** running CircuitPython. When plugged into a target computer, the Pico identifies itself as a USB keyboard and automatically "types" a pre-written script (a *payload*) at machine speed, letting you automate any sequence a keyboard could perform.

This project is built on the open-source **[Pico-Ducky](https://github.com/dbisu/pico-ducky)** firmware by Dave Bailey (dbisu / @daveisu). On the Pico W it also exposes a small Wi-Fi web interface for editing and running payloads remotely.

> ⚠️ **For education and authorised security testing only.** Keystroke-injection tools are powerful. Use this only on computers you own or have **explicit written permission** to test. Unauthorised access to computer systems is illegal in most jurisdictions. See [Legal & Ethical Use](#legal--ethical-use).

---

## Table of Contents

- [What It Does](#what-it-does)
- [How It Works](#how-it-works)
- [Hardware](#hardware)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [Writing Payloads (Ducky Script)](#writing-payloads-ducky-script)
- [The Example Payload](#the-example-payload)
- [Pico W Web Interface](#pico-w-web-interface)
- [Hardware Controls (GPIO pins)](#hardware-controls-gpio-pins)
- [Legal & Ethical Use](#legal--ethical-use)
- [Credits](#credits)

---

## What It Does

- Emulates a **USB HID keyboard** (and consumer-control/mouse) via Adafruit's HID library.
- Reads a payload file written in **Ducky Script** and replays it as keystrokes.
- Supports **multiple payloads**, selectable with jumper pins, plus a trigger button.
- On the **Pico W**, serves a web page so you can view, edit, create and run payloads over Wi-Fi.
- Can **hide its own USB drive** from the host so the payload storage isn't visible.

---

## How It Works

```
Plug Pico into host ──► CircuitPython runs code.py
        │
        ├─ boot.py decides whether the USB mass-storage drive is visible (GP15)
        │
        ├─ code.py checks setup mode (GP0)
        │       │
        │       ├─ setup mode  ──► wait for payload updates (drive stays editable)
        │       │
        │       └─ run mode    ──► selectPayload() picks a .dd file (GP4/5/10/11)
        │                               │
        │                               └─ runScript() → parseLine() interprets
        │                                     each Ducky Script line → HID keystrokes
        │
        └─ (Pico W only) start Wi-Fi access point + web server for remote editing
```

1. **Boot (`boot.py`).** Reads pin **GP15** to decide whether the Pico's storage shows up as a USB drive on the host. This lets you hide the payload files during an attack while still being able to re-mount them for editing. The default differs between the Pico and Pico W.
2. **Main (`code.py`).** Sleeps briefly so the host enumerates the device, disables auto-reload, sets up the LED, and checks **GP0** for *setup/programming mode*. If not in setup mode it selects and runs a payload; on the Pico W it also brings up Wi-Fi and the web service via asyncio tasks.
3. **Interpreter (`duckyinpython.py`, shipped in the zip).** Parses Ducky Script line by line (`REM`, `DELAY`, `STRING`, `GUI`, `ENTER`, `REPEAT`, `LED`, etc.) and converts each into Adafruit HID keycodes that are sent to the host as real keypresses.
4. **Web app (`webapp.py` + `wsgiserver.py`).** A tiny WSGI server (Adafruit's, adapted) serving routes to list (`/ducky`), create (`/new`), edit (`/edit/<file>`), write (`/write/<file>`) and run (`/run/<file>`) payloads from a browser on the Pico W's access point.

---

## Hardware

- **Raspberry Pi Pico** or **Raspberry Pi Pico W** (the W adds Wi-Fi + the web interface)
- A USB cable / adapter to connect to the target
- *(Optional)* jumper wires or switches on the GPIO pins below for payload selection, a trigger button, and USB-hiding

---

## Project Structure

| File | Description |
|---|---|
| `boot.py` | Runs at boot; uses **GP15** to show/hide the USB mass-storage drive. |
| `code.py` | Main entry point; setup-mode check, payload execution, and (Pico W) Wi-Fi + web service. |
| `webapp.py` | WSGI web application: list / create / edit / run payloads from a browser (Pico W). |
| `wsgiserver.py` | Lightweight WSGI server (Adafruit, MIT) that `webapp.py` runs on. |
| `payload.dd` | The default payload in Ducky Script (example: runs an Nmap scan via `cmd`). |
| `Rubber ducky project.zip` | Full deployable bundle: the above files, `duckyinpython.py` (the Ducky Script interpreter), and the `lib/` CircuitPython libraries (`adafruit_hid`, `adafruit_wsgi`, `asyncio`, etc.). |

> The core interpreter `duckyinpython.py` and the `lib/` dependencies live **inside the zip** — that zip is what you actually copy onto the Pico.

---

## Installation

1. **Flash CircuitPython** onto the Pico/Pico W
   - Hold **BOOTSEL**, plug the Pico in; it mounts as `RPI-RP2`.
   - Download the CircuitPython `.uf2` for your board from [circuitpython.org](https://circuitpython.org/) and copy it over. The Pico reboots as a `CIRCUITPY` drive.

2. **Copy the project files** onto the `CIRCUITPY` drive
   - Unzip `Rubber ducky project.zip` and copy its entire contents — `boot.py`, `code.py`, `duckyinpython.py`, `webapp.py`, `wsgiserver.py`, `payload.dd`, and the whole `lib/` folder — to the root of `CIRCUITPY`.

3. **Add your payload**
   - Edit `payload.dd` (or add `payload2.dd` … `payload4.dd`) with your Ducky Script. See below.

4. **Eject and deploy**
   - Safely eject the drive and unplug the Pico. On the next plug-in (into a target you're authorised to test), it enumerates as a keyboard and runs the selected payload.

> **Tip (setup vs. run mode):** connect **GP0 to GND** to boot into setup/programming mode so the Pico *doesn't* fire the payload while you're editing it.

---

## Writing Payloads (Ducky Script)

Payloads are plain-text `.dd` files. Commands supported by this build's interpreter include:

| Command | Meaning |
|---|---|
| `REM <text>` | Comment (ignored). |
| `DELAY <ms>` | Wait the given milliseconds. |
| `DEFAULT_DELAY <ms>` / `DEFAULTDELAY <ms>` | Delay inserted between every subsequent command. |
| `STRING <text>` | Type the text literally. |
| `ENTER`, `GUI`, `CTRL`, `ALT`, `SHIFT`, `TAB`, arrow keys, F-keys, etc. | Press those keys / combos (e.g. `GUI r` = Win+R). |
| `PRINT <text>` | Print to the serial console (debug). |
| `IMPORT <file>` | Run another script file. |
| `LED` | Toggle the onboard LED. |
| `REPEAT <n>` | Repeat the previous line `n` times. |

A minimal example:

```text
DELAY 1000
GUI r
DELAY 500
STRING notepad
ENTER
DELAY 1000
STRING Hello from the Pico Ducky
```

---

## The Example Payload

The included `payload.dd` demonstrates the format by launching a Windows command prompt and running an **Nmap** scan against the local gateway, saving the result to a file:

```text
DELAY 2000
REM Open Run dialog
DELAY 500
GUI r
DELAY 500
REM Type "cmd" and press Enter
STRING cmd
ENTER
DELAY 2000
REM Type the correct Nmap command and press Enter
STRING nmap -A -T4 -oN C:\nmap_scan.txt 192.168.0.1
ENTER
DELAY 5000
STRING exit
ENTER
```

It assumes Nmap is installed and on the host's `PATH`. It's a harmless demonstration of keystroke automation — swap in your own authorised-testing payload as needed.

---

## Pico W Web Interface

On a **Pico W**, `code.py` starts a Wi-Fi **access point** (SSID/password read from a `secrets.py` file you create) and a web server. Connect to that AP and open the printed IP in a browser to:

- **List** all `.dd` payloads on the device (`/ducky`)
- **Create** a new script (`/new`)
- **Edit** an existing one (`/edit/<filename>`)
- **Run** a payload on demand (`/run/<filename>` or `/api/run/<number>`)

Create `secrets.py` on the Pico with:

```python
secrets = {
    "ssid": "picoducky",
    "password": "a-strong-password",
}
```

---

## Hardware Controls (GPIO pins)

Connect a pin to **GND** to activate it:

| Pin | Function |
|---|---|
| **GP0** | Setup / programming mode (payload does **not** run). |
| **GP15** | USB-drive visibility toggle (behaviour differs Pico vs. Pico W — see `boot.py`). |
| **GP4** | Select `payload.dd`. |
| **GP5** | Select `payload2.dd`. |
| **GP10** | Select `payload3.dd`. |
| **GP11** | Select `payload4.dd`. |
| **GP22** | Trigger button (momentary). |

---

## Legal & Ethical Use

This project exists for **learning about USB/HID security, BadUSB attack vectors, and defensive awareness**. Keystroke-injection devices can do anything the logged-in user can do, very quickly and silently.

- Only use it on **your own devices** or systems you have **explicit, written authorisation** to test.
- Never use it to access, disrupt, or exfiltrate data from systems without permission.
- You are solely responsible for complying with the computer-misuse and privacy laws in your jurisdiction.

The authors and contributors provide this for educational purposes and accept no liability for misuse.

---

## Credits

- **[Pico-Ducky](https://github.com/dbisu/pico-ducky)** by Dave Bailey (dbisu / @daveisu) — the CircuitPython firmware this project is based on (GPLv2.0).
- **[Adafruit](https://github.com/adafruit)** — CircuitPython, `adafruit_hid`, and the WSGI server (`wsgiserver.py`, MIT).
- Original **USB Rubber Ducky** concept by Hak5.
