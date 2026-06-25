# HVAC Systems — A Comprehensive Engineering Reference

*Written for a controls/automation engineer: equipment first principles, then the sequences that drive them, then the communication layer that ties them into a BMS, DCS, or SCADA.*

---

## 1. The Big Picture — What HVAC Actually Does

Every HVAC system exists to move **heat** and **moisture** to where you want them and away from where you don't, while keeping air clean and at the right pressure. Strip away the equipment names and there are only a handful of physical jobs:

- **Sensible cooling/heating** — changing dry-bulb temperature.
- **Latent cooling/heating** — adding or removing moisture (humidity).
- **Air movement & distribution** — fans pushing/pulling a controlled CFM.
- **Air quality** — filtration, fresh-air dilution (CO₂ control), exhaust of contaminants.
- **Pressurization** — keeping spaces positive or negative relative to neighbors (cleanrooms, labs, kitchens, data centers).

Everything below is just a different machine doing one or more of those five jobs. The two great "fluids" that carry energy around a building are **air** (ductwork) and **water/refrigerant** (piping). A useful mental model:

```
        ENERGY SOURCE              DISTRIBUTION              DELIVERY
   ┌─────────────────────┐   ┌────────────────────┐   ┌──────────────────┐
   │ Chiller / Boiler    │──▶│ Chilled / Hot Water│──▶│ AHU / FCU coils  │
   │ Cooling Tower       │   │ Condenser Water    │   │ VAV boxes        │
   │ DX Refrigeration    │   │ Refrigerant (DX)   │   │ Diffusers/grilles│
   └─────────────────────┘   └────────────────────┘   └──────────────────┘
        "Central Plant"          "Hydronic/Refrig         "Air-side"
                                   Distribution"
```

---

## 2. The Refrigeration Cycle (The Heart of All Cooling)

Almost every cooling machine — chiller, CRAC, rooftop unit, heat pump, your home AC — runs the **vapor-compression cycle**. Four components, one loop of refrigerant:

1. **Evaporator** — low-pressure liquid refrigerant absorbs heat from the load and boils into vapor. *This is where "cold" is made.*
2. **Compressor** — raises refrigerant pressure and temperature (the energy-input step).
3. **Condenser** — high-pressure vapor rejects heat to outdoor air or condenser water and condenses to liquid. *This is where heat is dumped.*
4. **Expansion device** (TXV, EEV, or orifice) — drops pressure, setting up the next evaporation.

```
   Evaporator ──vapor──▶ Compressor ──hot vapor──▶ Condenser
       ▲                                                │
       │                                              liquid
   cold liquid ◀──── Expansion Valve ◀─────────────────┘
```

A **heat pump** is this exact cycle with a *reversing valve* that swaps the evaporator and condenser roles, so the same box can heat or cool.

**Absorption chillers** are the odd one out: they replace the compressor with a thermal/chemical process (lithium bromide + water, driven by steam, hot water, or gas). Useful where waste heat is cheap and electricity is expensive.

---

## 3. Central Plant Equipment

### 3.1 Chillers

A chiller makes **chilled water** (typically supplied ~42–45 °F / 6–7 °C, returning ~54–57 °F) that gets pumped to air-side coils. Classified two ways:

**By how they reject heat:**
- **Air-cooled** — condenser fans blow ambient air over the condenser coil. Simpler, no cooling tower, lower efficiency, usually outdoors/rooftop.
- **Water-cooled** — rejects heat to a **condenser water** loop served by a cooling tower. More efficient, used in larger plants, requires tower + condenser pumps + water treatment.

**By compressor type (smallest → largest capacity):**
- **Scroll** — orbiting scrolls; small/medium tonnage, common in packaged units.
- **Reciprocating** — pistons; older tech, mostly displaced by scroll/screw.
- **Screw (helical rotary)** — two meshing rotors; mid-to-large tonnage, good part-load.
- **Centrifugal** — impeller spins refrigerant vapor; large tonnage (hundreds to thousands of tons), best full-load efficiency. Often **magnetic-bearing/oil-free** in modern units (Daikin Magnitude, etc.).

