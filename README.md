# Observations and notes from linked materials and googling and LLMing

Optimal Vth is a moving target with complex dynamics (depends on temperature, charge leakage, read disturb, manufacturing variations, P/E cycles), and the drift is not a simple additive offset

----

- When FTL fetches raw bits from the NAND flash memory is has **direct visibility into RBER**: the ECC engine counts exactly how many bits in a given code word have been corrupted. The FTL requires RBER information during *every* read operation. If the RBER for a specific block exceeds a safe threshold (even though the data is still 100% correctable by ECC), the FTL immediately triggers protective procedures:

  - **Read Scrubbing**: It copies the data to a new, safe location before the errors become uncorrectable.
  - **Wear Leveling**: It marks the block as potentially worn out and updates the mapping tables.
  - **Read Retry** (our "20x computationally expensive algorithm"): A mechanism, often involving high-latency Soft-Decision LDPC decoding, to adaptively find the optimal threshold voltage (Vth) when data retention errors rise. A high Raw Bit Error Rate (RBER) exceeding standard ECC capabilities is the primary trigger for this algorithm. This algorithm is also often triggered when we cold boot (due to Cross-Temperature Effect, Volatile Table Loss, or FTL Metadata Boot Strapping).

But for everyday read operations, these are last resort measures.

Before the firmware triggers the heavy data-rescue modes, it uses a fast, hardware-assisted mathematical method called \(V_{th}\) Tracking or Binary Bisection. When a default read returns an elevated RBER, the controller leverages a hardware feature inside the ECC engine: **Bit Flip Asymmetry**. The ECC engine (like LDPC) does not just count errors, but also detects the direction of the errors:
  - \(1 \rightarrow 0\) flips: Means the cell voltage dropped (the distribution shifted left).
  - \(0 \rightarrow 1\) flips: Means the cell voltage increased (the distribution shifted right).

Using this directional feedback, the firmware runs a mathematical binary bisection.

```mermaid
graph TD
    Start([Trigger Vth Tracking]) --> Step1[<b>Initialize Search Window</b><br>Set initial bounds:<br>V_min = 1.0V, V_max = 1.4V]
    
    %% Loop Entry Point
    Step1 --> CalcMid[<b>Calculate Next Midpoint</b><br>V_test = V_min + V_max / 2]
    CalcMid --> ReadPage[<b>Read Page at V_test</b><br>Hardware applies V_test voltage<br>ECC evaluates Bit Flip Asymmetry]
    ReadPage --> DecGate{<b>Evaluate Asymmetry</b>}
    
    %% Branching Logic
    DecGate -- "Asymmetry ≈ 0<br>(Target Found)" --> Success[<b>Target Located</b><br>Save optimal V_test to SRAM<br>Return data to Host]
    DecGate -- "Excess of 1 ➔ 0 flips<br>(Voltage is too high)" --> AdjustLower[<b>Narrow Window Downward</b><br>Set V_max = V_test]
    DecGate -- "Excess of 0 ➔ 1 flips<br>(Voltage is too low)" --> AdjustHigher[<b>Narrow Window Upward</b><br>Set V_min = V_test]
    
    %% The Recursive Loop Back
    AdjustLower --> CheckResolution{<b>Is Window Size > Min Step?</b>}
    AdjustHigher --> CheckResolution
    
    CheckResolution -- Yes ➔ Next Iteration --> CalcMid
    CheckResolution -- No ➔ Bisection Failed --> Fail[<b>Drop to Soft LDPC Read Retry</b>]
```

# Finding optimal `Vth`, `Vmin`, `Vmax` inputs for bisect algorithm

The naive approach would be to use last known good values. However, we can improve on that by using a more sophisticated approach:

During the development of a new 3D NAND flash drive, trough testing and benchmarking, we can train an Artificial Neural Networks (ANNs) to model how the physical architecture behaves based on inputs such as Program/Erase cycles (wear level), retention time (how long the data has sat unpowered), temperature variations, physical location (the exact word-line layer or column position in the 3D stack, as tapering angles change cell sizes). The neural network maps out an incredibly complex, multi-dimensional prediction of exactly how the \(V_{th}\) distributions will shift under those combined stress factors.

Instead of deploying the raw, resource-heavy neural network into the SSD's micro-controller, engineers distill the neural network’s findings into lightweight mathematical formulas or polynomial regression models.The FTL firmware uses these ML-generated coefficients to proactively adjust the initial search bounds and starting midpoints.

Additionally,

