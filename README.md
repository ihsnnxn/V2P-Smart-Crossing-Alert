# V2P-Smart-Crossing-Alert

Project Idea
:
Our project explores a Vehicle-to-Pedestrian (V2P) communication system that warns drivers when pedestrians are about to cross the road.

Problem
:
Drivers may not see pedestrians clearly at night or when pedestrians are hidden behind obstacles such as parked vehicles.

Proposed Concept
:
A pedestrian's smartphone detects when the user is near a pedestrian crossing and sends a signal to nearby vehicles.

The vehicle system receives the signal and alerts the driver through a dashboard warning so the driver can slow down earlier.

Next Steps
: 
| Define system architecture
| Design message communication between pedestrian and vehicle
| Identify hardware parameters such as range and latency


## 1. System Integration

### 1.1 Solution Description & Literature Review

**Brief Description**

V2P-Smart-Crossing-Alert is a smartphone-based Vehicle-to-Pedestrian safety system that warns drivers when a pedestrian is about to cross the road — especially in low-visibility conditions such as nighttime, fog, or when the pedestrian is hidden behind parked vehicles. When a pedestrian's smartphone detects that the user is within 50 metres of a mapped pedestrian crossing, the app begins broadcasting a Bluetooth Low Energy (BLE) or C-V2X Basic Safety Message (BSM) containing the pedestrian's location, speed, and crossing intent. A compatible On-Board Unit (OBU) in the vehicle receives this signal, calculates the time-to-collision (TTC), and triggers a dashboard alert if the TTC falls below a defined safety threshold (4 seconds). The system requires no road-side infrastructure in its base configuration, making it cost-effective and deployable with existing consumer hardware.

**Why It Is Relevant**

Pedestrian fatalities remain one of the most persistent road safety challenges globally. According to the WHO Global Status Report on Road Safety (2023), pedestrians account for approximately 23% of all road traffic deaths. Singapore's Land Transport Authority similarly reports that pedestrian accidents are disproportionately severe. Current passive solutions — road markings, signal timers, speed bumps — do not communicate dynamic pedestrian presence to moving vehicles. V2P communication fills this gap by enabling real-time awareness.

**Literature Review**

Existing V2P research falls broadly into three categories. Infrastructure-based systems, such as those studied by Anaya et al. (2014) in the PUVAME project, use roadside units (RSUs) to relay pedestrian data to vehicles via DSRC/WAVE. These achieve very low latency (~20 ms) but require expensive infrastructure deployment. Smartphone-based BLE approaches, explored by researchers at MIT (Golestan et al., 2016), demonstrate that consumer smartphones can broadcast location beacons with acceptable range (~100 m) and latency (~80 ms end-to-end). Cellular V2X (C-V2X), standardised in 3GPP Release 14 (PC5 sidelink), extends range to ~300 m with latency under 100 ms and does not require network coverage for direct device-to-device mode, making it an attractive evolution path.

Our system differs from prior work in three ways: (1) it combines BLE as a primary channel with C-V2X as a fallback, adapting to hardware availability; (2) it runs entirely on commodity smartphones with no mandatory roadside infrastructure; and (3) it includes a crossing-intent flag in the BSM payload — a field not standardised in typical V2V BSMs — enabling earlier alert triggering even before the pedestrian steps off the kerb.

**References**

- Anaya, J.J. et al. (2014). *Vehicle to Pedestrian Communications for Protection of Vulnerable Road Users.* IEEE Intelligent Vehicles Symposium.
- Golestan, K. et al. (2016). *Situation Awareness within the Context of Connected Vehicles: A Comprehensive Review.* Information Fusion, 29.
- 3GPP Release 14 (2017). *LTE-based V2X Services (TS 22.185).*
- WHO (2023). *Global Status Report on Road Safety.*
- LTA Singapore (2022). *Singapore Road Accident Statistics.*


  ### 1.2 System Architecture

The system is divided into three subsystems: the **Pedestrian Subsystem** (smartphone app), the **Wireless Channel** (BLE / C-V2X / optional RSU), and the **Vehicle Subsystem** (OBU + HMI).

