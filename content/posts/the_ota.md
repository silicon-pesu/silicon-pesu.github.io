---
title: "Scratch to OTA: Building and Analyzing the Classic 5-Transistor OTA"
date: "2026-08-17"
description: "Demystifying OTAs from Small-Signal Models to Cascode Topologies"
layout: "post"
authors:
    - name: "Pavan K Rao"
      url: "/members/pavan/"
    - name: "Keval Pattani"
      url: "/members/keval/"
tags:
    - "Analog"
    - "OTA"
    - "Amplifiers"
---

## Intro to OTA

Historically, analog designs relied heavily on standard Op-Amps. They act as ideal Voltage-Controlled Voltage Sources (VCVS). These designs were designed to have very low impedance so the manufacuturers can drive resistive loads on a PCB without the signal dying out. Applying this same method to tiny silicon at a sub-micron level inside an SoC doesn't work well.

Modern highly integrated silicon chips operate on a fundamentally different paradigm, especially when you are designing inside a monolithic chip, the environment changes completely. The load that an internal amp has to drive is not a resistor but it's usually the gate of another MOSFET. And at really high frequencies, a MOSFET gate acts almost entirely like a capacitor.

If you try to build a traditional Op-Amp inside an SoC, you have to add multi stage voltage buffers to force that low output impedance. This is incredibly inefficient and bummer as it eats up valuable silicon real estate, drains static power, and introduces extra parasitic poles that can ruin your control loop's stability.

Because SoC loads are capacitive, modern microchips use the **Operational Transconductance Amplifier (OTA)** instead. 

How it works?
-> Instead of a VCVS, an OTA is a **Voltage-Controlled Current Source (VCCS)**. It takes a differential voltage at its input and outputs a proportional current.
-> The behavior is defined by the transconductance ($g_m$) in the formula provided:
$$I_{out} = g_m \cdot \Delta V_{in} = g_m \cdot (V_{in+} - V_{in-}) \tag{1}$$
Pros of having it : by removing the bulky voltage buffers, the OTA stays extremely compact. It maintains a high output impedance, which is perfect for directly driving those capacitive MOSFET gates.

Also because the output variable is a current, the signal's dynamic frequency limits are no longer constrained by large RC time constants at intermediate circuit nodes. Instead, voltage transitions occur smoothly across the final capacitive load, utilizing the maximum current-driving capability of the circuit's active components. 

This Blog assumes the reader understands the basic operation of an NMOS and PMOS transistor as a three-terminal voltage-gated switch, and builds upon that foundation to develop a comprehensive understanding of advanced OTA architectures.

## Transistor Physics: From Switch to Amplifier

To fully master OTA architectures, we must analyze the underlying physics of individual MOS transistors. We shift our view from seeing them as simple on/off digital switches to treating them as continuous analog transconductance devices operating deep within their saturation regions.

