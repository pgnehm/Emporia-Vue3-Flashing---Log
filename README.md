# Flashing an Emporia Vue 3 — Chasing Milliwatts

*How I spent way more hours than I'd like to admit proving that a two-dollar USB adapter cannot power a Wi-Fi radio — and everything else that broke on the way to flashing an Emporia Vue 3 with open firmware, so I can finally see my panel circuit by circuit.*

**Service:** 3-phase, 240V · Peru
**Firmware:** ESPHome (community)
**Status:** Bench-tested, not installed yet

No project code lives in this repo — it's just the write-up.

---

## Foreword: Before You Read This as My Work Alone

I could not have done any of this without AI. Claude walked me through every step of it — every command, every config edit, every diagnosis below was Claude's, not mine. I don't have deep ESP32 or embedded firmware knowledge, and I didn't have the time to go acquire it just to get sixteen circuits into Home Assistant. My side of it was soldering, plugging things in, taking pictures of the board when something looked wrong, and pasting back whatever the terminal showed me. That feedback loop — screenshot or paste, wait, respond — worked well. Everywhere below that reads like "I flashed" or "I fixed," read it as Claude doing the keyboard work while I did the soldering iron work.

That doesn't mean I checked out and let it run. At one point during the power debugging, it got lost chasing its own ideas — more capacitors, then more capacitors again, past the point the data had already answered the question — and I had to stop it, step back, and ask what problem we were actually trying to solve. That's what actually got us to the batteries. I also had to keep pushing for plain, honest summaries of what was going on electrically, instead of just trusting that a command had "worked." **You have to stay hands-on with the tool for this to actually succeed** — it's a collaborator you're directing, not a machine you hand the keys to and walk away from.

This isn't my first finished project with Claude, either — I've completed others the same way, and each one opened up something I'd have had no real path to on my own. You still need some baseline electrical and programming literacy, enough to know when an answer sounds wrong. But the gap between that baseline and actually flashing a custom energy monitor is exactly what AI closed for me here. All of it is headed into Home Assistant once the panel work is done.

## Panel Directory

