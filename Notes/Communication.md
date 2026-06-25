# Data Communication Reference: Media, Connectors, and Protocols

## Mental Model

The physical layer is really **three independent things** that get combined:

- **Medium** — what physically carries the signal (copper, fiber, air)
- **Connector** — the mechanical interface where the medium terminates
- **Signaling / Protocol** — the electrical rules and the language spoken over them

A given protocol picks a signaling standard, which runs over a medium, which terminates in a connector — but the combinations are **not fixed**. This is the main source of confusion (e.g. RS-485 mandates no specific connector). Keep these three axes separate in your head and most of the "what can have what" questions answer themselves.

---

## 1. Transmission Media

| Medium | Common variants | Signaling | Typical reach | Notes |
|---|---|---|---|---|
| Twisted pair (copper) | Cat5e, Cat6, Cat6a, Cat7, Cat8; UTP vs STP/FTP (shielded) | Differential | 100 m (Ethernet) | The workhorse. Higher cat = more bandwidth / shielding |
| Multi-conductor copper | Shielded pair/triad, instrument cable | Single-ended or differential | varies | RS-232, RS-485, 4–20 mA, HART |
| Coaxial | RG-6, RG-58, RG-59, RG-11 | Single-ended | 10s–100s m | TV/sat (RG-6), legacy 10BASE2 thinnet (RG-58) |
| Fiber – multimode (MMF) | OM1, OM2, OM3, OM4, OM5 | Optical | ~300–550 m (OM3/4 @ 10G) | Short-reach, cheaper optics, larger core |
| Fiber – single-mode (SMF) | OS1, OS2 | Optical | 10–80+ km | Long-haul, immune to EMI |
| Wireless / RF | Wi-Fi, BLE, Zigbee, LoRa, cellular | RF modulation | m to km | No connector; the "medium" is air |

---

## 2. Connectors (and what they carry)

| Connector | Medium | Typical protocols / use |
|---|---|---|
| RJ45 (8P8C) | Twisted pair | Ethernet (10BASE-T → 10GBASE-T), PROFINET, EtherNet/IP, EtherCAT, Modbus TCP; also serial console cables |
| RJ11 / RJ12 | Twisted pair | Telephone, some legacy serial |
| DB9 / DE-9 | Multi-conductor | RS-232, RS-485, CAN, PROFIBUS (9-pin D-sub) |
| DB25 | Multi-conductor | Legacy RS-232, parallel printer |
| M12 (A/B/D/X coded) | Twisted pair / multi-conductor | Industrial: D-code = 100 Mb Ethernet, X-code = 10 Gb Ethernet, B-code = PROFIBUS, A-code = sensors/DC power |
| M8 | Multi-conductor | Compact sensor / actuator connections |
| Screw / pluggable terminal (Phoenix) | Bare copper | RS-485, Modbus RTU, fieldbus, 4–20 mA, power |
| USB-A/B/C, Mini, Micro | Twisted pair + power | USB peripherals, device programming |
| BNC | Coax | Legacy 10BASE2, video, instrumentation |
| F-connector | Coax (RG-6) | Cable / satellite, DOCSIS modems |
| LC | Fiber | Modern duplex fiber (most data center / SFP) |
| SC | Fiber | Push-pull, common in telecom / older installs |
| ST | Fiber | Bayonet, legacy multimode |
| FC | Fiber | Screw-type, test equipment / older |
| MPO / MTP | Fiber (ribbon) | High-density 40/100G breakouts |
| XLR | Multi-conductor | Audio, DMX512 lighting |

---

## 3. Serial Electrical Standards

| Standard | Signaling | Topology | Medium | Common connector | Distance | Speed (typical) |
|---|---|---|---|---|---|---|
| RS-232 | Single-ended, ±3–15 V | Point-to-point | Multi-conductor | DB9 / DB25 | ~15 m | up to ~115 kbps |
| RS-422 | Differential | 1 driver, up to 10 rx (multidrop) | Twisted pair | DB9 / terminals | ~1200 m | up to 10 Mbps |
| RS-485 | Differential | Multidrop, bi-directional (half/full duplex) | Twisted pair (1–2 pair) | Terminals / DB9 / M12 | ~1200 m | up to 10 Mbps; 32 std loads (256 w/ modern transceivers) |
| CAN | Differential (CAN_H / CAN_L) | Multi-master bus | Twisted pair | DB9 | ~40 m @ 1 Mbps | 1 Mbps (CAN 2.0), faster w/ CAN FD; 120 Ω term both ends |

