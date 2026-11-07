# CLAUDE.md — Project Context & Engineering Guidelines for `flux-shaft`

## 1. Project Overview
`flux-shaft` (**FluxShaft**) is an R&D and marine drivetrain engineering repository for **4-mode parallel-hybrid electric propulsion, stationary & underway diesel generation, and under-sail hydro-regeneration** on **`30+ ft` (`48 V`, e.g., Hunter 326 / Hunter 35)** and **`40+ ft` (`96 V`, e.g., 2009 Beneteau Oceanis 40)** shaft-drive sailboats.

Primary reference document: [`parallel-hybrid-propulsion-regen-guide.md`](parallel-hybrid-propulsion-regen-guide.md).

---

## 2. Core Engineering Rules & Drivetrain Constraints (Do Not Violate)

When generating mechanical CAD, pulley/gear calculations, electrical schematics, controller maps, or BOMs in this repository, always adhere to these physical and regulatory constraints:

1. **Reference `40+ ft` Diesel & Marine Gearbox Specs (`Yanmar 4JH4-AE` + `Kanzaki KM35`):**
   * The reference `40+ ft` auxiliary diesel (`Beneteau 40` class) is the naturally aspirated `2.19 L` **Yanmar 4JH4-AE**, rated at **`39.6 kW` (`53.8 mhp` / `~54 SAE HP`) maximum at `3,000 RPM`** and **`36.0 kW` (`48.3 HP`) continuous at `2,907 RPM`** (`~38.5 kW` max at the shaft after `3%` gearbox loss). **Never use `3,800 RPM` for the `4JH4-AE`.**
   * Its crankshaft torque curve is flat at **`124–127 N·m` from `1,800 to 3,000 RPM`**.
   * Paired transmission is the mechanical cone-clutch **Kanzaki `KM35P`** (`2.36:1` or `2.61:1` forward, giving **`1,271` or `1,149 shaft RPM` max** at `3,000 RPM`) or **`KM35A2`** (`7°` down-angle, `2.33:1` or `2.64:1` forward, giving **`1,288` or `1,136 shaft RPM` max**).
   * Per Yanmar service advisory **`MSA08-003`**, `KM`-series mechanical cone-clutch gearboxes **must be left in `NEUTRAL`** when the shaft is spun by the water (sailing regen) or by an aft electric motor. **Never design a topology that back-drives a Kanzaki `KM35` in Forward gear with the diesel off**, as reverse torque un-wedges the helical mechanical cone clutch.

2. **Mandatory 2-Clutch Architecture for Stationary Anchor Generation (Mode 1):**
   * To support all 4 modes—including **Mode 1: Stationary Diesel Genset at Anchor/Idle with the propeller stopped**—the electric motor takeoff must sit *between* **`Clutch 1` (the stock `KM35`/`KM2P` transmission)** and **`Clutch 2` (an aft shaft square-jaw dog disconnect clutch)**.
   * Whenever `Clutch 2` interrupts the propeller shaft, a **dedicated marine thrust bearing (`Python Drive`, `Aquadrive`, or flanged double-tapered roller pillow block)** must be mounted *aft* of `Clutch 2` (or via a concentric pilot-bearing through-shaft) so forward/reverse propeller thrust never loads the dog clutch teeth axially, and **dual pillow-block bearings** must support the intermediate stub shaft to absorb the `200–400+ lbf` radial belt tension.
   * Mechanical dog clutches (`Clutch 2` and optional motor-pad `Clutch 3`) must use **straight square jaws (never spiral jaws)** and be interlocked to **engage at `0 RPM` only**.

3. **Motor Selection & Reduction Ratio Physics (`ME1616` vs. `ME1304`):**
   * Prefer the **Motenergy `ME1616`** (`~$1,280`, radial-flux `IPM`, `IP67`, `45–51 N·m` continuous / `125 N·m` peak, `NEMA 143T` flange, `7/8"` [`22.2 mm`] keyed shaft, $K_t \approx 0.22\text{ N·m/A}$, $K_v \approx 45\text{ RPM/V}$) over the axial-flux `ME1304` (`~$965`, `22–25 N·m` cont.) because **`1 × ME1616` matches `2 × ME1304` on continuous shaft torque** (`~50 N·m`), eliminating the second controller and twin-pad pump-drive gearbox, while **`2 × ME1616` (`~100 N·m` cont.)** delivers full `25.4–38.5 kW` continuous diesel-replacement power.
   * **Target Belt Reduction (`Beneteau 40`):** **`2.5:1` to `3.0:1`** using a **`Gates PolyChain GT Carbon 8M` (`36 mm` wide)** polyurethane/carbon timing belt (`28T` motor pulley $\rightarrow$ `70T` [`178.3 mm` PD] or `84T` [`213.9 mm` PD] shaft pulley). At `3,000 RPM` diesel WOT, `3.0:1` spins the motor at **`3,448–3,814 RPM`**, keeping unloaded back-EMF (`76.6–84.8 V`) safely below the `96 V` (`102.4 V` nominal) battery bus without uncontrolled diode rectification.

4. **Diesel-Driven Charging Curve ("Golden Zone" at `2,000–2,200 RPM`):**
   * Because `4JH4-AE` available power scales linearly ($\propto \text{RPM}$) while propeller load scales cubically ($\propto \text{RPM}^3$), surplus diesel shaft power peaks at **`2,000–2,200 RPM`** (`10.5–14.3 kW` surplus $\implies$ **`9.1–12.1 kW DC` net battery charging** on `1 × ME1616`).
   * Always program the `Sevcon Gen4` diesel-charging torque map to **taper braking torque down above `950 shaft RPM` (`~2,400 diesel RPM`)**, reaching `0 N·m` by `1,100 shaft RPM` (`~2,750 diesel RPM`) so the generator never overloads the diesel at WOT.

5. **High-Voltage (`96 V`) Marine Electrical Safety (`ABYC E-30` & `ISO 16315:2026`):**
   * **`48 V` systems (`16S LiFePO4`, `51.2 V` nom / `58.4 V` max)** remain inside **ABYC E-11 (`≤60 V DC`)**.
   * **`96 V` systems (`32S LiFePO4`, `102.4 V` nom / `116.8 V` max)** exceed `60 V DC` (**ABYC E-30**) and `50 V DC` (**ISO 16315:2026**) and **must** enforce:
     1. **100% Floating Bus:** Never bond `96 V-` to `12 V` house negative, engine block, or hull ground. Use an isolated `96 V → 12 V` DC-DC converter and isolated controller logic supply.
     2. **Active Insulation Monitoring Device (`IMD`):** Include an `IEC 61557-8` `IMD` (`Bender ISOMETER iso165C`) between `96 V+`, `96 V-`, and hull ground (`500 Ω/V` warning, `100 Ω/V` trip).
     3. **Single `32S` Contactor BMS + Regen Load-Dump Cutback:** Never series-chain consumer `48 V` MOSFET BMS packs unless the MOSFETs are rated `≥150 V`. Use a single `32S` BMS (`Orion Jr. 2` / `REC Q-BMS`) with `900 V` `Gigavac`/`Kilovac` contactors, pre-charge circuit, and a hardwired/CAN **high-cell (`3.50 V/cell`) regen cutback signal** that ramps `Sevcon` braking current to `0 A` *before* the contactor opens.
     4. **DC Fusing:** All `96 V` fuses must be rated **`≥125 V–160 V DC`** with **`≥20,000 A` AIC** (`Class T` or `Bussmann BS88 EV`).
