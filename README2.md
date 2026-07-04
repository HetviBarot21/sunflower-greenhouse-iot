# ICS 4111: Embedded Systems & IoT
## Semester Project – Deliverable 2: Prototyping

## 1. Objective

This deliverable builds on the schematics produced in Deliverable 1 and turns them into working prototypes, both physical and simulated, based on the three device architectures assigned for the sunflower greenhouse monitoring system:

- **Architecture A** – 1 ESP32S connected to 1 MQ-5 gas sensor, 1 DHT22 temperature/humidity sensor and 1 OLED display. Built as both a physical and a simulated model.
- **Architecture B** – 1 ESP32S connected to an MQ-5 sensor, interfaced directly with a second ESP32S connected to a DHT22.
- **Architecture C** – 1 ESP32S connected to a DHT22, feeding a relay which connects to a second ESP32S connected to an MQ-5.

Since B and C are interchangeable, our team built a physical prototype for one and a Wokwi simulation for the other, so that between the two we cover both wiring approaches.

---

## 2. Prototype Summary

| # | Architecture | Type | Status | Link / Evidence |
|---|--------------|------|--------|------------------|
| 1 | A – ESP32 + MQ-5 + DHT22 + OLED | Physical | Working | See section 3 |
| 2 | A – ESP32 + MQ-5 + DHT22 + OLED | Simulated (Wokwi) | Working | https://wokwi.com/projects/467006558712168449 |
| 3 | C – ESP32 (DHT22) → Relay → ESP32 (MQ-5) | Physical | See section 5 | See section 5 |
| 4 | B – ESP32 (MQ-5) ↔ ESP32 (DHT22) | Simulated (Wokwi) | Working | https://wokwi.com/projects/467017703807057921 |

---

## 3. Prototype 1: Physical – Architecture A (ESP32 + MQ-5 + DHT22 + OLED)

### 3.1 Components used

- 1x ESP32S development board
- 1x MQ-5 gas sensor module
- 1x DHT22 temperature and humidity sensor
- 1x 0.96" I2C OLED display
- Breadboard, jumper wires
- Current-limiting resistors on the sensor lines to protect the ESP32 GPIO pins

### 3.2 Wiring notes

The OLED is wired over I2C (VCC, GND, SCL, SDA), the MQ-5 analog output goes into an ESP32 ADC pin, and the DHT22 data line is pulled up and connected to a digital GPIO. Resistors were placed in line with the sensor signal wires as required by the deliverable instructions, to avoid overdriving the ESP32 pins.

### 3.3 Output

The OLED displays live readings in the format:

```
Temp: 26.3C
Hum:  66.4%
Gas:  15
```

Readings were captured across multiple trials to confirm the sensors respond to environmental changes (breathing near the MQ-5 raises the gas reading, and humidity shifts as expected):

| Trial | Temp (°C) | Humidity (%) | Gas reading |
|-------|-----------|---------------|--------------|
| 1     | 26.3      | 66.4          | 15           |
| 2     | 26.3      | 67.2          | 0            |
| 3     | 26.3      | 62.8          | 73           |
| 4     | 26.3      | 62.9          | 64           |
| 5     | 26.3      | 63.7          | 60           |
| 6     | 26.3      | 63.1          | 61           |
| 7     | 26.3      | 68.0          | 15           |

The gas value climbs noticeably in trials 3 and 4, which lines up with the sensor being triggered by hand near the sensing element during testing.

### 3.4 Evidence
Full breadboard view, OLED reading Temp 26.3C, Hum 66.4%, Gas 15

<img src="full_breadboard_view.jpeg" width="400"/>

Close-up of the OLED display showing live readings

<img src="closeup.jpeg" width="400"/>

MQ-5 sensor being triggered by hand, OLED reading Gas 73

<img src="image3.jpeg" width="400"/>

MQ-5 sensor triggered a second time, OLED reading Gas 64

<img src="image4.jpeg" width="400"/>

Wide view of the full architecture, a breadboard build

<img src="image5.jpeg" width="400"/>

Final reading on the OLED, Hum 68.0%, Gas 15

<img src="image6.jpeg" width="400"/>

---

## 4. Prototype 2: Simulated – Architecture A (Wokwi)

A matching simulation of the same architecture (ESP32 + MQ-5 + DHT22 + OLED) was built on Wokwi to validate the logic independently of physical component tolerances.

- Wokwi project link: https://wokwi.com/projects/467006558712168449
- DHT22 is wired to GPIO 4, the gas sensor's analog output goes into GPIO 34, and the SSD1306 OLED runs over I2C at address 0x3C.
- The sketch shows a startup splash screen ("Flower Monitor v1.0"), then loops every 2 seconds reading temperature, humidity and gas level, printing them to the serial monitor, and redrawing them on the OLED under an "ENV METRICS" header.
- If the DHT22 read fails, the sketch prints a "DHT22 Error!" message to the OLED instead of stale data, which was used to confirm the error-handling path works as well as the normal reading path.
- Serial monitor output and the simulated OLED were used to confirm the code logic matches the physical prototype's behaviour.