┌─────────────────────────────┐      Wireless     ┌──────────────────────────────┐
│     PEDESTRIAN SUBSYSTEM    │      Channel      │      VEHICLE SUBSYSTEM        │
│                             │                   │                               │
│  ┌─────────────────────┐    │  BLE (~100 m)     │  ┌──────────────────────┐    │
│  │   Smartphone App    │────┼──────────────────►│  │   OBU Receiver       │    │
│  │  GPS + BLE/C-V2X    │    │  C-V2X (~300 m)   │  │  BLE / C-V2X radio   │    │
│  └────────┬────────────┘    │                   │  └──────────┬───────────┘    │
│           │                 │  [RSU relay]       │             │                │
│  ┌────────▼────────────┐    │  optional          │  ┌──────────▼───────────┐    │
│  │  Geofence Detector  │    │                   │  │   Message Parser     │    │
│  │  <50 m from node    │    │                   │  │  Decode BSM payload  │    │
│  └────────┬────────────┘    │                   │  └──────────┬───────────┘    │
│           │                 │                   │             │                │
│  ┌────────▼────────────┐    │                   │  ┌──────────▼───────────┐    │
│  │    BSM Builder      │    │                   │  │    Risk Engine       │    │
│  │ LAT,LON,spd,intent  │    │                   │  │  Compute TTC, dist.  │    │
│  └────────┬────────────┘    │                   │  └──────────┬───────────┘    │
│           │                 │                   │             │                │
│  ┌────────▼────────────┐    │                   │  ┌──────────▼───────────┐    │
│  │   BLE/C-V2X Tx      │────┼──────────────────►│  │   HMI Alert          │    │
│  │   10 Hz broadcast   │    │                   │  │  Dashboard warning   │    │
│  └─────────────────────┘    │                   │  └──────────────────────┘    │
└─────────────────────────────┘                   └──────────────────────────────┘

**Key design decisions:**
- Pedestrian app runs in foreground or background service; triggers geofence check every 1 second via GPS.
- BSM is broadcast at **10 Hz** within the active zone and at **1 Hz** heartbeat otherwise to conserve battery.
- Vehicle OBU evaluates incoming BSMs and issues an alert only when TTC < 4 s and distance < 80 m, reducing false positives.


### 1.3 Functions and Messages

#### Message Format — Basic Safety Message (BSM) Payload
| Field | Type | Size | Description |
|---|---|---|---|
| `msg_id` | uint8 | 1 B | Always `0x02` (BSM type) |
| `timestamp` | uint32 | 4 B | Unix epoch, milliseconds |
| `latitude` | int32 | 4 B | WGS84 × 10⁷ |
| `longitude` | int32 | 4 B | WGS84 × 10⁷ |
| `speed` | uint16 | 2 B | cm/s (0–4000) |
| `heading` | uint16 | 2 B | 0.0125° units (0–28799) |
| `intent_flag` | uint8 | 1 B | `0x01` = intending to cross |
| `accuracy` | uint8 | 1 B | GPS accuracy class (0=low, 2=high) |
| `temp_id` | uint32 | 4 B | Rotating pseudonym for privacy |
| **Total** | | **23 B** | Fits comfortably in BLE advertisement payload (max 31 B) |

#### Function Descriptions

**Pedestrian App — `checkGeofence(lat, lon)`**
INPUT:  current GPS position (lat, lon)
LOAD:   crossing_nodes[] from local map DB
FOR each node in crossing_nodes:
    dist = haversine(lat, lon, node.lat, node.lon)
    IF dist < GEOFENCE_RADIUS (50 m):
        RETURN TRUE, node.id
RETURN FALSE

**Pedestrian App — `buildBSM(pos, speed, heading, intent)`**
INPUT:  position, speed, heading, crossing_intent boolean
BSM.msg_id      = 0x02
BSM.timestamp   = currentEpochMs()
BSM.latitude    = pos.lat × 10^7 (int32)
BSM.longitude   = pos.lon × 10^7 (int32)
BSM.speed       = speed in cm/s
BSM.heading     = heading in 0.0125° units
BSM.intent_flag = IF intent THEN 0x01 ELSE 0x00
BSM.accuracy    = getGPSAccuracyClass()
BSM.temp_id     = getCurrentPseudonym()
RETURN serialize(BSM)  // 23 bytes

**Vehicle OBU — `evaluateRisk(bsm, vehicle_state)`**

INPUT:  decoded BSM, own vehicle position/speed/heading
d    = haversine(vehicle.lat, vehicle.lon, bsm.lat, bsm.lon)
v_rel = vehicle.speed - bsm.speed  // relative approach speed
TTC  = d / v_rel  IF v_rel > 0 ELSE INFINITY

