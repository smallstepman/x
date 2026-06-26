# Observations and notes from linked materials and googling and LLMing

Optimal Vth is a moving target with complex dynamics (depends on temperature, charge leakage, read disturb, manufacturing variations, P/E cycles), and the drift is not a simple additive offset

----

- When FTL fetches raw bits from the NAND flash memory is has **direct visibility into RBER**: the ECC engine counts exactly how many bits in a given code word have been corrupted. The FTL requires RBER information during *every* read operation. If the RBER for a specific block exceeds a safe threshold (even though the data is still 100% correctable by ECC), the FTL immediately triggers protective procedures:

  - **Read Scrubbing**: It copies the data to a new, safe location before the errors become uncorrectable.
  - **Wear Leveling**: It marks the block as potentially worn out and updates the mapping tables.
  - "Read Retry" (our "20x computationally expensive algorithm"): A mechanism, often involving high-latency Soft-Decision LDPC decoding, to adaptively find the optimal threshold voltage (Vth) when data retention errors rise. A high Raw Bit Error Rate (RBER) exceeding standard ECC capabilities is the primary trigger for this algorithm. This algorithm is also often triggered when we cold boot (due to Cross-Temperature Effect, Volatile Table Loss, or FTL Metadata Boot Strapping).

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

# Finding optimal \(V_{th}\), \(V_{min}\), \(V_{max}\) inputs for bisect algorithm

The naive approach would be to use last known good values. 

However, we can improve on that by using a more sophisticated approach:

During the development of a new 3D NAND flash drive, trough testing and benchmarking, we can train an Artificial Neural Networks (ANNs) to model how the physical architecture behaves based on inputs such as Program/Erase cycles (wear level), retention time (how long the data has sat unpowered), temperature variations, physical location (the exact word-line layer or column position in the 3D stack, as tapering angles change cell sizes). The neural network maps out an incredibly complex, multi-dimensional prediction of exactly how the \(V_{th}\) distributions will shift under those combined stress factors.

Instead of deploying the raw, resource-heavy neural network into the SSD's micro-controller, engineers distill the neural network’s findings into lightweight mathematical formulas or polynomial regression models.The FTL firmware uses these ML-generated coefficients to proactively adjust the initial search bounds and starting midpoints.

Additionally,

> [!NOTE]
> and I'd like to point out that I came up with this idea without reading it first ^^ (proof init commit LoC 88)[https://github.com/smallstepman/x/commit/03a61e760c42d4e180fe3ebc6a739a63dfa565d8#diff-b335630551682c19a781afebcf4d07bf978fb1f8ac04c6bf87428ed5106870f5R88]

When the SSD is idle (not actively reading or writing for the host), the FTL controller does have the computational headroom to run more advanced ML algorithms. Many modern enterprise and high-end consumer SSDs execute Active Learning or Reinforcement Learning loops in the background. The FTL will intentionally sample degrading blocks during idle time, track how the \(V_{th}\) behaves, and adjust its internal prediction tables dynamically. This ensures that the drive learns your specific usage patterns (e.g., if you run a very hot server environment or a cold desktop boot environment) and updates its voltage boundaries before you ever request a file.




  - wionalle can precompute for manufacturing variations 
- we can gather information about previous Vth value that resulted in correct read operation (with timestamp and it's PBA to later be able to determine which neighboring cells could've been affected by the Read Disturb), and other factors like temperature
- looks like reading BER > 1.1% could be a good candidate for invoking our expensive algorithm

# Problem re-statment 

At the core, the question is: 
  A) what sequence of read ops at different Vth values maximizes the probability of success before falling back to the expensive algorithm, and 
  B) if read ops fails and our predefined sequence allows for next read try, in which direction we tune Vth and how big of a step we take. 

# Finding optimal sequence / number of read attempts

To answer `A)`, we can begin by asking a question: is there a statistically-naive way of finding a **static** number of tries we can attempt to get the best performance, and while we can come up with a formula, part of that formula will depend on read error rate for previous successful data fetching operations, and that error rate isn't constant (because the drive might age, or environmental factors may change(temp)), therefore the **static** value doesn't exist, and we need to come up with a formula for finding optimal number of attempts based on collected information so far.



# 

- the expensive algorithm can run on a separate thread when the device is idle (assuming it doesnt degrade the disk), and cache the correct Vth for the block