**Key chiller control points you'll see on a BMS/DCS:**
- Leaving Chilled Water Temp (LCHWT) setpoint — the primary control variable.
- Chilled water supply/return temps, condenser water supply/return temps.
- Evaporator/condenser refrigerant pressures, compressor amps/speed, % RLA.
- Capacity command (often 0–100 %, via vanes, slide valve, or VFD speed).
- Run/stop, alarm/fault, ready status, run hours.
- **Chilled water reset** — raising LCHWT setpoint when loads are light saves big energy.

> *Your 800xA experience:* Trane (and York/Carrier/Daikin) chillers usually expose all of this over **BACnet/IP, Modbus TCP, or a proprietary gateway** (Trane Tracer, Carrier i-Vu). On the CI867A you'd typically pull them in as **Modbus TCP** registers.

### 3.2 Cooling Towers

Reject condenser-water heat to atmosphere by **evaporative cooling** — water cascades over fill media while a fan pulls air through; a fraction evaporates, cooling the rest. Output is **condenser water supply** (~85 °F design) going back to the chiller condenser.

- **Types:** induced-draft vs forced-draft; counterflow vs crossflow; open (water touches air) vs closed-circuit (coil keeps process water isolated).
- **Control:** fan VFD modulates to hold condenser water supply temp; basin level/makeup valve; bleed/blowdown for water treatment; vibration and basin-heater (freeze protection) interlocks.
- **Approach** = tower leaving water temp − ambient wet-bulb. The tighter the approach, the better the tower performance.

### 3.3 Boilers

Make **hot water** (typically 140–180 °F) or **steam** for heating coils, reheat, and DHW.

- **Types:** firetube vs watertube; condensing (high efficiency, low return-water temp) vs non-condensing; electric, gas, oil.
- **Control:** firing rate / modulating burner to hold supply water temp; **hot water reset** (lower supply temp in mild weather); flame safety (a separate safety system — BMS *monitors*, the burner management system *controls* the burner per NFPA 85/86), low-water cutoff, stack temp, O₂ trim.

### 3.4 Pumps

Move water around the loops. You'll meet:
- **Chilled water pumps (CHWP)** — primary, secondary, or primary/variable.
- **Condenser water pumps (CWP)** — chiller condenser ↔ cooling tower.
- **Hot water pumps (HWP)** — boiler ↔ heating coils.

**Loop topologies:**
- **Constant primary** — fixed flow, simple, energy-hungry.
- **Primary/secondary (decoupled)** — constant flow through chillers, variable flow to the building. The classic large-plant arrangement.
- **Variable primary flow (VPF)** — single set of VFD pumps, minimum-flow bypass valve protects the chiller. Modern, efficient, more control-intensive.

Control: VFD speed to hold a **differential pressure (DP) setpoint** across the loop, with **DP reset** based on the most-open valve.

---

## 4. Air-Side Equipment

### 4.1 Air Handling Units (AHUs)

The workhorse box that conditions and moves air. Walk through it in airflow order — these are the components you asked about, in context:

```
 OUTSIDE AIR ─▶│         │ MIXED  │      │      │      │       │
               │ OA      │ AIR    │FILTER│ HTG  │ CLG  │ SUPPLY│─▶ SUPPLY AIR (SA)
 RETURN AIR ──▶│ DAMPER  │ PLENUM │ BANK │ COIL │ COIL │  FAN  │   to spaces
      ▲        │ RA/EA   │        │      │      │      │       │
      │        │ DAMPERS │                                      
   from spaces │         │──▶ EXHAUST/RELIEF AIR (EA) out
```

**Component by component:**

- **Outside Air (OA) damper** — admits fresh air for ventilation (ASHRAE 62.1 minimums, CO₂-based demand-control ventilation).
- **Return Air (RA) damper** — recirculated air from the space.
- **Exhaust/Relief (EA) damper** — dumps excess air to maintain building pressure.
- **Mixing plenum / economizer** — OA, RA, EA dampers modulate together. The **economizer** uses cool outside air for "free cooling" when conditions allow (dry-bulb or enthalpy high-limit lockout).
- **Filters** — pre-filters + final filters (MERV-rated, or HEPA for cleanrooms). Monitored by **differential pressure** across the bank (dirty-filter alarm).
- **Heating coil** — hot water, steam, or electric. Raises SA temp; also used for freeze protection (preheat).
- **Cooling coil** — chilled water or DX. Provides sensible + latent cooling; the condensate it produces dehumidifies the air. Has a condensate drain pan.
- **Humidifier** (if equipped) — steam or evaporative, adds moisture in winter.
- **Supply fan** — pushes conditioned air into the supply duct.
- **Return fan** (in larger units) — pulls air back from spaces, enables proper economizer operation and pressure control.