IF d < 80 m AND TTC < 4 s:
    triggerAlert(level = HIGH)
ELSE IF d < 120 m AND bsm.intent_flag == 0x01:
    triggerAlert(level = CAUTION)
ELSE:
    noAlert()

#### Flowchart (textual representation)

[Pedestrian App]
START
  → GPS fix acquired?
      NO  → retry in 1 s → loop
      YES → checkGeofence()
              NOT in zone  → broadcast heartbeat BSM at 1 Hz → loop
              IN ZONE      → set intent_flag=1 if user confirms crossing
                           → buildBSM()
                           → broadcast at 10 Hz via BLE/C-V2X
                           → monitor GPS until out of zone → loop

[Vehicle OBU]
START
  → listening for BSM on BLE/C-V2X channel
      BSM received → parse payload
                   → evaluateRisk(bsm, own_state)
                       TTC < 4 s → HMI HIGH ALERT (visual + audio)
                       intent + d < 120 m → HMI CAUTION
                       else → silent monitoring

### 1.4 Hardware Components and Parameters

| Component | Specification | Parameter | Value |
|---|---|---|---|
| Pedestrian smartphone | Any BLE 5.0+ / 5G handset (e.g., iPhone 14, Samsung S22) | GPS accuracy | ±3–5 m (good sky view) |
| BLE radio | BLE 5.0, advertising mode | Max TX power | +20 dBm |
| BLE radio | BLE 5.0 | Effective range | ~100 m (open road) |
| BLE radio | BLE 5.0 | Data rate | 1 Mbps (adv. channel) |
| BLE radio | BLE 5.0 | Latency (adv. → receive) | ~40–80 ms |
| C-V2X module | Qualcomm 9150 C-V2X or equivalent | Frequency | 5.9 GHz (PC5 sidelink) |
| C-V2X module | Qualcomm 9150 | Effective range | ~300 m |
| C-V2X module | Qualcomm 9150 | Latency (PC5 D2D) | ~20–50 ms |
| C-V2X module | Qualcomm 9150 | Bandwidth | 10 MHz channel |
| Vehicle OBU | Aftermarket BLE/C-V2X gateway | Processing latency | ~5–10 ms |
| Vehicle OBU | — | Update rate accepted | up to 10 Hz |
| HMI display | Vehicle head unit / aftermarket tablet | Alert display time | ≤ 500 ms post-receive |
| GPS (pedestrian) | Integrated chipset | Update rate | 1 Hz standard, 10 Hz fast mode |
| BSM payload | — | Packet size | 23 bytes |
| System end-to-end | BLE path | Total latency | ~80–100 ms |
| System end-to-end | C-V2X path | Total latency | ~30–60 ms |
| Battery impact | App background service | Estimated drain | ~3–5% per hour additional |

**Latency budget breakdown (BLE path):**

| Stage | Duration |
|---|---|
| GPS polling + geofence check | ~5 ms |
| BSM construction + serialization | ~2 ms |
| BLE advertising transmission | ~20–40 ms |
| OBU receive + parse | ~5–10 ms |
| Risk evaluation | ~3 ms |
| HMI render | ~5 ms |
| **Total** | **~40–65 ms typical, <100 ms worst case** |


### 1.5 Use Case Scenario

**Scenario: Nighttime crossing at a signalised junction in Choa Chu Kang, Singapore**

It is 11:45 PM. Sarah is walking home after her shift. She approaches the junction of Choa Chu Kang Avenue 4 and Choa Chu Kang Street 51. The road is wet, street lighting is partial, and a delivery van is parked near the kerb, blocking the view of the pedestrian crossing from northbound traffic.

As Sarah comes within 40 metres of the crossing node, her smartphone app — running silently in the background — detects the geofence trigger. She taps the crossing button on the traffic light pole. The app simultaneously sets `intent_flag = 0x01` and begins broadcasting BSMs at 10 Hz via BLE.

A northbound private-hire car, travelling at 45 km/h, is 95 metres away. Its OBU picks up Sarah's BSM within 80 ms. The risk engine computes a TTC of approximately 7.6 seconds — below the 10-second caution threshold — and `intent_flag` is set, so a **CAUTION** alert appears on the driver's head unit: *"Pedestrian ahead — crossing intended."* The driver eases off the accelerator.

