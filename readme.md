# Compact Screw Press Dewatering Controller (Omron CP1E)

[![Bahasa Indonesia](https://img.shields.io/badge/Language-Bahasa%20Indonesia-brightgreen)](README_ID.md)
*Available in other languages: [🇮🇩 Versi Bahasa Indonesia](README_ID.md)*

An industrial PLC automation program developed in **Omron CX-Programmer** for an automated **Screw Press Dewatering Separator** ("Auger Press / Waste Compactor"). 

This design implements the dewatering process principles established in academic research on manure separators (*Senfter et al., Separations 2024*) and addresses real-world maintenance challenges from field teardowns (screen blinding, flush solenoid automation, and cake plug jams).

---

## 📌 Project Overview & Physics

A screw press separates liquid filtrate from thick slurry/manure via a continuous auger rotating inside a cylindrical wedge-wire screen. Dewatering efficiency relies heavily on the **mechanical counter-pressure** maintained at the discharge cone/choke:
* **Too little back-pressure:** Wet uncompacted slurry blows through the exit without dewatering.
* **Too much back-pressure:** Screen blinds, auger shaft torque spikes, and the motor stalls/jams.

This controller automates the process using closed-loop analog flow feedback and a simple directional command interface to an electric linear actuator.

---

## ⚙️ Hardware Architecture

| Component | Industry Specification / Model | PLC Interface | Role in Process |
| :--- | :--- | :--- | :--- |
| **PLC** | **Omron CP1E** | Built-in CPU + Analog | Master Controller |
| **Flow Sensor** | **Allen-Bradley 839E** Solid-State Flow Transmitter | 4–20 mA Current Loop into `CIO 90` | Monitors filtrate discharge rate; detects screen blinding & dry run |
| **Actuator** | **Allen-Bradley MPAR** Kinetix Electric Cylinder | Discrete outputs: `100.02` (Ext), `100.03` (Ret) | Dynamically regulates the discharge choke flap back-pressure |
| **Auger Drive** | Induction Motor / Inverter Drive | Digital output `100.00` | Drives the main compression auger shaft |
| **Flush Solenoid** | 2-Way Water Solenoid Valve | Digital output `100.01` | Screen backwash spray nozzles for CIP and purge |

---

## 🗺️ PLC I/O & Memory Allocation Table

### Physical Inputs (`CIO 0` & `CIO 90`)
| Address | Symbol | Type | Description |
| :--- | :--- | :--- | :--- |
| `0.00` | `PB_START` | Digital (NO) | Cycle Start Pushbutton |
| `0.01` | `PB_STOP` | Digital (NC) | Cycle Stop Pushbutton |
| `0.02` | `ESTOP_OK` | Digital (NC) | Emergency Stop Safety Circuit Healthy |
| `0.03` | `AUGER_OL_OK` | Digital (NC) | Motor Overload / VFD Fault OK Contact |
| `CIO 90`| `FLOW_ANALOG` | Analog In | Raw Flow Rate from Flow Transmitter (0–6000 = 4–20 mA) |

### Memory Setpoints & Internal Bits (`D` & `W`)
| Address | Symbol | Description |
| :--- | :--- | :--- |
| `D10` | `FLOW_SP_LOW` | Low-Flow Alarm Threshold Setpoint (e.g. `&600` = 10%) |
| `W0.00` | `SYS_RUN` | Master System Running Latch |
| `W0.01` | `SEQ_PRECHOKE` | Phase 1: Pre-Choke / Plug Setup Active |
| `W0.02` | `SEQ_DEWATER` | Phase 2: Active Dewatering Run |
| `W0.03` | `JAM_RELIEF` | Phase 3: Auto Cake-Plug Jam Relief Active |
| `W0.04` | `SEQ_SHUTDOWN` | Phase 4: Purge & Screen Washdown Cycle |
| `W0.05` | `ALM_LOW_FLOW` | Low Filtrate / Blinding Flow Alarm Flag |

### Physical Outputs (`CIO 100`)
| Address | Symbol | Description |
| :--- | :--- | :--- |
| `100.00` | `AUGER_RUN` | Auger Motor Contactor / VFD Run Command |
| `100.01` | `FLUSH_SOLENOID` | Wedge-Wire Screen Washdown Water Valve |
| `100.02` | `ACTUATOR_EXT` | Electric Cylinder Extend (Increase Back-Pressure) |
| `100.03` | `ACTUATOR_RET` | Electric Cylinder Retract (Relieve Back-Pressure) |
| `100.04` | `ALARM_BEACON` | Audible/Visual Alarm Beacon |

### Timers
| Timer | Preset | Duration | Function |
| :--- | :--- | :--- | :--- |
| `TIM 000` | `#50` | 5.0 s | Startup flow stabilization grace period |
| `TIM 001` | `#40` | 4.0 s | Actuator auto-retract duration to clear cake jams |
| `TIM 002` | `#150` | 15.0 s | Washdown flush and barrel cleanout purge on stop |
| `TIM 003` | `#30` | 3.0 s | Initial actuator travel to establish pre-choke restriction |

---

## 🔄 State Machine & Operational Sequence

```
                      [ E-Stop / Standby ]
                               │ (PB_START 0.00)
                               ▼
                   [ Phase 1: Pre-Choke ]
       (Actuator extends for 3.0s to set initial restriction)
                               │ (TIM 003 Done)
                               ▼
                [ Phase 2: Dewatering Run ]
         (Auger runs; Flow Sensor monitors filtrate flow)
          │                                         │
 (Auger Overload 0.03 = 0)              (Flow < D10 Setpoint)
          │                                         │
          ▼                                         ▼
[ Phase 3: Jam Relief ]                    [ Pulse Wash Solenoid ]
(Actuator retracts 4.0s to                 (Backwash screen to
 release plug; resets motor)                clear cake blinding)
                               │ (PB_STOP 0.01)
                               ▼
                 [ Phase 4: Shutdown Purge ]
        (Actuator opens; Flush Solenoid & Auger run 
         for 15.0s to prevent cake hardening inside)
```

---

## 🛠️ Ladder Logic (CX-Programmer Instruction List)

```text
// Rung 0: Master Start/Stop Latch with Safety Interlocks
LD      0.02
AND     0.03
AND-NOT 0.01
LD      0.00
OR      W0.00
OR      W0.04
AND-LD
OUT     W0.00

// Rung 1: Phase 1 - Pre-Choke Plug Formation
LD      W0.00
AND-NOT TIM003
AND-NOT W0.02
AND-NOT W0.04
OUT     W0.01
TIM     003     #0030

// Rung 2: Phase 2 - Active Dewatering Run
LD      W0.00
AND     TIM003
AND-NOT W0.03
AND-NOT W0.04
OR      W0.02
OUT     W0.02

// Rung 3: Flow Stabilization Delay Timer
LD      W0.02
TIM     000     #0050

// Rung 4: Flow Sensor Analog Comparison & Alarm
LD      W0.02
AND     TIM000
AND<    90      D10
OUT     W0.05

// Rung 5: Phase 3 - Auto Jam-Relief Routine
LD      W0.02
AND-NOT 0.03
LD      W0.03
AND-NOT TIM001
OR-LD
OUT     W0.03
TIM     001     #0040

// Rung 6: Phase 4 - Controlled Shutdown & Screen Purge
LD      0.01
AND     W0.00
AND-NOT TIM002
LD      W0.04
AND-NOT TIM002
OR-LD
OUT     W0.04
TIM     002     #0150

// Rung 7: Physical Output - Auger Motor
LD      W0.02
AND-NOT W0.03
OR      W0.04
OUT     100.00

// Rung 8: Physical Output - Flush Water Solenoid
LD      W0.04
LD      W0.02
AND     W0.05
OR-LD
OUT     100.01

// Rung 9: Physical Output - Actuator Extend
LD      W0.01
OUT     100.02

// Rung 10: Physical Output - Actuator Retract
LD      W0.03
OR      W0.04
OUT     100.03

// Rung 11: Physical Output - Alarm Beacon
LD      W0.03
OR      W0.05
OUT     100.04

END
```

---

## 🧪 Simulation & Testing Guide (CX-Simulator Step-by-Step)

Follow these exact steps to test the logic entirely inside CX-Programmer without any physical hardware.

### Essential Windows & Shortcuts Checklist
* **Compile Program:** `Ctrl + F7` *(Make sure Output Window says `0 errors, 0 warnings`)*
* **Work Online Simulator:** `Ctrl + Shift + W` *(Starts the virtual CP1E PLC)*
* **Project Workspace (Left Tree):** `Alt + 1` *(Or `View` > `Windows` > `Project Workspace`)*
* **Watch Window:** `Alt + 3` *(Or `View` > `Windows` > `Watch`)*
* **PLC Operating Mode:** Ensure the status bar at bottom says **MONITOR** or **RUN** *(Press `Ctrl + M` if stuck in PROGRAM mode)*

---

### Step 1: Pre-Flight Setup (Setting Memory & Watch Window)

Before toggling start buttons, configure your initial analog setpoint and healthy hardware statuses:

1. **Open the Watch Window (`Alt + 3`)**:
   * Click the first row under the **Name** column, type `90` (Flow Sensor Analog Input), and hit `Enter`.
   * Click the next row, type `D10` (Low-Flow Threshold Setpoint), and hit `Enter`.
   * Click the next row, type `W0.05` (Alarm Flag), and hit `Enter`.
2. **Preset Values in Watch Window**:
   * Double-click the **Value** cell for **`D10`**, type `600` (or `&600` in decimal), and hit `Enter`.
   * Double-click the **Value** cell for **`90`**, type `1200` (or `&1200` in decimal), and hit `Enter` (representing healthy initial filtrate flow > 600).
3. **Set Normally-Closed Safety Inputs to Healthy (1)**:
   * On Rung 0, double-click contact **`0.02` (`ESTOP_OK`)** $\rightarrow$ change value to `1` $\rightarrow$ `Enter`.
   * On Rung 0, double-click contact **`0.03` (`AUGER_OL_OK`)** $\rightarrow$ change value to `1` $\rightarrow$ `Enter`.

---

### Step 2: Test Start & Pre-Choke Phase (Phase 1 $\rightarrow$ 2)

1. Double-click contact **`0.00` (`PB_START`)**, set to `1`, hit `Enter`.
2. Immediately double-click **`0.00`** again, set back to `0` (simulates releasing a momentary button).
3. **Verify:**
   * `W0.00` (`SYS_RUN`) turns green and latches ON.
   * `W0.01` (`SEQ_PRECHOKE`) turns ON.
   * Output coil **`100.02` (`ACTUATOR_EXT`)** energizes to push the choke door into position.
   * Timer **`TIM 003`** counts down 3.0 seconds (`#30` $\rightarrow$ `0`).
4. **When `TIM 003` reaches 0:**
   * `100.02` (`ACTUATOR_EXT`) turns OFF.
   * `W0.02` (`SEQ_DEWATER`) turns ON and latches.
   * Output coil **`100.00` (`AUGER_RUN`)** energizes (motor running).
   * Timer **`TIM 000`** begins its 5.0-second flow stabilization grace period (`#50` $\rightarrow$ `0`).

---

### Step 3: Test Analog Low-Flow & Screen Blinding Alarm (Phase 2)

1. Wait until **`TIM 000`** finishes counting down to `0` (contact `T000` glows green).
2. Go to the **Watch Window (`Alt + 3`)**:
   * Double-click the **Value** for address **`90`**.
   * Change it from `1200` down to **`200`** (or `&200`) and hit `Enter`.
3. **Verify:**
   * The comparison condition `90 < D10` ($200 < 600$) becomes TRUE.
   * In the Watch Window and Ladder, **`W0.05` (`ALM_LOW_FLOW`)** turns ON (value = `1`, green dot inside coil).
   * Output coil **`100.01` (`FLUSH_SOLENOID`)** energizes to open washdown spray nozzles.
   * Output coil **`100.04` (`ALARM_BEACON`)** lights up.
4. **Clear Alarm:**
   * In the Watch Window, change `90` back to **`1200`**.
   * `W0.05`, `100.01`, and `100.04` turn OFF immediately.

---

### Step 4: Test Overload & Auto Cake Jam Relief (Phase 3)

1. While dewatering (`W0.02` is active and `100.00` is ON), simulate an auger motor overload:
   * Double-click contact **`0.03` (`AUGER_OL_OK`)**, set to `0` (simulates motor thermal trip / jam).
2. **Verify:**
   * Output **`100.00` (`AUGER_RUN`)** immediately drops out (stops the motor).
   * Internal bit `W0.03` (`JAM_RELIEF`) energizes and latches.
   * Output coil **`100.03` (`ACTUATOR_RET`)** energizes to pull the choke door back and dump cake pressure.
   * Timer **`TIM 001`** counts down for 4.0 seconds (`#40` $\rightarrow$ `0`).
3. **Reset Overload:**
   * Set contact **`0.03`** back to `1` (simulates resetting motor protector).
   * Once `TIM 001` completes, `W0.03` and `ACTUATOR_RET` turn OFF, allowing normal dewatering to resume.

---

### Step 5: Test Stop & Shutdown Purge Flush (Phase 4)

1. Double-click contact **`0.01` (`PB_STOP`)**, set to `1`, then set back to `0`.
2. **Verify:**
   * `W0.00` drops out of normal run mode.
   * `W0.04` (`SEQ_SHUTDOWN`) latches ON.
   * Output **`100.03` (`ACTUATOR_RET`)** energizes to open the discharge chute completely.
   * Output **`100.01` (`FLUSH_SOLENOID`)** and **`100.00` (`AUGER_RUN`)** remain energized.
   * Timer **`TIM 002`** counts down 15.0 seconds (`#150` $\rightarrow$ `0`), flushing sludge out of the basket so it doesn't dry into a cement plug.
3. **When `TIM 002` hits 0:**
   * All outputs de-energize and the machine safely returns to Standby.

---

## 📚 References
* **Research Paper:** Senfter, T., Schweiggl, I., Berger, M., Mayerl, C., Kofler, T., Kraxner, M., Steffens, A., & Pillei, M. (2024). *The Dewatering Performance of a Compact Screw Press Manure Separator for Non-Typical Substrates*. **Separations**, 11(1), 28. [https://doi.org/10.3390/separations11010028](https://doi.org/10.3390/separations11010028)
* **Teardown Reference:** AvE - *Fixing the Poop Screw* (Industrial waste compactor auger & flush solenoid troubleshooting): [YouTube](https://youtu.be/6O5Lpm2iE44?t=374)