**The air streams, named:**
| Term | Abbrev | What it is |
|---|---|---|
| Outside Air | OA / OSA | Fresh air drawn in |
| Return Air | RA | Air coming back from the space |
| Mixed Air | MA | OA + RA blend before the coils |
| Supply Air | SA | Conditioned air sent to the space |
| Exhaust / Relief Air | EA / RLF | Air discharged from the building |
| Discharge Air | DA | Air leaving the AHU (≈ SA) |

**AHU control essentials:**
- **Supply Air Temp (SAT/DAT) control** — sequence heating valve → economizer dampers → cooling valve in a non-overlapping band (the classic "cooling-economizer-heating" sequence).
- **Supply Air Static Pressure** — supply fan VFD modulates to hold duct static; **static pressure reset** trims it down based on VAV box damper positions.
- **Building/space pressure** — return/relief control.
- **Mixed air low-limit / freeze stat** — hardwired safety that trips the fan and opens the heating valve if MA gets near freezing.

### 4.2 Rooftop Units (RTUs / Packaged Units)

An AHU + DX cooling (and often gas heat) all in one self-contained box on the roof. Common in retail/commercial. Same air streams, but cooling is **direct-expansion** (refrigerant coil) instead of chilled water. Stages of DX compression + economizer + gas burner staging.

### 4.3 Variable Air Volume (VAV) vs Constant Air Volume (CAV)

- **CAV** — fixed airflow, vary the supply temperature. Simple, less efficient.
- **VAV** — fixed (cool) supply temp, vary the *volume* per zone via **VAV terminal boxes**. The dominant commercial design.

**VAV box** = a damper + airflow sensor serving one zone, often with a **reheat coil** (hot water or electric) for when the zone needs heat. Types:
- **Single-duct** (cooling + optional reheat).
- **Fan-powered** (series or parallel fan) — boosts air and recovers plenum heat.
- **Dual-duct** — separate hot and cold ducts mixed at the box.

### 4.4 Fan Coil Units (FCUs)

Small, local units (hotel rooms, offices): a fan + coil(s) + filter, fed by central chilled/hot water (2-pipe or 4-pipe). No ductwork to speak of. Cheap, decentralized.

### 4.5 Dedicated Outdoor Air Systems (DOAS)

A unit that conditions **100 % outside air** (handling all the ventilation + latent load) and delivers it separately, while FCUs/VAVs/chilled beams handle the sensible load. Decouples ventilation from cooling — increasingly standard in efficient designs. Usually paired with **energy recovery**.

### 4.6 Energy/Heat Recovery (ERV / HRV)

Captures energy from exhaust air to pre-condition incoming OA:
- **HRV** — sensible only (heat).
- **ERV** — sensible + latent (heat + moisture), via enthalpy wheel, plate exchanger, or heat-pipe.

### 4.7 Make-Up Air Units (MAU)

Supply tempered outside air to replace air removed by large exhaust systems (commercial kitchens, industrial process exhaust). Keeps the building from going excessively negative.

---

## 5. Fans (Supply, Return, Exhaust)

Fans are categorized by airflow path and impeller:

**By function:**
- **Supply fan** — delivers conditioned air to spaces.
- **Return fan** — draws air back from spaces to the AHU.
- **Exhaust fan** — removes air (and contaminants) from a space directly to outside (restrooms, kitchens, fume hoods, parking garages, labs).
- **Relief fan** — dumps excess building air during economizer operation.