By the time the signal turns green for pedestrians and Sarah steps off the kerb, the TTC has dropped to 3.8 seconds, crossing the HIGH alert threshold. The driver has already slowed to 20 km/h and stops well before the crossing. No collision occurs. The interaction lasts under 8 seconds and required no internet connectivity, no RSU, and no action beyond Sarah's normal use of the crossing button.


## 3. AI Usage and Individual Reflection

### 3.1 AI Tools Used

| Tool | Phases Used | Purpose |
|---|---|---|
| Claude (Anthropic) | All phases | Architecture reasoning, BSM payload design, decision log drafting, simulation pseudocode, literature search |
| ChatGPT 4o | Research phase | Cross-reference on 3GPP V2X standards and BLE power consumption |
| GitHub Copilot | Simulation coding | Autocomplete for Python haversine function and BSM serialisation |


### 3.2 Example Prompts and Generated Outputs

**Prompt 1 (Architecture design):**
> *"We are building a V2P system where a pedestrian's smartphone alerts nearby vehicles via BLE. The vehicle has a BLE receiver. Design the system architecture including all subsystems, message format, and key parameters such as range and latency."*

**Output summary:** Claude produced a structured breakdown of three subsystems (pedestrian app, wireless channel, vehicle OBU), a BSM field table, and latency estimates (~80 ms end-to-end for BLE). The architecture closely matched what we adopted, though we added the intent_flag field and two-tier alert logic ourselves after reviewing driver reaction time literature.


*Prompt 2 (BSM payload):**
> *"Design a compact BSM packet for BLE advertisement that fits in 31 bytes and carries GPS position, speed, heading, and a crossing intent flag. Show field names, data types, and byte sizes."*
**Output summary:** Claude proposed a 26-byte packet. We reduced it to 23 bytes by removing a redundant checksum field (BLE handles integrity at link layer) and compressing the accuracy 
field from uint16 to uint8. This demonstrates the need for human engineering review of AI-generated specifications.


**Prompt 3 (Risk engine pseudocode):**
> *"Write pseudocode for a vehicle on-board unit that receives BSMs from pedestrians, computes time-to-collision, and triggers a two-tier driver alert (CAUTION and HIGH). Use haversine for distance."*

**Output summary:** Claude generated correct pseudocode structure. However, it initialised TTC as `d / v_rel` without guarding against `v_rel ≤ 0` (vehicles moving away from pedestrian). We added the `IF v_rel > 0 ELSE INFINITY` guard to prevent division by zero, which would have been a critical runtime bug.


### 3.3 Identified AI Weaknesses / Hallucinations

**Weakness 1 — Incorrect latency for BLE advertising:**
Claude initially stated BLE advertising latency as "approximately 10–20 ms". After checking Nordic Semiconductor's power profiler documentation and published measurements, actual typical latency is 40–80 ms at standard advertising intervals. We updated all tables accordingly. The AI underestimated latency by approximately 2–4×, which would have affected our TTC threshold calculations.

**Weakness 2 — Incorrect Singapore spectrum allocation:**
Claude stated that "Singapore has not yet allocated 5.9 GHz for V2X". A check of the IMDA website confirmed that Singapore has indeed designated the 5.9 GHz band for Intelligent Transport Systems (ITS), consistent with international harmonisation. This hallucination would have led us to incorrectly exclude C-V2X from our architecture.

**Weakness 3 — Pseudonym change interval:**
Claude suggested a pseudonym rotation interval of "every 60 seconds" for privacy. The SAE J2945/1 standard specifies a change interval of "not more than 5 minutes with event-driven changes". We aligned our implementation to the standard rather than the AI's suggestion, which would have caused unnecessary frequent re-identification disruptions in practice.


### 3.4 Individual Reflections
**[Member 1 — Joseph]**
My primary contribution was defining the overall system architecture. I found AI tools most useful for quickly generating a first draft of the BSM field table and the risk engine pseudocode, which gave us a concrete starting point to critique and improve. The most important lesson was that AI tends to present outputs with false confidence — the BLE latency and spectrum allocation errors could have gone unnoticed without independent verification. In future projects, I would treat AI outputs as a first draft requiring engineering validation rather than a final answer.

**[Member 2 — ]**

**[Member 3 —]**

**[Member 4 — ]**

**[Member 5 — ]**




