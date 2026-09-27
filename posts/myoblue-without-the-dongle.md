# Throwing away the dongle

### My EMG sensors came with a USB dongle I had to carry everywhere. It turned out to be a relay for ordinary Bluetooth LE packets, and everything it does fits in a few lines of code.

---

I bought a pair of ELEMYO MYOblue v1.2 sensors to measure my triceps in the gym. They're small
dry-electrode EMG sensors, about the size of a large coin, and they come with a USB dongle and a
Python GUI. The intended setup is: plug the dongle into a laptop, open the GUI, watch the muscle signal.

That's fine at a desk. It's useless in a gym, where I didn't want to carry a laptop around between
cable machines. What I wanted was the sensors talking to my phone, recording in my pocket, with
nothing else in between.

The first question was whether that was even possible, or whether the dongle was doing something
clever the sensors depended on.

## What the dongle actually is

The dongle enumerates as a USB serial port (CDC-ACM) on an nRF52, Nordic's Bluetooth LE chip. That was
already a hint: nRF52 is what you'd pick to build a Bluetooth relay, not a proprietary radio.

With the dongle unplugged and a sensor switched on, a plain Bluetooth scan shows it:

```
1_MYOblue_v1.2_XXXXX
```

The number at the front is the module number, and the scan response advertises the
**Nordic UART Service** (NUS), `6e400001-b5a3-f393-e0a9-e50e24dcca9e`. NUS is Nordic's stock "serial
over Bluetooth" service: one characteristic you write to, one that sends notifications to you. Nothing
custom.

To be sure the dongle wasn't doing anything more than that, I watched it work with an HCI trace (`btmon`)
while it connected. The whole handshake is:

1. Connect.
2. Ask for a bigger MTU (247) and data length (251).
3. Discover the NUS service.
4. Turn on notifications on the TX characteristic.

No pairing. No bonding. No writes to the sensor at all. The dongle just listens, and forwards each
notification to the serial port with two `0xFF` bytes in front. The sensors don't check the dongle's
address either; anything that connects and enables notifications gets the data.

So the dongle is a relay. Anything with Bluetooth LE can replace it.

## The packet

Each notification is 244 bytes:

| Bytes | Meaning |
|---|---|
| 1 | module number |
| 3 | packet sequence number (little-endian) |
| 2 | battery (V = raw / 16384 × 7.2) |
| 238 | 119 samples, 16-bit little-endian, 14-bit values, 8192 = zero |

Microvolts are `(raw − 8192) × 0.30518`, the same constant ELEMYO's GUI uses.

Two things surprised me.

**The sample rate isn't 1000 Hz.** Nominally it is. But fitting the sequence number against arrival
time over a few minutes gives about 975 samples per second: the sensor's clock runs 2–3% slow. It's the
same with or without the dongle, so it's the sensor, not the radio. If you assume 1000 Hz, your time
axis drifts by about 1.5 seconds per minute. Everything I built since fits the clock from the sequence
numbers instead of trusting the nominal rate.

**Once a minute, a packet is all zeros.** Every sample is exactly 8192. It turns out the sensor measures
its battery instead of EMG in that slot. If you don't know that, you see a mysterious flat line in the
signal every sixty seconds. Now it's treated as missing data.

## The part that took the longest: Linux wouldn't connect

The first attempt to connect from my laptop failed. Every time, with the same error:

```
HCI error 0x3e: Connection Failed to be Established
```

The sensor was clearly advertising and clearly reachable. The dongle connected instantly. My laptop,
using BlueZ, never did.

The HCI trace from the dongle had the answer. The dongle asks for a **30 ms connection interval with a
4-second supervision timeout**. BlueZ's defaults are a 45 ms interval with a 420 ms timeout. The
sensors apparently can't hold a link with a timeout that short: the connection drops before it's fully
set up, which is exactly what `0x3e` means.

The fix is three lines in `/etc/bluetooth/main.conf`, in the `[LE]` section that's already there with
the keys commented out:

```ini
[LE]
MinConnectionInterval=24
MaxConnectionInterval=24
ConnectionSupervisionTimeout=400
```

(The units are 1.25 ms and 10 ms, so that's 30 ms and 4 s: the dongle's numbers.)
After `systemctl restart bluetooth`, it connected first time, every time. Android and Windows already
use longer timeouts, which is why phones never had this problem.

One more quirk worth knowing: the sensors advertise as *limited discoverable*. An unconnected sensor goes
quiet after a few minutes, and you have to power-cycle it. More than once I thought something was broken,
and the sensor had simply stopped advertising.

## Three ways to use it without the dongle

Once the protocol was clear, I built the replacement three times, for three different situations.

### 1. A software dongle for ELEMYO's own GUI

I didn't want to give up the GUI that came with the sensors. It reads from a serial port, so the simplest
move was to give it one.

A small Python bridge connects to all the sensors over Bluetooth (with `bleak`), and writes
`FF FF` + each packet to a **pseudo-terminal**, byte for byte what the real dongle sends. A launcher
puts that pty first in the GUI's port menu. There was one snag: the GUI toggles DTR and RTS on the port,
which a pty rejects, so the launcher patches those calls to do nothing. The GUI's own code doesn't change
at all.

I also forked the GUI and added a **"Bluetooth (no dongle)"** entry to its port list, which talks to the
sensors directly and skips the bridge. That's the version I actually use on the laptop now.

### 2. A web page

Chrome's Web Bluetooth can do everything the dongle does: connect, request notifications, receive the
244-byte packets. So I wrote a small static web app that records, calibrates and exports CSV. It runs
on a phone, needs no install, and works anywhere with HTTPS.

It has one fatal flaw for my purpose: browsers pause pages that aren't on screen. Lock the phone, take a
call, switch apps, and the recording stops. For a gym, that's a deal-breaker.

### 3. An Android app that records in the background

So the last version is an Android app. The UI is React Native (Expo), but the Bluetooth connection and
the file writing live in a small native Kotlin module, running inside an Android **foreground service**:
the kind with a permanent notification (mine has Mark and Stop buttons). That means recording doesn't
depend on the UI at all. The screen can be off, a call can come in, I can switch to music, and the packets
keep arriving and getting written to disk.

Each recording is an append-only file per sensor. Every row is the phone's timestamp plus the raw
244 bytes, exactly as they came off the air. I never throw away the raw data, which turned out to matter
a lot later, when I kept finding better ways to analyse the same recordings.

## What I'd tell someone with the same sensors

- **The dongle is optional.** The sensors are ordinary Bluetooth LE peripherals using a standard Nordic
  service. Unplug the dongle first, though: if it's connected, it grabs the sensors before anything else
  can.
- **On Linux, fix the BlueZ timing first.** A 30 ms interval and a 4 s supervision timeout. Without that,
  nothing connects and the error message doesn't tell you why.
- **Don't trust the nominal sample rate.** Fit it from the sequence numbers.
- **Expect a flat packet once a minute.** It's the battery reading.
- **For anything mobile, use a native background service,** not a web page.

The code for all three (the bridge, the web app and the Android app) is in
[myoblue-ble](https://github.com/MohitBurkule/myoblue-ble), and the GUI fork is
[MYOblue-GUI](https://github.com/MohitBurkule/MYOblue-GUI).

The more interesting part came next: once the signal was on my phone, what could it actually tell me
about my training? That's the [next post](myolift-emg-gym-app.md).