> [!NOTE]
> and I'd like to shamelessly point out that I came up with this idea without reading it first ^^ [proof: init commit README.md LoC:88](https://github.com/smallstepman/x/commit/03a61e760c42d4e180fe3ebc6a739a63dfa565d8#diff-b335630551682c19a781afebcf4d07bf978fb1f8ac04c6bf87428ed5106870f5R88)

When the SSD is idle (not actively reading or writing for the host), the FTL controller does have the computational headroom to run more advanced ML algorithms. Many modern enterprise and high-end consumer SSDs execute Active Learning or Reinforcement Learning loops in the background. The FTL will intentionally sample degrading blocks during idle time, track how the \(V_{th}\) behaves, and adjust its internal prediction tables dynamically. This ensures that the drive learns your specific usage patterns (e.g., if you run a very hot server environment or a cold desktop boot environment) and updates its voltage boundaries before you ever request a file.

# Data acquisition

## 1. The Development Phase (Silicon Lab Characterization)

We can use automated, highly specialized NAND Characterization Testers connected to raw NAND wafers and packaged chips. These chips are placed inside climate-controlled thermal chambers and subject the NAND to extreme conditions: writing and erasing blocks thousands of times (P/E cycle acceleration), baking the chips at high temperatures to simulate months of data leakage in hours (Accelerated Retention Loss), and reading the same cells billions of times (Read Disturb simulation). The tester sweeps voltages in incredibly tiny increments (e.g., 1mV steps) across the entire voltage range to map out the exact shape of the $V_{th}$ distribution curves. This creates petabytes of raw data showing exactly how cell distributions deform under every permutation of age, temperature, and wear. This massive dataset is used to train the Deep Neural Networks on the manufacturer’s server farms.

## 2. The Manufacturing Phase (Accounting for Variations)

No two silicon wafers are identical. Due to microscopic variations in chemical vapor deposition, etching depth, and lithography, NAND cells in the center of a wafer might behave differently than cells on the outer edge.

During the IC Sorting and Outgoing Quality Control (OQC) steps at the factory, every single manufactured NAND die undergoes a series of hardware tests. The tester measures the baseline resistance, cell capacitance, and native, out-of-the-box RBER of different sectors of the die. In 3D NAND, holes are etched through hundreds of layers. The geometry of these holes tapers slightly at the bottom. The manufacturing tests acquire data on how much the top layers differ from the bottom layers in performance.

Instead of running a heavy neural network here, the factory uses the acquired variation data to calculate individual Calibration Matrices (Fuses). These unique calibration coefficients are permanently burned into a special, non-volatile zone of that specific NAND die (often called "OTP" or One-Time Programmable memory). When the SSD controller boots up, its firmware reads these fuses to customize its baseline $V_{th}$ tables for the exact silicon variations of that specific drive.

## 3. The Daily Operations Phase (Real-Time In-Drive FTL Telemetry)

Once the SSD is running in a live system, the FTL acquires data dynamically to maintain an up-to-date map of the drive’s health. Data is captured by hardware performance counters baked directly into the SSD's ASIC and the ECC engine.

- For every single block or zone, the FTL updates registers tracking the exact number of P/E cycles, the time elapsed since the last write (timestamp tracking), and the number of times a block has been read since its last erase (Read Disturb Counter).
- The drive's internal On-Die Temperature Sensors continuously report real-time operating temperatures.
- Every time the Level 2 "Bisection Algorithm" runs, the FTL logs the final voltage offset that was required to fix the data. It also records the precise ratio of 1 → 0 vs 0 → 1 bit flips.
- The newly discovered $V_{th}$ offset from a successful bisection is stored in the drive's volatile SRAM/DRAM lookup tables for use on subsequent reads. 
- When the Host OS goes idle, the FTL's reviews accumulated data and runs the lightweight regression formulas against the real-time data (Temp + Age + P/E + Bit Flip history).


# Comments 

- see `glossary.md` for the raw braindump
- it took me a while to understand the problem space well enough to be able to start asking the right questions to the LLM
- intuitively I knew there must be a bisect algorithm involved, but for a few hours I couldn't figure out where was the directionality information coming from, and LLMs were overloading me with information I didn't understand
- clearly, no denying, lots of the text here was LLM generated, but nothing is an effect of "one-shotting", the knowledge and understanding was built over several agent and chat sessions by asking probably well over a hundred questions. No part of both of the included documents were touched in any shape or capacity by agent harness, any generated text was manually copy-pasted from the chat UI (and of course well understood and formatted (e.g. to remove all of the "rapid dramatic endurence test"-kind of wording)). I decided this is better course of action as I will learn more as I go, as opposed to forcing my own wording, and have you watching me stumble by e.g. calling ASIC the "ARM CPU" (as I would)