**By type:**
- **Centrifugal** (squirrel cage) — air enters axially, exits radially. High pressure capability. Sub-types by blade: **forward-curved** (quiet, low pressure), **backward-inclined/airfoil** (efficient, high static), **radial** (industrial, material handling).
- **Axial** — air flows straight through (propeller, tube-axial, vane-axial). High flow, low pressure.
- **Plenum/plug fan** — unhoused backward-curved wheel in a plenum; common in modern AHUs (fan-wall arrays of multiple small plug fans = "fan array").

**Control:** Almost always a **VFD** modulating speed to hold a pressure or flow setpoint. Key points: speed command (%/Hz), speed feedback, run/fault, current, drive temperature, bypass status. Status proof via **current sensor** or **differential pressure switch** across the fan.

---

## 6. Data Center & Precision Cooling (CRAC / CRAH)

This is your Vertiv Liebert world, so it gets its own section.

- **CRAC (Computer Room Air Conditioner)** — a precision unit with **its own DX refrigeration** (compressor + condenser, air- or water-cooled). Self-contained cooling.
- **CRAH (Computer Room Air Handler)** — a precision unit with a **chilled water coil** instead of DX; relies on a central chiller plant. More efficient at scale, common in large data centers.

Both are **precision** units: tight temperature *and* humidity control, high sensible heat ratio (data centers are nearly all sensible load — servers don't sweat), high airflow, designed for 24/7/365 duty with redundancy (N+1).

**Vertiv Liebert specifics you've touched:**
- **Liebert CW** = chilled-water (CRAH-type). **Liebert PDX/DS** = DX (CRAC-type, often with pumped refrigerant or Lee-Temp condensers).
- **iCOM** controller — the brains; does teamwork/lead-lag across multiple units, and exposes data.
- **IntelliSlot** card cage — the comms gateway. **IS-UNITY-DP** is the Unity card that speaks **BACnet/IP, BACnet MS/TP, Modbus RTU, Modbus TCP, and SNMP** simultaneously, plus a web UI. This is exactly how you'd integrate it into 800xA over Modbus TCP, or into a BMS over BACnet.

**Airflow management** (the real game in data centers):
- Hot-aisle / cold-aisle containment.
- Raised-floor (downflow units pressurize a plenum) vs in-row vs overhead cooling.
- Free cooling / **economization** (airside or waterside) to skip the compressor when ambient is cold.
- **Supply air temp / under-floor pressure** control, humidity within a tight band (ASHRAE TC9.9 envelope).

---

## 7. Specialized / Process Equipment You May Meet

- **Precision process chillers** — tight ±0.1 °C loops for lab/industrial equipment.
- **Glycol systems** — freeze-protected loops for outdoor/low-temp service.
- **Radiant systems** — heating/cooling via floor/ceiling water loops.
- **Chilled beams** — passive/active convective units fed by chilled water; quiet, low-energy.
- **Unit heaters / cabinet heaters** — localized space heating.
- **Air curtains** — fan barrier over doorways.
- **VRF/VRV (Variable Refrigerant Flow)** — many indoor DX evaporators on one outdoor unit, refrigerant modulated per zone; heat-recovery versions move heat between zones. Daikin/Mitsubishi/LG dominate; they speak their own protocols with BACnet/Modbus gateways.

---

## 8. Sensors, Valves, Actuators, Dampers (The Field Layer)

This is the I/O that lands on your controller. You'll recognize most of it.

**Sensors:**
- **Temperature** — RTDs (Pt100/Pt1000), thermistors (10kΩ NTC, the BAS standard), thermocouples for high temp. Duct, immersion (well), averaging, and outdoor types.
- **Humidity** — capacitive RH sensors, often combined T/RH; sometimes dewpoint.
- **Pressure** — duct static (low ΔP transmitters, inches WC), building pressure, water DP (across coils/pumps), refrigerant pressure transducers.
- **Flow** — air (pitot/thermal-dispersion airflow stations), water (magnetic, vortex, ultrasonic, turbine).
- **CO₂ / IAQ** — NDIR CO₂ for demand-control ventilation; VOC, PM sensors.
- **Status/proof** — current switches, DP switches, end-switches.

**Final control elements:**
- **Control valves** — 2-way (vary flow) or 3-way (divert/mix); globe valves with linear/equal-percentage characteristics; **PICV** (pressure-independent control valves) increasingly standard.
- **Valve & damper actuators** — modulating (0–10 V / 4–20 mA / floating) or 2-position; electric or pneumatic (legacy). Spring-return for fail-safe positions.
- **Dampers** — OA/RA/EA control dampers, fire/smoke dampers (life-safety, separate logic), VAV box dampers.

**Signal types (your home turf):** 4–20 mA, 0–10 V, 0–135 Ω (legacy floating), thermistor/RTD resistance, dry contacts, and increasingly **fully digital field buses** (see below).

---

## 9. Control Strategies & Sequences of Operation (SOO)

The SOO is the written logic that ties it together — the HVAC equivalent of your functional spec. Core strategies:

- **PID loops** — the bread and butter (SAT control, static pressure, DP, valve position). Same tuning principles you know; HVAC loops are generally slow and forgiving.
- **Economizer / free cooling** — dry-bulb or enthalpy comparison decides when OA replaces mechanical cooling. High-limit lockout prevents bringing in hot/humid air.
- **Reset schedules** — trim setpoints against load or weather: chilled-water reset, hot-water reset, supply-air-temp reset, static-pressure reset, condenser-water reset. *Biggest energy lever in the building.*
- **Demand-Control Ventilation (DCV)** — modulate OA against CO₂ instead of running design ventilation constantly.
- **Optimal start/stop, night setback, occupancy scheduling.**
- **Staging & lead-lag** — rotate multiple chillers/boilers/pumps/towers for runtime balancing and capacity matching.
- **Safety interlocks** — freezestat, smoke detection (shutdown via fire alarm), high static cutout, low-water cutoff. These are hardwired/independent of the optimizing logic — same philosophy as your SIS/BPCS separation under IEC 61511. (HVAC life-safety follows NFPA 90A, 92, etc., not 61511, but the architectural principle of keeping protection independent is identical.)

**ASHRAE Guideline 36** is worth knowing: it's the standardized "high-performance sequences of operation" for VAV systems — essentially a vetted, copy-pasteable SOO library that's becoming the industry baseline.

---

## 10. The Control Hierarchy — Where HVAC Lives

```
   ┌──────────────────────────────────────────────┐
   │  SUPERVISORY / BMS HEAD-END (BAS front-end)   │  ← Niagara, Metasys, 
   │  Scheduling, trending, alarms, dashboards     │    Desigo, EcoStruxure,
   └───────────────┬──────────────────────────────┘    or your 800xA / SCADA
                   │  IP backbone (BACnet/IP, Modbus TCP, OPC UA, MQTT)
   ┌───────────────┴──────────────────────────────┐
   │  PLANT / SYSTEM CONTROLLERS (DDC)             │  ← chiller plant mgr,
   │  Sequences of operation, PID, staging         │    AHU controllers, PLCs
   └───────────────┬──────────────────────────────┘
                   │  Field bus (BACnet MS/TP, Modbus RTU, LonWorks)
   ┌───────────────┴──────────────────────────────┐
   │  UNITARY / TERMINAL CONTROLLERS               │  ← VAV box, FCU, RTU,
   │  Local loops                                  │    CRAC iCOM controllers
   └───────────────┬──────────────────────────────┘
                   │  Hardwired I/O + smart field bus
   ┌───────────────┴──────────────────────────────┐
   │  FIELD DEVICES — sensors, valves, dampers     │
   └──────────────────────────────────────────────┘
```

**DDC** = Direct Digital Control, the generic term for the controllers running HVAC logic. In commercial HVAC these are purpose-built DDC controllers (Distech, ALC, JCI, Siemens); in industrial/process facilities — your world — they're often **PLCs** (your Beckhoff, ABB AC 800M, Allen-Bradley) doing the same job.

---

## 11. Communication Protocols — The Part You Care About

This is where HVAC and your automation background fully overlap. Protocols cluster by layer.

### 11.1 BACnet — the HVAC native language
**BACnet (ASHRAE 135 / ISO 16484-5)** is *the* building-automation protocol — vendor-neutral, designed specifically for HVAC/BMS. You'll see it everywhere.

- **BACnet/IP** — runs over Ethernet/UDP; the backbone for plant controllers, chillers, large equipment, and head-ends. Uses a **BBMD** to route across IP subnets.
- **BACnet MS/TP** (Master-Slave/Token-Passing) — runs over **RS-485** twisted pair, typically up to 76.8 kbps (some go to 115.2). The field-bus level: VAV boxes, FCUs, unitary controllers daisy-chained on a trunk. *This is what your Carrier units used.*
- **BACnet/SC** (Secure Connect) — newer, TLS-secured, WebSocket-based; the modern secure replacement for BACnet/IP, getting around the security holes in legacy BACnet.
- **Data model:** everything is an **Object** (Analog Input, Analog Value, Binary Output, Schedule, Trend Log…) with standardized **Properties** (Present_Value, Status_Flags, Priority_Array…). The **priority array** (16 levels) is BACnet's elegant way of arbitrating who commands an output — operator override beats schedule beats default.
- **Discovery:** "Who-Is / I-Am" broadcasts make devices largely self-describing.

### 11.2 Modbus — the industrial lingua franca
**Modbus** is simple, ancient, ubiquitous, and the protocol you've been using on the **CI867A**. No object model — just registers.

- **Modbus RTU** — serial over **RS-485** (sometimes RS-232), binary framing, master/slave. Common on chillers, meters, VFDs, and the Vertiv IS-UNITY card.
- **Modbus TCP** — Modbus framing over Ethernet; what you used to land Trane chillers and Vertiv units into 800xA via the CI867A.
- **Data model:** four register types — **Coils** (1-bit RW), **Discrete Inputs** (1-bit RO), **Input Registers** (16-bit RO), **Holding Registers** (16-bit RW). You live and die by the **vendor register map** — there's no self-description, so you scale and interpret every register by hand (watch for 32-bit values split across two registers, word/byte order, signed vs unsigned).
- **Why it's everywhere:** dead simple, royalty-free, every device supports it. **Why it's painful:** no standardization above the register, no built-in scaling, no discovery, weak security.

### 11.3 LonWorks (LON)
**LonTalk / ANSI 709.1**, Echelon's protocol. Was BACnet's big rival in the '90s–2000s. Peer-to-peer, uses **SNVTs** (Standard Network Variable Types) for interoperability, runs on twisted pair (FT-10) or IP. Still found in installed base (especially lighting, some European gear) but declining — BACnet largely won the BMS war.

### 11.4 KNX
European building-automation standard (ISO/IEC 14543). Strong in lighting/blinds/HVAC in Europe and high-end residential. Twisted pair, RF, or IP. Less common in North American commercial HVAC.

### 11.5 The IT/IIoT layer — increasingly relevant
- **OPC UA** — your bridge between HVAC/BMS and the SCADA/MES/historian world. Secure, platform-neutral, rich information model. More and more BMS head-ends and gateways expose **OPC UA** servers; you'd consume them into Ignition or 800xA exactly as you do process data.
- **MQTT** (often with **Sparkplug B**) — lightweight pub/sub for IIoT and smart-building telemetry; pairs naturally with Ignition's MQTT modules.
- **SNMP** — used for monitoring infrastructure gear (the Vertiv IS-UNITY card speaks it; data center DCIM platforms lean on it heavily for power/cooling/UPS).
- **Niagara Framework / Fox / oBIX / Haystack** — Tridium **Niagara** is a near-universal *integration* platform (the "JACE" controller) that normalizes BACnet + Modbus + LON + everything else into one tagged model (**Project Haystack** tagging); **oBIX** is its web-services API. If you walk into a commercial site, odds are good there's a Niagara JACE gluing it together.

### 11.6 Protocol cheat-sheet

| Protocol | Physical | Role | Where you'll meet it |
|---|---|---|---|
| BACnet/IP | Ethernet/UDP | Supervisory + plant | Chillers, head-ends, AHU controllers |
| BACnet MS/TP | RS-485 | Field bus | VAV boxes, FCUs, unitary controllers |
| BACnet/SC | Ethernet/TLS | Secure supervisory | Modern/secure deployments |
| Modbus RTU | RS-485 | Device-level | Chillers, VFDs, meters, Vertiv card |
| Modbus TCP | Ethernet | Device-level | **Your CI867A integrations** |
| LonWorks | Twisted pair/IP | Field/peer | Legacy BMS, lighting |
| KNX | TP/RF/IP | Building automation | European/residential |
| OPC UA | Ethernet/TLS | IT↔OT bridge | SCADA/historian integration |
| MQTT | Ethernet | IIoT telemetry | Smart buildings, Ignition |
| SNMP | Ethernet | Infra monitoring | Data center power/cooling, DCIM |
| Niagara Fox | Ethernet | Integration framework | Multi-vendor commercial sites |

### 11.7 Gateways & protocol converters (your Babel Buster world)
Real buildings are protocol zoos, so **gateways** are everywhere. You've used **FieldServer/Babel Buster**-type converters — exactly the right tool: they map, say, a Modbus RTU chiller into BACnet/IP, or a BACnet MS/TP trunk into Modbus TCP for a DCS. The Vertiv **IS-UNITY-DP** is itself a multi-protocol gateway. Key gotchas you already know from process work: register mapping fidelity, scaling/engineering-unit conversion, polling rate vs point count, address conflicts, and byte/word order on 32-bit values.

---

## 12. Key Terms, Units & Numbers Worth Memorizing

- **Ton of refrigeration** = 12,000 BTU/hr = 3.517 kW of cooling. (One ton = melting one ton of ice in 24 hr.)
- **CFM** — cubic feet per minute (airflow). **GPM** — gallons per minute (water flow).
- **"Inches of water column" (in. WC / "WG)** — the tiny pressures of air systems (duct static often 0.5–3 "WC).
- **Delta-T (ΔT)** — temperature difference; design CHW ΔT ≈ 10–12 °F, HW ΔT ≈ 20–40 °F, CW ΔT ≈ 10 °F.
- **Sensible Heat Ratio (SHR)** — sensible / total cooling. Data centers ≈ 0.9–1.0; humid comfort cooling ≈ 0.7.
- **Approach** (chiller/tower), **Range** (tower), **Lift** (chiller pressure differential).
- **Economizer**, **reset**, **lead-lag**, **staging**, **deadband**, **throttling range**.
- **kW/ton** — chiller efficiency (lower is better; good water-cooled centrifugal ≈ 0.5–0.6 kW/ton).
- **COP / EER / SEER / IPLV** — efficiency metrics (COP dimensionless, EER/SEER in BTU/Wh, IPLV = integrated part-load value).
- **Wet-bulb / dry-bulb / dewpoint / enthalpy** — psychrometric properties; learn to read a **psychrometric chart**, it's the single most useful HVAC mental tool.

---

## 13. How to Tie It Back to What You Know

The honest summary for someone with your background: **HVAC is process control with slower loops, gentler consequences, and its own vocabulary.** A chiller plant is a unit operation. An AHU is a small process line (filtration → heat exchange → fluid transport) with cascaded PID. VAV boxes are distributed flow controllers. The "exotic" part is mostly (a) the psychrometrics — air carries latent heat, which trips up people used to single-phase liquid processes — and (b) BACnet, which is genuinely different from the register-based protocols you've lived in, because it carries a real object/property model and a priority-array command-arbitration scheme that Modbus simply doesn't have.

Your existing toolkit maps almost one-to-one:
- 4–20 mA / HART / RTD instrumentation → same field layer.
- PID, feedforward, cascade → same loops, slower.
- Modbus TCP on CI867A → same integration path for chillers/CRACs.
- IEC 61511 protection-layer independence → same philosophy as HVAC life-safety interlocks (freezestat, smoke shutdown).
- OPC UA / Ignition / SQL historian → exactly how you'd trend and store BMS data.

The one genuinely new study item is **BACnet's object model + priority array**, and after that, **psychrometrics**. Nail those two and the rest is familiar territory wearing different nameplates.

---

*Reference compiled for an automation/controls engineer. For authoritative design values, always defer to ASHRAE Handbooks (Fundamentals, HVAC Systems & Equipment, HVAC Applications), ASHRAE Guideline 36 for sequences, and the specific equipment vendor's integration/register documentation.*