---

## 5. Prototype 3: Physical – Architecture C (DHT22 → Relay → MQ-5, dual ESP32)

### 5.1 Components used

- 2x ESP32S / Arduino Nano-style boards (one per node)
- 1x DHT22 sensor
- 1x MQ-5 gas sensor module
- 1x relay module
- Breadboard, jumper wires, resistor for the sensor signal line

### 5.2 Build notes

Node 2 (climate node) reads the DHT22 on GPIO 4 and controls the relay on GPIO 12. When temperature exceeds 28°C, it closes the relay; otherwise the relay stays open. Node 1 (gas node) reads the MQ-5 on GPIO 36 and listens to the relay contacts on GPIO 14, configured as `INPUT_PULLUP` so it reads HIGH by default and drops to LOW the moment node 2 closes the relay. This is the interlock behaviour required by architecture C: node 1's serial output reports the local gas level plus whether it is seeing a "TRIGGERED" or "NORMAL" state from node 2's relay.

Since both boards run the same sketch, each ESP32 determines its own role at boot by checking its virtual MAC address (even vs odd), which is how a single codebase can serve either node depending on which board it is flashed to.

### 5.3 Status and issues encountered

During physical assembly we ran into the following:

- **Loose header connections between the two boards.** The jumper wires connecting the DHT22 node to the relay and the relay to the MQ-5 node kept disconnecting during handling, which produced intermittent readings on the first few test runs.
  - *Solution explored:* reseated all jumper wires directly into the breadboard rows instead of stacking them on the module headers, and shortened the wire runs between the two ESP32 boards to reduce strain on the connectors.
- **Relay module drawing more current than expected**, which caused voltage dips visible in unstable sensor readings on the MQ-5 side.
  - *Solution explored:* powered the relay from a separate section of the breadboard's power rail rather than sharing the rail directly with the sensors, and added a resistor to the sensor signal line as instructed.
- **Recommendation going forward:** if the connection instability persists past this deliverable, we recommend soldering a small interconnect harness between the two boards rather than relying on loose jumper wires, since the two-ESP32 setups are more physically fragile than the single-board architecture A build.

### 5.4 Evidence

[HERE GOES IMAGE: second breadboard with the two ESP32 boards, DHT22, MQ-5 and relay module wired together]

[HERE GOES IMAGE: close-up of the dual-board wiring with the relay module visible]

[HERE GOES IMAGE: team member connecting the interconnect wires between the two boards]

[HERE GOES IMAGE: team member testing the dual-board setup with laptop nearby for serial monitor output]

---

## 6. Prototype 4: Simulated – Architecture B (Wokwi)

Since architecture C was built physically, architecture B (ESP32 with MQ-5 interfaced directly with a second ESP32 running the DHT22, without the relay stage) was built as the corresponding Wokwi simulation.

- Wokwi project link: https://wokwi.com/projects/467017703807057921
- The two boards communicate over hardware UART2 (RX/TX on GPIO 16/17) rather than a relay. Node 1 reads the gas sensor on GPIO 34 and transmits the raw value as a line of text over UART. Node 2 reads its local DHT22 on GPIO 4 and listens for incoming gas readings over the same UART link.
- Each board figures out its own role at runtime: it first checks whether a DHT22 responds on GPIO 4. If it gets a valid temperature reading, it settles into the "climate receiver" role; if not, it falls back to the "gas transmitter" role. This means the same sketch can be flashed to both boards.
- Node 2's serial monitor output prints both its own local temperature and humidity and the most recent gas value received from node 1, confirming the direct ESP32-to-ESP32 link works without needing a relay in between.

---

## 7. Evidence of groupwork

- Physical assembly and testing were carried out together on the same breadboard setup, with team members alternating between wiring, powering the circuit, and reading the OLED output.
- Photos in sections 3 and 5 show team members actively working on the physical builds, including the architecture C assembly photos where a team member is seen connecting the interconnect wires between the two boards while another monitors output on a laptop.
- Wokwi projects were shared as public links so all members could view, edit, and test the simulated circuits.

---

## 8. Repository structure

```
sunflower-greenhouse-iot/
├── README.md              # Deliverable 1 overview
├── README1.md              # Deliverable 1 detail
├── README2.md              # This document (Deliverable 2)
├── images/                 # Photos of physical prototypes
├── wokwi/                  # Wokwi project exports/screenshots (optional)
```

---

## 9. Conclusion

Between the physical and simulated builds, the team has working prototypes for all three architectures from Deliverable 1. Architecture A was validated on both physical hardware and Wokwi, giving confidence that the OLED, MQ-5 and DHT22 readings match across both environments. Architectures B and C were split between a physical build and a simulation as permitted, and the physical build for architecture C surfaced real wiring issues that a simulation alone would not have caught, which is documented in section 5.3 along with the fixes applied and a recommendation for a more permanent connection method going forward.
