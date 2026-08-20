# KHOJ

**A leaderless rescue swarm that finds people it cannot see.**

*("Khoj" — the search / the quest.)*

Search-and-rescue drones are blind to the victims who need them most — the ones
buried, trapped, or out of sight. KHOJ isn't. It doesn't look for victims; it
listens for the one signal a trapped person can't stop sending: their phone's
radio. Several agents share signal-strength samples over their own mesh and
**cooperatively climb the gradient to a phone none of them can see** — something
no single drone can do. A real GPS drone flies the confirmation. Kill any agent
and its work is re-auctioned in seconds, with no leader and no central computer.

> **Pitch line:** *Other swarms search together. KHOJ finds the invisible.*

The hero demo: **a judge hides their own phone; the swarm points to it within a
metre or two, having never seen it.**

---

## The one idea that makes it all click

Two worlds run at once:

- **The real world** lives on the laptop (and, for the real drone, on its RPi):
  where victims actually are, the terrain, the hidden phone. Agents never see
  this answer key.
- **Each agent's belief** lives inside its ESP32: its own private guess, built
  only from what it has personally sensed.

The laptop is the *universe*; the ESP32s are *minds* moving through it. Minds
only get sensor readings and must figure out the rest themselves. That
separation is the whole reason it's a real swarm and not a puppet show — and the
clean answer to "is the laptop secretly in charge?" is **no**.

## The novelty, in one line

The swarm bids in **information gain**, not area. Searching empty ground, taking
a second look at an uncertain sighting, and triangulating an invisible phone all
become the *same decision* — "where will my next move most reduce what the swarm
doesn't yet know?" Cooperative RF localization is the headline proof of it.

---

## Who owns what

| Owner | Track | First 6 hours |
|---|---|---|
| **A (firmware lead)** | ESP-NOW mesh, the auction/consensus, the agent loop, the on-drone ESP32. Owns `quorum_proto.h`. | Flash Module 1 (the wire) on the one board; get the echo test green. |
| **B (sim / algorithm)** | Laptop "World": 2D physics, the **RF propagation model** (simulated RSSI vs. distance to the hidden phone), the information-gain bid math, lawnmower **baseline + metrics**. | Stand up the sim loop + dashboard skeleton with 3 fake drones and a score. No hardware needed. |
| **C (drone)** | F450 + Pixhawk + RPi + **MAVSDK** waypoint flight, onboard camera detection, onboard sniffer → real RSSI → onboard ESP32. Owns the grid↔GPS mapping. | Assemble, arm, GPS lock outdoors, fly to one test waypoint via MAVSDK. |
| **D (RF + dashboard + pitch)** | Promiscuous-WiFi **RSSI sniffer** firmware, the hideable **"victim beacon"** rig, the dashboard (RF field heatmap + drones converging + the "found it" moment), rubric + pitch. | Get one board reporting phone MAC + RSSI over UART; prep a reliable beacon. |

Everyone meets at the two message formats in `quorum_proto.h`. **Nobody changes
the treaty without telling the other three.**

---

## Architecture

```
LAPTOP = "the world"           REAL DRONE (one agent, embodied)
 physics · RF sim · YOLO         F450 + Pixhawk (flies)
 dashboard · baseline            RPi: MAVSDK + camera + sniffer  <- plays "World" for real
      │  USB (per-agent            │  UART
      │  sensor packet down,       │
      │  goal up)                  ESP32 (same agent firmware)
  ┌───┴───┬───────┬───────┐        │
ESP-1   ESP-2   ESP-3   ...  ──ESP-NOW──┘     ESP-S (RF sniffer, UART only)
        bids · awards · heartbeats · RF samples
```

- **ESP-NOW** = agents negotiating with each other. **USB/UART** = an agent
  talking to its physics engine (laptop or RPi). They never mix.
- The real drone is a **drop-in body swap** for one agent: the ESP firmware is
  identical; only its "World" changes from the laptop to the RPi.

---

## Build order (each module testable before the next)

1. **The wire** ✅ *(this repo)* — USB contract works both ways on one board.
2. **Skeleton world** *(B)* — sim moves fake drones, finds a victim, scores vs. baseline. Ugly, end-to-end, demoable early.
3. **The mesh** — two ESP32s exchanging messages over ESP-NOW.
4. **The auction** — agents decide; rip out any laptop shortcut. Info-gain bids.
5. **Cooperative RF localization** — agents share RF samples, climb the gradient.
6. **Resilience** — heartbeat timeout → re-auction (the power-yank beat).
7. **RF sniffer** → CONFIRM_RF tasks (the hidden-phone hero beat).
8. **F450 integration** — real drone joins as a live member (hour-24 checkpoint).

Hedge: build the whole swarm in sim first (1–6). The real drone plugs in on top
of a working system, never underneath a broken one.

---

## Module 1 — how to test "the wire"

You need: one ESP32, a USB cable, [PlatformIO](https://platformio.org/) (or
Arduino IDE — see note), and Python 3.

**Flash the board:**
```bash
cd firmware
pio run -e agent -t upload
```

**Run the laptop side (in another terminal — not the pio monitor, it needs the
port to itself):**
```bash
cd sim
pip install -r requirements.txt
python wire_test.py
```

**Success:** the onboard LED blinks at ~1 Hz, and the terminal streams `GOAL ...`
lines whose coordinates track what we send, with `sent` and `goals_back` rising
together. Every ~20th line flips to `REOBSERVE` (that's the fake "detection"
round-tripping through the ESP's trivial decision). That means the byte contract
and framing are proven on real hardware, and every later module is safe to build
on top.

> **Arduino IDE instead of PlatformIO?** Copy `firmware/lib/quorum_proto/quorum_proto.h`
> next to a `.ino` containing the code from `firmware/src/agent/main.cpp`, select
> your ESP32 board, and upload. PlatformIO is recommended once we have two
> firmware targets (agent + sniffer).