![NMOS](https://images.kvlp.in/silicon/nmos_saturation.png)
>NMOS is in Saturation State

Consider an n-channel metal-oxide-semiconductor field-effect transistor (NMOS). When the voltage applied across its gate-to-source terminals (V<sub>GS</sub>) exceeds its intrinsic physical threshold voltage (V<sub>thn</sub>), an inversion layer of free electrons forms beneath the gate dielectric, creating a conductive channel. If the applied Drain-to-Source voltage (V<sub>DS</sub>) is driven high enough to satisfy the saturation condition (V<sub>DS</sub> ≥ V<sub>GS</sub> − V<sub>thn</sub>), the channel pinches off near the drain junction. In this saturation state, the drain current (I<sub>D</sub>) becomes largely independent of V<sub>DS</sub> to a first-order approximation, governed by the classical square-law relationship:
$$ I_D = \frac{1}{2} \mu_n C_{ox} \left( \frac{W}{L} \right) (V_{GS} - V_{thn})^2 (1 + \lambda V_{DS}) \tag{2} $$
Where μ<sub>n</sub> represents electron mobility, C<sub>ox</sub> is the gate oxide capacitance per unit area, W/L is the physical geometry aspect ratio (channel width divided by channel length), and λ represents the channel length modulation parameter. For analog circuit amplification, the critical metric is transconductance (g<sub>m</sub>), defined as the small-signal change in drain current resulting from an incremental change in gate-to-source voltage:
$$ g_m = \frac{\partial I_D}{\partial V_{GS}} = \mu_n C_{ox} \left( \frac{W}{L} \right) (V_{GS} - V_{thn}) = \sqrt{2 \mu_n C_{ox} (W/L) I_D} \tag{3} $$
Equation (3) highlights a foundational concept in analog IC design: the transconductance parameter can be dynamically tuned by altering the bias current (I<sub>D</sub>). In an OTA, this bias current is controlled globally by an external tail current source. This relationship enables electronic tuning of the system, allowing the circuit's transconductance and gain properties to be adjusted dynamically in real time.

## The Foundation Block: The Tail Current Source and Balanced Pair Layout

The foundational architecture of an OTA begins with a controlled current source feeding an input differential pair. The differential pair acts as a comparative engine that converts an input voltage difference into balanced current adjustments. Let us examine the design and construction of this building block.

![Foundational Block](https://images.kvlp.in/silicon/foundational_block.png)

At the base of the circuit sits the Tail Current Source, usually built using a wide NMOS transistor (M<sub>5</sub>) biased by a fixed voltage (V<sub>bias</sub>). This device is sized with a longer channel length (L) to maximize its output resistance (r<sub>o5</sub>), ensuring it behaves as a stable constant current sink that draws a fixed total tail current (I<sub>SS</sub>) down to the lower voltage rail (V<sub>SS</sub>). The output resistance of this tail current source directly impacts the circuit's ability to reject common-mode noise.

Directly above the tail current source are two matched NMOS transistors (M<sub>1</sub> and M<sub>2</sub>), whose sources connect at a shared junction (Tail Node P). When the input voltages V<sub>in+</sub> and V<sub>in−</sub> are perfectly balanced, the tail current splits equally between the two devices (I<sub>D1</sub> = I<sub>D2</sub> = I<sub>SS</sub>/2). 

If V<sub>in+</sub> increases relative to V<sub>in−</sub>, the conduction channel of M<sub>1</sub> widens, causing it to claim a larger share of the tail current, while the current through M<sub>2</sub> drops proportionally. This balanced differential transition forms the basis of the current-splitting mechanism inside the amplifier.

## Step-by-Step Evolution of the 5-Transistor Monolithic OTA

To convert the balanced current variations from our differential pair into a single-ended current output, we integrate a pull-up active load mirror at the upper voltage rail (V<sub>DD</sub>). This forms the classic 5-Transistor (5T) Monolithic OTA architecture.

![OTA](https://images.kvlp.in/silicon/OTA_final.png)

The upper current mirror uses two PMOS transistors (M<sub>3</sub> and M<sub>4</sub>). Device M<sub>3</sub> is diode-connected, meaning its gate is physically tied to its drain node (Internal Node X). This configuration forces M<sub>3</sub> to operate in saturation, automatically adjusting its gate voltage to handle whatever current M<sub>1</sub> draws through the left branch. Since M<sub>4</sub> shares the same gate and source connections as M<sub>3</sub>, it mirrors this current into the right branch. The output terminal (Node Y) combines the sourcing current from M<sub>4</sub> and the sinking current from M<sub>2</sub>, producing a single-ended output current.

## The Circuit Mechanism Under Dynamic Differential Operations

Let us analyze the current paths within the 5-Transistor OTA under dynamic differential input conditions. We examine how a changing voltage difference ΔV<sub>in</sub> = V<sub>in+</sub> − V<sub>in−</sub> steers current through the active branches of the circuit.

When a positive differential voltage is applied (ΔV<sub>in</sub> > 0), the gate voltage of M<sub>1</sub> rises while the gate voltage of M<sub>2</sub> falls. As a result, M<sub>1</sub> draws a larger portion of the total tail current (I<sub>SS</sub>), increasing the branch current I<sub>D1</sub>. This increased current passes through the diode-connected PMOS device M<sub>3</sub>, forcing its gate-to-source voltage to adjust. The current mirror copies this increased current over to M<sub>4</sub>, which pushes a larger sourcing current into the output node (I<sub>D4</sub> = I<sub>D1</sub>). At the same time, because V<sub>in−</sub> has decreased, M<sub>2</sub> turns off proportionally and draws less current (I<sub>D2</sub>). This creates a current imbalance at the output node: M<sub>4</sub> is sourcing more current than M<sub>2</sub> can sink, causing a positive net current to flow out of the amplifier into the external capacitive load:

$$I_{out} = I_{D4} - I_{D2} = I_{D1} - I_{D2} > 0 \tag{4}$$

Conversely, when a negative differential voltage is applied (ΔVin < 0), the current balancing flips. Transistor M<sub>1</sub> draws less current, which reduces the current passing through M<sub>3</sub> and drops the mirrored sourcing current of M<sub>4</sub> to a minimal value. Meanwhile, M<sub>2</sub> turns on harder, drawing a larger share of the tail current. Because M2 attempts to sink more current than M<sub>4</sub> can provide, it draws current directly from the external load capacitance. This reverses the output current flow, sinking current back into the amplifier substrate (I<sub>out</sub> < 0). This complementary action allows the OTA to drive capacitive nodes efficiently in both directions.

## Large-Signal Analytical Derivations & Slew Rate Boundaries

To characterize the large-signal limits of the 5T OTA, we will derive the individual branch currents mathematically. Let the differential input voltage be defined as ΔV<sub>in</sub> = V<sub>in+</sub> − V<sub>in−</sub>. The sum of the currents passing through the input devices is strictly limited by the tail current source:

$$I_{D1} + I_{D2} = I_{SS} \tag{5}$$

Using the square-law equations for transistors in saturation, we can express the difference in gate-to-source voltages as:

$$\Delta V\_{in} = V\_{GS1} - V\_{GS2} = \sqrt{\frac{2I\_{D1}}{\mu\_n C\_{ox}(W/L)\_{1,2}}} - \sqrt{\frac{2I\_{D2}}{\mu\_n C\_{ox}(W/L)\_{1,2}}} \tag{6}$$

By solving equations (5) and (6) simultaneously, we find expressions for the individual branch currents as functions of the differential input voltage:

$$I\_{D1} = \frac{I\_{SS}}{2} + \frac{I\_{SS}}{2} \sqrt{\frac{\mu\_n C\_{ox}(W/L)\_{1,2} \Delta V\_{in}^2}{I\_{SS}} - \left(\frac{\mu\_n C\_{ox}(W/L)\_{1,2} \Delta V\_{in}^2}{2I\_{SS}}\right)^2} \tag{7}$$

$$I\_{D2} = \frac{I\_{SS}}{2} - \frac{I\_{SS}}{2} \sqrt{\frac{\mu\_n C\_{ox}(W/L)\_{1,2} \Delta V\_{in}^2}{I\_{SS}} - \left(\frac{\mu\_n C\_{ox}(W/L)\_{1,2} \Delta V\_{in}^2}{2I\_{SS}}\right)^2} \tag{8}$$

The total single-ended output current is equal to $I_{D1} - I_{D2}$. When the input voltage difference becomes very large ($\vert{}\Delta V_{in}\vert{} \ge \sqrt{\frac{2 I_{SS}}{\mu_n C_{ox} (W/L)}}$), the current steering reaches its limit. One of the input transistors completely cuts off, and the entire tail current ($I_{SS}$) is routed down a single branch. Under these conditions, the output current saturates at its maximum limits, bounded by $\pm I_{SS}$.

This large-signal current limit determines the maximum Slew Rate (SR) of the OTA when driving a capacitive load, representing the absolute physical speed limit of the circuit under large transient swings:

$$SR = \left|\frac{dV\_{out}}{dt}\right|\_{max} = \frac{I\_{SS}}{C\_L} \tag{9}$$

## Small-Signal Equivalent Model Modeling & Analytical Gains

To understand the behavior of the OTA under small-signal conditions, we convert the large -signal circuit into its low-frequency small-signal equivalent model. This allows us to calculate the circuit's open-loop voltage gain and internal node impedances.

![small signal model](https://images.kvlp.in/silicon/small-signal-model.png)

Time to see how the physical transistors translate into this idealized AC model:

**The Differential Pair (Bottom):** The input transistors (M<sub>1</sub> and M<sub>2</sub>) act as the transconductance engines of the amplifier. We model them as voltage-controlled current sources ($g_{m1}v_{in+}$ and $g_{m2}v_{in-}$). To account for real-world physical imperfections, we place their intrinsic channel output resistances ($r_{o1}$ and $r_{o2}$) in parallel with these sources.

**The Diode-Connected Load (Top Left):** Because M3 has its gate tied to its drain, it acts much like a simple resistor in the AC domain. It presents a small equivalent input resistance of approximately $1/g_{m3}$. Because this resistance value is so small, **Node X** acts as a _low-impedance_ internal node.

**The Current Mirror (Top Right):** The mirroring PMOS transistor (M4) is represented by a dependent current source ($g_{m4}v_x$). It takes the voltage generated at Node X and drives current into the output.    

**Calculating Output Resistance ($R_{out}$)**

If you look into the output terminal (**Node Y**), you see a _high-impedance_ node. The total small-signal output resistance is simply the parallel combination of the intrinsic resistances of the two transistors directly connected to it (M<sub>2</sub> looking down to AC ground, and M<sub>4</sub> looking up to AC ground): 
$$R_{out} = r_{o2} \parallel r_{o4}$$

We know that $r_o = 1 / (\lambda I_D)$. Under perfectly balanced DC conditions, the total tail current splits equally between the two branches ($I_{D2} = I_{D4} = I_{SS}/2$). Substituting this in, we can see exactly how the output resistance relates to the physical parameters of the semiconductor process:

$$R_{out} = \frac{1}{\lambda_2 \cdot I_{D2}} \parallel \frac{1}{\lambda_4 \cdot I_{D4}} \approx \frac{2}{(\lambda_2 + \lambda_4) \cdot I_{SS}} \tag{10}$$

**The Open-Loop Voltage Gain ($A_v$)**

Finally, the total open-loop small-signal voltage gain of the OTA is found by multiplying the effective input transconductance by the total output resistance we just derived.

$$A_v = g_{m1} \cdot R_{out} = g_{m1} \cdot (r_{o2} \parallel r_{o4}) \tag{11}$$

This fundamental equation highlights a critical takeaway for IC design: the overall voltage gain of your amplifier is inextricably tied to the channel-length modulation parameters ($\lambda$) of your specific fabrication process.

## Characterization Metrics: The Seven Critical Parameters (Part I)

Analog IC designers characterize and optimize OTA architectures by evaluating seven core performance parameters. In this section, we analyze the first three metrics: Transconductance, Output Impedance, and Open-Loop Voltage Gain.

**Parameter 1: Overall Transconductance ($g_m$)**

The transconductance parameter defines how effectively the OTA converts an input voltage difference into an output current. It is measured with the output node connected to an AC short-circuit ($g_m = i_{out} / \Delta v_{in}$). In the classic 5T architecture, the overall transconductance is determined by the input devices (M<sub>1</sub> and M<sub>2</sub>). To maximize $g_m$ without increasing the bias current, designers expand the channel width ($W$), though this introduces larger parasitic capacitances at the input gates.

**Parameter 2: Intrinsic Output Impedance ($R_{out}$)**

Unlike standard operational amplifiers, an OTA is designed to maintain a high output impedance. This high impedance allows the circuit to function as a stable current source. The total output resistance is bounded by the parallel combination of the channel resistances of M<sub>2</sub> and M<sub>4</sub>. Increasing the physical channel lengths ($L$) of these devices reduces channel-length modulation, boosting the output impedance, but it also decreases the transconductance according to equation (3).

**Parameter 3: Open-Loop Voltage Gain ($A_v$)**

The open-loop voltage gain represents the maximum voltage amplification achievable when the OTA drives an open circuit or a purely capacitive load. It is defined as the product of the input transconductance and the output impedance ($A_v = g_m \cdot R_{out}$). In a single-stage 5T architecture, this gain is typically limited to a range of **30 dB** to **40 dB** due to short-channel effects in modern nanoscale nodes. Achieving higher voltage gains requires the use of advanced architectures, such as cascode topologies, to boost the output impedance without compromising transconductance.

## Characterization Metrics: The Seven Critical Parameters (Part II)

In this section, we complete our characterization framework by evaluating the remaining four performance metrics: Gain-Bandwidth Product, Slew Rate, Input Common-Mode Range, and Power Supply Rejection Ratio.

**Parameter 4: Gain-Bandwidth Product (GBW)**

When an OTA drives a load capacitance ($C_L$), the combination of the high output resistance ($R_{out}$) and $C_L$ forms a dominant low-frequency pole. The Gain-Bandwidth Product (GBW) defines the frequency range over which the amplifier can provide useful amplification, representing the point where the small-signal voltage gain drops to unity (**0 dB**):

$$GBW = \frac{g_{m1,2}}{2\pi C_L} \tag{12}$$

**Parameter 5: Slew Rate (SR)**

As derived in _Large-Signal Analytical Derivations & Slew Rate Boundaries_, the Slew Rate (SR) defines the maximum rate of change of the output voltage under large transient input conditions. It is limited by the total current available to charge or discharge the load capacitance ($SR = I_{SS} / C_L$). This creates a classic trade-off: increasing the slew rate requires a larger tail current, which increases the static power consumption of the circuit.

**Parameter 6: Input Common-Mode Range (ICMR)**

The ICMR defines the range of common-mode input voltages over which all transistors remain safely in their saturation regions. The lower limit is determined by the voltage needed to keep the tail current source M<sub>5</sub> saturated, while the upper limit is constrained by the requirement to keep the input devices M<sub>1</sub> and M<sub>2</sub> from entering the triode region as the input voltage rises toward $V_{DD}$.

**Parameter 7: Power Supply Rejection Ratio (PSRR)**

PSRR measures the amplifier's ability to reject noise or ripple on the power supply rails. In the 5T OTA, high-frequency noise on $V_{DD}$ can couple directly to the output node through the parasitic capacitances of the PMOS mirror load, which can degrade signal integrity in noisy mixed-signal environments.

## The Interdependent Trade-Off Matrix & Frequency Response

Analog IC design involves balancing multiple competing performance metrics. Optimizing one parameter often introduces trade-offs in others, requiring careful design balancing. Let us examine these relationships using a structured engineering matrix.

| **Design Decision / Action**           | **Transconductance (gm​)**          | **Output Resistance (Rout​)**  | **Slew Rate (SR)**           | **Dominant Pole (ωp​)**          | **Primary Engineering Trade-Off**                                 |
| -------------------------------------- | ----------------------------------- | ------------------------------ | ---------------------------- | -------------------------------- | ----------------------------------------------------------------- |
| **Increase Tail Current ($I_{SS}$)**   | Increases ($\propto \sqrt{I_{SS}}$) | Decreases ($\propto 1/I_{SS}$) | Increases ($\propto I_{SS}$) | Shifts Higher ($\propto I_{SS}$) | Elevates static power consumption across the substrate.           |
| **Increase Input Width ($W_{1,2}$)**   | Increases ($\propto \sqrt{W}$)      | No Change                      | No Change                    | No Change                        | Increases input gate parasitic capacitance, lowering input speed. |
| **Increase Output Length ($L_{2,4}$)** | No Change                           | Increases ($\propto L$)        | No Change                    | Shifts Lower ($\propto 1/L$)     | Narrows the dominant pole bandwidth, restricting open-loop speed. |
| **Increase Load Capacitance ($C_L$)**  | No Change                           | No Change                      | Decreases ($\propto 1/C_L$)  | Shifts Lower ($\propto 1/C_L$)   | Reduces both transient slewing speed and unity-gain bandwidth.    |

This matrix illustrates the core trade-offs in OTA design. For example, increasing the tail current to boost transconductance and slew rate inherently reduces the output resistance, which directly lowers the open-loop voltage gain (A<sub>v</sub> = g<sub>m</sub> · R<sub>out</sub>). Consequently, achieving high gain alongside high speed requires moving beyond simple single-stage topologies into advanced architectures designed to decouple these parameters.

## Advanced Design: Moving Beyond Single-Stage Limits

As fabrication technologies scale down into nanoscale regions, short-channel effects like channel length modulation become more pronounced, reducing the intrinsic output resistance (r<sub>o</sub>) of individual transistors. In these advanced nodes, a standard 5-Transistor OTA struggles to provide sufficient open-loop voltage gain for precision closed-loop applications. This limitation requires the development of advanced architectural topologies designed to boost gain and bandwidth performance.

To increase voltage gain, we can either stack transistors vertically or cascade multiple amplification stages horizontally. Stacking devices vertically leads to Cascode Topologies, which use common-gate shielding transistors to significantly increase the output impedance. Cascading stages horizontally leads to Multi-Stage Topologies, which amplify the signal through successive stages to achieve high gain, but require frequency compensation to maintain stability under closed-loop conditions. In the following sections, we will analyze four major advanced OTA architectures used in modern industrial design.

|**Architectural Topology**|**Description**|
|---|---|
|**5T Baseline**|Low-gain block for small loads.|
|**Cascode Stacking**|Multiplies output impedance for gain.|
|**Folded Cascading**|Folds signal paths for wider headroom.|
|**Two-Stage Multi-Path**|Separates gain and output swing stages.|

## Symmetrical Topologies: Balancing Bandwidth and Layout Symmetry

The first advanced architecture we examine is the Simple Symmetrical OTA. This design uses multiple current mirrors to decouple the input transconductance stage from the output loading node, improving layout symmetry and bandwidth performance.

```
                              [ VDD Supply Rail ]
                                       |
       +---------------+---------------+---------------+---------------+
       |               |                               |               |
     [M4]            [M3]                            [M9]            [M10]
    (PMOS)       (PMOS Diode)                    (PMOS Diode)        (PMOS)
       |               |                               |               |
       |               +---+                       +---+               |
       |               |   |                       |   |               |
       +-----Gate------+   |                       |   +------Gate-----+
       |                   |                       |                   |
       |                   |                       |                   |
       |                 [M1]                     [M2]                 |
       |          Vin+ --->| (NMOS)         (NMOS) |<--- Vin-          |
       |                   |                       |                   |
       |                   +-----------+-----------+                   |
       |                               |                               |
       |                             [M5]                              |
       |                               |<--- Vbias                     |
       |                             (NMOS)                            |
       |                               |                               |
       |                             [GND]                             |
       |                                                               |
     [M6]                                                            [M7]
	(NMOS Diode)                                                       (NMOS)
       |                                                               |
       +-------Gate----------------------------------------------------+
       |       |                                                       |
       +-------+                                                       +--> Output
       |                                                               |
     [GND]                                                           [GND]
```

In the Symmetrical OTA, the drain terminals of the input pair (M<sub>1</sub> and M<sub>2</sub>) drive two independent, diode-connected current mirrors (M<sub>3</sub> and M<sub>4</sub>). These mirrors can be scaled by a current multiplication factor (K = (W/L)<sub>mirror</sub> / (W/L)<sub>input</sub>) to multiply the signal current before it reaches the final output node. This structure provides a wider input common-mode range and improved symmetrical slew rate behavior compared to the classic 5T design, making it useful for high speed applications driving larger capacitive loads.

## Telescopic Cascode Topologies: High Gain with Headroom Trade-offs

To significantly increase open-loop voltage gain within a single stage, designers employ the Telescopic Cascode OTA architecture. This design stacks cascode shielding transistors vertically above the input differential pair to boost the output impedance.

![Telescopic OTA](https://images.kvlp.in/silicon/telescopic_OTA.png)

You can see how the transistors are stacked to form this _telescope_.
Time to break it down layer by layer from the very top:

**The Tail Current Source:** At the base sits **M<sub>9</sub>**, driven by a fixed bias voltage ($V_{b4}$), providing the constant tail current for the circuit.

**The Input Pair Layer:** Just above the tail are **M<sub>1</sub>** and **M<sub>2</sub>**. These are the core transconductance engines that convert your differential input voltages ($V_{in}$ and $V_{in2}$) into current.

**The NMOS Cascode Layer:** This is where the magic happens. **M<sub>3</sub>** and **M<sub>4</sub>** are stacked directly on top of the input pair, with their gates tied to a constant bias voltage ($V_b$). They act as impedance shields, multiplying the resistance looking down into the input pair.

**The PMOS Cascode Active Load:** At the top, connected to the $V_{DD}$ rail, is a four-transistor PMOS current mirror block. **M<sub>7</sub>** and **M<sub>8</sub>** act as the primary mirror, while **M<sub>5</sub>** and **M<sub>6</sub>** form a secondary PMOS cascode layer. The left side (M<sub>7</sub> and M<sub>5</sub>) is diode-connected to set the bias for the right side, ensuring high output impedance looking up from the output node. 

**The Gain Advantage**

Because of this vertical stacking, the total output impedance at the single-ended output node ($V_{out}$) is multiplied by the intrinsic gain of the cascode devices. The resistance looking into the output is the parallel combination of the effective impedance looking down into M<sub>4</sub> and up into M<sub>6</sub>:

$$R_{out} \approx (g_{m4} r_{o4} r_{o2}) \parallel (g_{m6} r_{o6} r_{o8})$$

By multiplying the output impedance so drastically, the Telescopic Cascode enables extremely high voltage gain in a single stage while preserving excellent high-frequency performance (since there are no internal low-frequency poles).

**The Headroom Trade-Off**

The primary drawback of this architecture is obvious just by looking at it: it is incredibly tall. You have five transistors stacked vertically between $V_{DD}$ and Ground (**M<sub>8</sub>, M<sub>6</sub>, M<sub>4</sub>, M<sub>2</sub>, M<sub>9</sub>**). Keeping all five of these devices safely in their saturation regions consumes a massive amount of voltage headroom. This severely restricts your allowable output voltage swing, making the telescopic architecture challenging to implement on modern, low-voltage power supply rails.

## Architectural Comparison Matrix & Performance Taxonomy

To help designers select the optimal topology for a given application, we evaluate the four major advanced OTA architectures across five key performance criteria: Open-Loop Voltage Gain, High-Frequency Speed/Bandwidth, Output Voltage Swing Range, Power Efficiency, and Silicon Layout Area Footprint.

| **OTA Structural Topology** | **Open Loop Voltage Gain (A<sub>v</sub>​)** | **High Frequency Speed / Bandwidth**        | **Output Voltage Swing Range**                  | **Power Efficiency Optimization**           | **Silicon Layout Area Footprint**             |
| --------------------------- | ------------------------------------------- | ------------------------------------------- | ----------------------------------------------- | ------------------------------------------- | --------------------------------------------- |
| **Simple Symmetrical**      | Low to Moderate (25 dB – 35 dB)             | Outstanding (High pole frequencies)         | Moderate to Symmetrical Range                   | Moderate (Current split into multiple legs) | Compact (Minimal device count, easy matching) |
| **Telescopic Cascode**      | Highly Elevated (50 dB – 65 dB)             | Maximum Limit (Single-stage pole response)  | Severely Restricted (Limited by vertical stack) | Outstanding (Minimal pathways required)     | Moderate (Requires extra bias networks)       |
| **Folded Cascode**          | Highly Elevated (45 dB – 60 dB)             | Excellent (High dominant pole)              | Wide Range (Decoupled input limits)             | Moderate (Demands extra branch current)     | Large (Higher device count, complex routing)  |
| **Miller Two Stage**        | Maximum Potential (70 dB – 90 dB)           | Restricted (Limited by Miller compensation) | Maximum Limit (Rail-to-rail output swing)       | Low to Moderate (Two active stages)         | Extremely Large (Requires capacitor space)    |

This comparison taxonomy highlights the fundamental trade-offs inherent in analog IC design. For instance, the Telescopic Cascode architecture offers exceptional speed and low power consumption but suffers from a restricted output swing. In contrast, the Miller Two-Stage architecture delivers high voltage gain and wide output swings but requires a larger silicon area and consumes more power due to its internal compensation networks. Selecting the right architecture depends on the specific constraints and performance priorities of the target application.

## Monolithic Silicon Layout Architectures & Symmetrical Matching

Transitioning an OTA design from a schematic diagram to a physical silicon layout requires careful attention to matching and parasitic suppression. In nanoscale fabrication processes, minor layout asymmetries can lead to matching errors, causing input offset voltages, degraded common-mode rejection, and increased harmonic distortion. Designers employ structured layout techniques to mitigate these effects. 

For the input differential pair (M<sub>1</sub> and M<sub>2</sub>), maintaining thermal and geometric symmetry is critical. Designers split each input transistor into multiple parallel sub elements and arrange them in a Common Centroid Layout Configuration. This interlocking pattern ensures that linear gradients in temperature or oxide thickness across the die affect both composite devices equally, minimizing input offset errors.

```
| Dummy 1 | M1_A | M2_B | Dummy 2 |
| ------- | ---- | ---- | ------- |
| Dummy 3 | M2_A | M1_B | Dummy 4 |
```

Additionally, Dummy Transistors are placed at the outer edges of the layout matrix to protect the active devices from etching non-uniformities during manufacturing. Guard rings connected to the supply rails surround the entire amplifier block to isolate it from substrate noise generated by adjacent digital circuits. Routing channels for high-frequency signal lines are carefully managed using shield tracks to minimize parasitic cross-coupling and preserve the amplifier's frequency response.

## The gm/ID Optimization Framework

As fabrication technologies scale down into deep sub-micron and nanoscale regions, classical square law model equations lose accuracy due to short-channel effects like velocity saturation. To address this, modern analog IC designers utilize the systematic gm/ID Optimization Framework, which centers design calculations around the transconductance efficiency metric rather than simplified analytical approximations. 

The gm/ID ratio represents the transconductance generated per unit of bias current, serving as a direct indicator of power efficiency. This metric is plotted against the normalized current density (I<sub>D</sub> / (W/L)) using data extracted directly from the foundry's Process Design Kit (PDK) simulations. This approach allows designers to navigate smoothly between the weak, moderate, and strong inversion regions without relying on piecewise analytical models.

## Technical Summary & Concluding Reference Specifications

This comprehensive blog outlines the essential role of the Operational Transconductance Amplifier (OTA) in modern monolithic analog integrated circuit design. By omitting low impedance voltage buffers and delivering a direct current-mode output, the OTA provides an efficient, high-speed solution for driving capacitive loads in deep sub-micron CMOS silicon processes.

Optimizing an OTA design requires a careful balancing of competing performance parameters, managing the trade-offs between transconductance efficiency, output impedance, noise, and frequency response.

## Bonus!!

To develop and test your own OTA you can use [HeiChip 2026's Analog workshop](https://github.com/HeiChips/heichips26-analog-workshop)