1. [Cracking the Case](#01--cracking-the-case) — Resolved
2. [The Backup Nobody Wants to Need](#02--the-backup-nobody-wants-to-need) — Resolved
3. [Three Bugs Before the First Byte](#03--three-bugs-before-the-first-byte) — Resolved
4. [A 400-Millisecond Life](#04--a-400-millisecond-life) — Tripped
5. [The Wire That Wouldn't Stay Put](#05--the-wire-that-wouldnt-stay-put) — Partial
6. [Silencing the Alarm Isn't Fixing the Fire](#06--silencing-the-alarm-isnt-fixing-the-fire) — Tripped
7. [Two AA Batteries](#07--two-aa-batteries) — Resolved
8. [The Feature I Didn't Know I Needed](#08--the-feature-i-didnt-know-i-needed) — Resolved
9. [What's Still Live](#09--whats-still-live) — Open

---

### 01 — Cracking the Case

I've got an Emporia Vue 3 sitting in a panel here in Peru, on 3-phase, 240V service, and what I actually want out of it is circuit-by-circuit visibility — not just a total kWh number, but every one of the sixteen breakers I'm monitoring, separately, in Home Assistant. That meant local open-source firmware instead of phoning home to Emporia's cloud: faster updates, works offline, and I can automate on the data without asking anyone's permission. The community that maintains the ESPHome component for this board has a whole [tutorial site and software](https://emporia-vue-local.github.io/) for exactly this, so that's where we started: back up the factory firmware first, then flash.

The Vue 3 doesn't give you a header to clip onto. You solder directly to five bare test pads on the board: RXD, TXD, GND, an IO0 pad you jumper to ground to force the chip into its bootloader, and a 3.3V pad to power the thing while it's out of the panel. I don't own a fine-tip soldering iron suited to pads the size of a grain of rice — I used the pointiest iron I had and just kept at each pad until the wire held. Barely. That word is going to matter later.

I got the wiring backwards exactly once, sent a photo, and had it corrected before anything got plugged in. Small win, but I'll take it — this is the kind of mistake that, done with real mains voltage instead of 3.3V, doesn't get a do-over.

> **Note to self:** Five wires, barely holding, and every single one of them will cause a problem later in this story. Keep reading.

### 02 — The Backup Nobody Wants to Need

Before touching the factory firmware, I pulled a full copy of it with `esptool`. This is the boring, unglamorous step everyone tells you not to skip, and everyone is right. If this whole experiment goes sideways, that 8MB file is the only way back to a working, Emporia-supported device.

Windows did actually have a driver for my cheap CH340-based adapter — it just sat there for a while with a sad little warning triangle in Device Manager before it got around to recognizing it. Gave it a minute, and it came up clean as a real COM port.

```
esptool v5.3.0
Connected to ESP32 on COM5:
Chip type:          ESP32-D0WD-V3 (revision v3.1)
...
A serial exception error occurred: ClearCommError failed
-- computer went to sleep mid-transfer, at 41.5% --
Read 8388608 bytes ... Hash of data verified.
```

The first attempt died at 41% because my laptop quietly went to sleep in the middle of the transfer — eight minutes of watching a progress bar is apparently long enough for Windows to decide I'd stepped away. Disabled sleep, ran it again, full 8MB, hash verified. Backup done, filed away, never touched again.

### 03 — Three Bugs Before the First Byte

With the backup safe, I tried to flash my actual config — sixteen circuits, split-phase sensing, the works — and hit three completely unrelated landmines before a single byte reached the board.

**Bug one:** my YAML used a top-level `filters:` key purely to hold some reusable anchors. ESPHome's loader saw an unrecognized top-level key and tried to import it as an actual component named "filters." It doesn't exist. Fix: prefix it with a dot — `.filters:` — which ESPHome silently ignores, exactly the escape hatch it exists for.

**Bug two:** my project lived under a OneDrive folder with spaces in the path. ESP-IDF's build system flatly refuses to build from a path containing spaces. Moved the whole project to `C:\esphome-projects\` and the build proceeded.

**Bug three** was the one that actually cost time. I was running everything through Git Bash, which quietly sets an environment variable (`MSYSTEM`) that ESP-IDF's installer checks for — and refuses to install its own toolchain if it finds it. Every compiler, every binary it needed, silently never downloaded. No error that pointed at the real cause, just a cascade of "file not found" further down the line. Switching to plain PowerShell fixed it in one shot.

> **Note to self:** None of these three bugs were about the Emporia hardware at all. They were all just Windows being Windows. First real compile, first real flash, first boot — all clean.

### 04 — A 400-Millisecond Life

The board booted. For about four tenths of a second.

```
E BOD: Brownout detector was triggered
ESP-IDF 5.5.4 2nd stage bootloader
chip revision: v3.1 ...
E BOD: Brownout detector was triggered
ESP-IDF 5.5.4 2nd stage bootloader
… repeats forever, once every ~390ms …
```

The ESP32's brownout detector exists to force a clean reset the instant supply voltage sags below a safe threshold, rather than let the chip run on marginal power and corrupt something. Mine was tripping in a tight loop, over and over, before it ever got far enough to try Wi-Fi.

Claude's working theory: the tiny linear regulator soldered onto the [two-dollar USB-serial adapter](https://www.amazon.com/dp/B00LZV1G6K?ref=ppx_yo2ov_dt_b_fed_asin_title&th=1) I was using is built to blink an LED, not run an ESP32 through a Wi-Fi radio's current draw. We tried a different USB port on the theory that a beefier port might help. Identical loop, right down to the timing. Not a port problem.

### 05 — The Wire That Wouldn't Stay Put

While chasing the power problem, the board went completely silent — not even the brownout loop, just nothing on the serial monitor. That's a different, worse symptom: it usually means the wire carrying the board's transmit signal back to my adapter has lost contact, not that the firmware is misbehaving.

Sure enough, the White wire — hand-soldered to a bare pad the size of a grain of rice — had worked itself loose. Re-soldered it. Fine for a while. Then, a few reflashes later, silence again. Same wire, same pad. Twice was enough to learn the lesson: this time I added a small anchor of hot glue a few millimeters back from the joint, so any tug on the wire pulls against the glue instead of the solder.

> **Note to self:** A loose wire and a starved power supply produce almost the same silence on a serial monitor. Don't assume it's the interesting bug; check the boring one first.

### 06 — Silencing the Alarm Isn't Fixing the Fire

With the wire solid again, I still had a board that reset itself four times a second the moment Wi-Fi tried to come up. So I ran an experiment: disable the brownout detector entirely in firmware, just to see what happened underneath the reset loop it was hiding.

That's a genuinely bad idea to leave in place — the detector exists specifically to stop corrupted writes during real undervoltage, and this board is going to sit in an electrical panel unattended. But as a diagnostic, for one afternoon, it was useful: instead of a clean reset every 390ms, I got total silence again. The chip wasn't resetting anymore. It was just dying quietly instead.

So I stripped the config down further — pulled Wi-Fi out entirely, just to see if the ESP32 itself could run stably without that specific current spike.

```
[12:36:55] I2C error 2 (repeats every ~200ms, no crash)
… 24 seconds and counting, still running …
```

Rock solid. Twenty-four straight seconds with zero resets — the longest the board had stayed alive since I'd started. That one result told me the ESP32, the wiring, and the firmware logic were all completely fine. The *only* unstable thing was Wi-Fi's power draw meeting a supply that couldn't keep up. That I2C error, incidentally, was a second, unrelated discovery: the onboard metering chip lives on what I'm fairly sure is a completely separate power rail, one that's only ever energized by the real mains-derived supply — meaning it was never going to respond on a bench, no matter what I fed the ESP32.

Naturally, I reached for a capacitor next — a few hundred microfarads across the 3.3V rail, then a few more in parallel once one wasn't enough, eventually stacking three together for close to 1400µF. Same loop, completely unchanged. That result mattered: a capacitor only survives a brief spike by recharging between spikes. Wi-Fi's elevated draw during association isn't a millisecond blip, it's sustained — and no amount of extra capacitance out-runs a source that's fundamentally too weak to keep up in the first place.

I even looked into whether I could power it from Emporia's own supply. Turns out there isn't one — this panel-mount model has no wall adapter at all; by design it draws its own operating power straight out of the same harness that taps your breakers for voltage sensing. I have two phases and a ground available on my bench from a three-phase source, no neutral, and Emporia's documented wiring always needs a real neutral on that harness, never a ground substitute. That door stayed shut, for good reason.

### 07 — Two AA Batteries

No spare bench supply, but I did have a cheap electronics assortment kit with a stack of capacitors, resistors, and voltage regulators in it — and two ordinary AA batteries. Two in series gives roughly 3V, just inside the ESP32's tolerance, and batteries have something a $2 regulator doesn't: very low internal resistance, meaning they don't sag under a sudden current pull the way a cheap linear regulator does.

Wired the battery pack straight to the 3.3V and ground pads, powered on, and watched the status LED go from a fast, anxious blink to a slow, steady glow — the visual signature of a device that connects once and stays connected, instead of retrying in a loop.

```
INFO Successfully resolved emporia-vue.local in 0.157s
INFO Successfully connected to emporia-vue @ 192.168.x.x in 0.051s
[C][wifi] Connected: YES  Signal strength: -24 dB
```

Two loose AA batteries, held together with a piece of tape, out-performed every piece of purpose-built power electronics I'd thrown at this all night. Home Assistant picked the device up automatically a few minutes later.

### 08 — The Feature I Didn't Know I Needed

With Wi-Fi finally solid, I started thinking about the real installation, and realized my panel is three-phase, and I'd only wired the config for two legs. Adding a third phase to the YAML was mechanical. What I actually wanted was something better: the ability to tell the device which phase each of my sixteen circuits sits on *after* it's already installed, from inside Home Assistant, without ever touching a soldering iron again.

Went and read the actual C++ behind the ESPHome component to see if that was even possible, and found something I hadn't expected: the metering chip already computes power against **all three phases, for every single circuit, on every single reading**. The YAML doesn't choose which phase to measure — it just chooses which of three numbers, already sitting in memory, to report. That meant I didn't need to touch the component's source at all. I could point three separate sensor entries at the very same physical input, one per phase, and build a Home Assistant dropdown that just switches which of the three it's currently reading from.

> **Note to self:** Sixteen dropdowns, one per circuit, right in the Home Assistant device page. Guess wrong during installation, fix it from your phone. No re-flash required.

### 09 — What's Still Live

Nothing is in the panel yet. The firmware is done and tested, living on batteries on my bench, ready to go — but the board itself hasn't been installed. What's left is the actual electrical work: current transformers on the mains and on all sixteen branch circuits I actually care about, and the voltage-sensing harness wired into real breakers.

That harness is where it gets interesting. I already know my panel has three phases and a ground — and no neutral. Emporia's documented wiring for this exact harness always calls for a real neutral on its white wire, in every configuration they publish, single-phase or three. Ground isn't a substitute; it's not meant to carry that current, and I'm not improvising around that on a live panel. So this is a real problem I get to solve at install time, not a bench problem — and I don't have the answer yet.

Everything past this point is mains voltage, and it's getting handed to a licensed electrician — which, happily, is a box I get to check myself. Once it's actually in the panel, this is the part I've been after the whole time: sixteen circuits, each one visible on its own, instead of one number for the whole house.

---

> ⚠️ **Before you copy any of this:** Everything in this log happened at 3.3V on an open bench. The moment a Vue enters an electrical panel, you are working next to energized service mains, full stop. That step should be done by a licensed electrician, in accordance with local code — not improvised the way a bench power supply can be.

*Field log · written after the fact, mistakes and all*