> **Note:** Distance and speed trade off inversely on all of these — the figures above are the friendly ends of the curve, not simultaneous maximums.

---

## 4. Ethernet Physical Layers

| Variant | Speed | Medium | Max length | Connector |
|---|---|---|---|---|
| 10BASE-T | 10 Mbps | Cat3+ | 100 m | RJ45 |
| 100BASE-TX | 100 Mbps | Cat5+ | 100 m | RJ45 |
| 1000BASE-T | 1 Gbps | Cat5e+ | 100 m | RJ45 |
| 2.5G / 5GBASE-T | 2.5 / 5 Gbps | Cat5e / Cat6 | 100 m | RJ45 |
| 10GBASE-T | 10 Gbps | Cat6a / Cat7 | 100 m (Cat6 ~55 m) | RJ45 |
| 25 / 40GBASE-T | 25 / 40 Gbps | Cat8 | 30 m | RJ45 |
| 100BASE-FX | 100 Mbps | MMF | 2 km | SC / LC / ST |
| 1000BASE-SX | 1 Gbps | MMF | ~550 m | LC / SC |
| 1000BASE-LX | 1 Gbps | SMF | ~5–10 km | LC / SC |
| 10GBASE-SR / LR | 10 Gbps | MMF / SMF | 300 m / 10 km | LC |

---

## 5. Protocols & Fieldbuses

| Protocol | Physical layer it rides on | Medium | Connector |
|---|---|---|---|
| Modbus RTU | RS-485 (or RS-232) | Twisted pair | Terminals / DB9 |
| Modbus TCP | Ethernet | Twisted pair / fiber | RJ45 |
| PROFIBUS DP | RS-485 | Twisted pair | DB9 / M12 (B-code) |
| PROFINET | Ethernet | Twisted pair / fiber | RJ45 / M12 (D/X) |
| EtherCAT | Ethernet | Twisted pair | RJ45 / M8 / M12 |
| EtherNet/IP | Ethernet | Twisted pair | RJ45 / M12 |
| DeviceNet | CAN | Twisted pair + power | Open / sealed M12, terminals |
| CANopen | CAN | Twisted pair | DB9 / M12 |
| Foundation Fieldbus H1 | Manchester-coded current | Twisted pair | Terminals |
| HART | Analog 4–20 mA + FSK overlay | Twisted pair | Terminals |
| BACnet MS/TP | RS-485 | Twisted pair | Terminals |
| BACnet/IP | Ethernet | Twisted pair / fiber | RJ45 |
| AS-i | Unshielded 2-wire (data + power) | Flat cable | Pierce / terminals |
| DMX512 | RS-485 | Twisted pair | XLR (5-pin) |
| KNX / DALI | Dedicated 2-wire bus | Twisted pair | Terminals |

---

## Cross-Cutting Points

- **Differential vs single-ended:** Differential signaling (RS-422/485, CAN, Ethernet pairs) is what buys you noise immunity and distance over single-ended RS-232. Each pair carries the signal and its inverse; the receiver looks at the *difference*, so common-mode noise cancels out.
- **Protocol families:** "Modbus" and "BACnet" are protocol *families* that run over either serial or Ethernet — always specify the variant (RTU vs TCP, MS/TP vs IP).
- **Fiber immunity:** Fiber is the only medium fully immune to EMI and ground-loop issues, which matters on plant floors and between buildings with separate ground references.
- **Connectors are not protocols:** A DB9 might be RS-232, RS-485, CAN, or PROFIBUS. An RJ45 might be Ethernet or a serial console. The connector tells you the mechanics, not the language.
- **Termination:** Bus standards (RS-485, CAN, PROFIBUS) need termination resistors (typically 120 Ω) at both physical ends to prevent reflections. Missing or doubled termination is a classic field fault.