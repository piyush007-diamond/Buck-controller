# Discrete High-Side Bootstrap Buck Converter (LTspice Simulation)

## Project Overview
This repository contains an LTspice simulation of an Asynchronous Buck Converter. Rather than relying on an integrated, black-box gate driver IC, this project implements a **fully discrete high-side gate driver utilizing a bootstrap capacitor circuit and a push-pull amplifier stage**. 

This circuit steps down a 24V DC input to a lower DC output voltage to drive a 5Ω load, operating at a switching frequency of 100kHz.

## Why Use a Gate Driver? (The High-Side N-Channel Problem)
In a standard buck converter, the main switching element is typically an N-channel MOSFET (`M1` in this schematic). N-channel MOSFETs are preferred over P-channel due to their lower ON-resistance ($R_{DS(on)}$) and faster switching characteristics, which minimize conduction and switching losses.

However, using an N-channel MOSFET on the **high side** (between the power supply and the load) creates a specific driving challenge:
To turn the N-channel MOSFET completely ON, the voltage at its gate must be significantly higher than the voltage at its source ($V_{GS}>V_{th}$). 
When `M1` turns ON, its source voltage rises rapidly to the input voltage (24V). To keep it ON, the gate voltage must be pushed to ~34V (24V + 10V drive). A standard microcontroller or logic signal cannot provide this voltage. 

**The Solution:** A discrete Bootstrap Gate Driver.

## Detailed Circuit Operation

### 1. The Bootstrap Network (`D2` and `C2`)
This is the heart of the high-side driver. 
* **Charging Phase:** When `M1` is OFF, the inductor `L1` freewheels through the catch diode `D1`, pulling the source of `M1` (the switching node) down to slightly below ground (approx. -0.4V). During this time, the bootstrap capacitor `C2` (100nF) charges from the 12V supply (`V2`) through the bootstrap diode `D2` (1N5817). 
* **Pumping Phase:** When `M1` needs to turn ON, the gate is connected to the top plate of `C2`. Because the voltage across a capacitor cannot change instantaneously, as the source of `M1` rises to 24V, the top plate of `C2` is "bootstrapped" up to 24V + 12V = 36V. This provides the necessary higher-than-supply voltage to maintain a strong $V_{GS}$ and keep `M1` saturated.

### 2. The Push-Pull Emitter Follower (`Q1` and `Q2`)
MOSFET gates act like tiny capacitors. To turn a MOSFET on and off quickly (to reduce switching losses), we need to inject and remove current from the gate very rapidly.
* The NPN (`Q1` - 2N2222) and PNP (`Q2` - 2N2907) bipolar transistors form a push-pull amplifier.
* When the control signal commands `M1` to turn ON, `Q1` conducts, rapidly dumping the charge from the bootstrap capacitor `C2` into the gate of `M1`.
* When commanded OFF, `Q2` conducts, rapidly discharging the gate capacitance back to the source, ensuring a crisp, fast turn-off.

### 3. The Level Shifter (`M2`)
The control signal (`V3`) is a standard 0-12V logic pulse referenced to ground. However, the push-pull stage is floating on top of the switching node voltage. `M2` (BSP89) acts as a level shifter, translating the ground-referenced PWM signal into a signal that the floating push-pull stage can use.

### 4. The Power Stage (`L1`, `D1`, `C1`)
* **`L1` (40µH):** Stores energy in its magnetic field during the ON-time and releases it to the load during the OFF-time.
* **`D1` (1N5817):** A Schottky diode used as the freewheeling/catch diode. It provides a path for the inductor current when `M1` turns off. Schottky is chosen for its low forward voltage drop and virtually zero reverse recovery time.
* **`C1` (22µF):** Output filter capacitor that smooths the voltage ripple to provide a stable DC output to the 5Ω load (`R3`).

---

## Waveform Analysis & Results

### 1. Inductor Current `I(L1)` (Continuous Conduction Mode)
*(Reference: `image_dacc4d.png`)*
The simulation shows a classic triangular current waveform fluctuating between **~0.8A and ~2.3A**.
* Because the current never drops to zero during the switching cycle, the converter is operating strictly in **Continuous Conduction Mode (CCM)**. 
* This is highly desirable for high-power applications as it reduces the peak current stresses on the components and lowers the output voltage ripple compared to Discontinuous Conduction Mode (DCM).

### 2. Switching Node Voltage `V(n002)`
*(Reference: `image_dac563.png`)*
This is the voltage at the junction of `M1`, `D1`, and `L1`. The waveform toggles cleanly between the 24V input and slightly below 0V.
* **Why does it go below 0V?** When `M1` turns off, the inductor resists the change in current. It forces the switching node voltage negative until the Schottky diode `D1` becomes forward-biased (clamping it at around -0.4V). 

### 3. Understanding Transients and Ringing
*(Reference: Differential voltage plots `image_db1ee5.png` & `image_db224c.png`)*
If you look closely at the switching transitions, you will notice sharp, high-frequency voltage spikes (ringing). **This is not a simulation error; it represents real-world physical behavior.**
* **The Cause:** Even though they aren't explicitly drawn on the schematic, every physical component has parasitic properties. The MOSFET has output capacitance ($C_{oss}$), the diode has junction capacitance, and the PCB traces/component leads have parasitic stray inductance ($L_{stray}$). 
* **The Effect:** During the extremely fast switching transients (high $dv/dt$ and $di/dt$), this parasitic inductance and capacitance form an underdamped RLC resonant tank circuit. The sharp edges "ring" this tank, causing the high-frequency oscillations seen on the leading and trailing edges of the pulses.
* **In Practice:** In physical hardware, this ringing can cause EMI (Electromagnetic Interference) issues or overvoltage stress on the MOSFET. It is usually mitigated by adding a small RC snubber circuit across the freewheeling diode.

### 4. Steady State vs. Start-up
The waveforms shown represent the **Steady State** operation of the converter. 
* During initial start-up, the output capacitor `C1` is completely uncharged. The inductor would experience massive inrush currents as it tries to charge the capacitor to the target voltage. 
* In steady state (achieved after a few milliseconds in this simulation), the energy put into the inductor during the ON-time perfectly balances the energy consumed by the load during the total switching period. The capacitor voltage and inductor current ripple stabilize into the repeating, predictable patterns shown in the results.

## How to Run the Simulation
1. Ensure you have [LTspice](https://www.analog.com/en/design-center/design-tools-and-calculators/ltspice-simulator.html) installed.
2. Clone this repository.
3. Open the `.asc` file in LTspice.
4. Click the "Run" icon.
5. Probe the schematic to view the currents and voltages described above.
