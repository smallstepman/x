# Neanderthal's Glossary

> [!NOTE]
> The contents of this section represent my (shamefully coarse and primitive) understanding of the problem space I was able to obtain while learning about how to approach the task. I'm not removing it, because that's the only way for you 🫵 to backtrack where my understanding slipped which might've caused incorrect approach or wording in attempt to come up with the solution to the presented task. I'm not attaching it for a review, simply because the problem space is massive, and I could dedicate only several hours to it, however if you wish and have time to correct me where LLM couldn't, I'd be grateful.


- RBER (Raw Bit Error Rate) — measures the number of bit errors present in the raw data read from NAND flash before any ECC correction is applied.
- UBER (Uncorrectable Bit Error Rate) — measures the rate of bit errors that remain uncorrectable after the SSD’s ECC has attempted to correct them.
- LDPC codes = Low-Density Parity Check - highly efficient error-correcting codes used to transmit data reliably over noisy channels, performing very close to the theoretical maximum limit (the Shannon limit). For the LDPC code in the Konstanz paper, 91% bits are data, 9% parity. Raw bit error rate ≤ 1.1% = decoder succeeds (residual error probability ~ 1 in a million). Raw bit error rate ≥ 1.9% = decoder fails entirely. In between, the chance of successful decode still exists but it decreases at increasingly higher rate (hence the name: ECC Cliff). 


----

NAND Die Layout:
- the die is split into Planes (Each Plane can execute read/write/erase operations independently because it has its own dedicated page buffers and row decoders)
  - P-well (Studnia typu P): Fizyczny basen w krzemie odizolowany barierami wysokonapięciowymi, w którym znajduje się podłoże dla tranzystorów. Each p-well has multiple blocks. p-well connects to ground (0V), but only during specific operations (triggered by a dynamic voltage switch)
    - Block = All wordlines combined across all bitlanes. All blocks connect to same ground provided in P-well. 
    - Page = All cells connected to a single selected Wordline. Total count of pages in a block is #Wordlines x #bits-per-cell
      - Wordline = Rows. Data is read along the wordline (row-by-row), not down the bitline string 
      - Bitline = Columns. Bitlane is a global copper wire that runs over Bitline String (or a "NAND String Assembly" == SGD + NAND String(linear chain of memory transistors physically wired together in series (Drain of Transistor 1 is fused directly into the Source of Transistor 2)) + SGS ((SGD / SGS) = dedicated isolation transistors located at the absolute top and absolute bottom of every vertical NAND transistor string ))
        - NAND memory cells are located at the intersections where bitlines and wordlines cross
          - MSB/CSB/LSB = Most/Center/Least Significant Bit - on the silicon level, TLC cell splits its charge capacity into 8 distinct physical states (bins of electrons). To fully decipher all 3 indexed bits you have to apply up to 7 different voltage levels (Vr1-Vr7).

----

Voltages:
When you read a single Page, the row decoders light up the block with this exact snapshot of voltages:
  - Selected Row: Vth loops through Vr1-Vr7 (e.g., 1.2V, 1.8V, etc.) 
  - All Other Rows: locked at Vpass (e.g., 5.5V)
  - Top/Bottom of String: open at Vsgd/Vsgs (aka Vcc) (e.g., 3.3V)
  - Columns: pre-charged to Vbl (e.g., 0.7V)
  - Silicon Base: sunk to Vgnd (0V)

Sequence (tldr: warm up the bitlines, ping the wordlines, read from matrix and convert to digital):
1. Bitline Pre-charge Phase (setup phase)
    - The Sense Amplifiers pump the electrical capacitance of all Bitlines up to a steady, low positive voltage Vbl, usually around 0.5V to 0.7V). 
    - Simultaneously, the string boundary gates Vsgd and Vsgs and all unselected rows Vpass are ramped up to fully open the rest of the vertical transistor chain (it is an "data OFF == elecricity ON" switch - by applying Vpass, you are forcing that unselected cell to stay wide open, turning it into a temporary copper wire so electricity can pass right through it)
2. Wordline Development Phase (or Sensing Phase) (It is called the "development" phase because the voltages on the Bitline columns are actively diversifying (developing a delta) depending on the stored data state.)
    - The row decoders drop the exact target reference voltage Vr1-7 (aka Vth) directly onto the selected Wordline. This triggers the physical current reaction of the target transistor: 
      - If its Vth is low == 1 (the transistor opens, creating a path to the 0V ground substrate. The pre-charged voltage on that specific Bitline drains away (Discharge))
      - If its Vth is high == 0 (the transistor remains closed. The pre-charged voltage stays trapped on the Bitline (No Discharge))
3. Latching Phase (or Data Evaluation Phase) - this is the final step
    - once the voltage drop on the discharging Bitlines hits a measurable threshold, the Sense Amplifiers isolate themselves from the cell matrix. Internal CMOS differential† latches compare the final Bitline voltages against an internal reference voltage to see which columns held their charge and which ones drained. The analog state is instantly converted into digital 0s and 1s and locked into local SRAM cache latches, ready to be sent to the SSD controller over the digital bus.

----

histogram - often called a "Vth distribution curve", is a statistical graph that maps out exactly how the threshold voltages Vth of millions of physical transistors across a block are distributed along the voltage spectrum. On the silicon and firmware level, this histogram is the most critical diagnostic tool used by the SSD controller to fight data corruption, electrical leakage, and silicon aging

```
```markdown
Number of 
Cells
  ▲       L0       L1       L2       L3       L4       L5       L6       L7
  │      ┌─┐      ┌─┐      ┌─┐      ┌─┐      ┌─┐      ┌─┐      ┌─┐      ┌─┐
  │     /   \    /   \    /   \    /   \    /   \    /   \    /   \    /   \
  │____/     \__/     \__/     \__/     \__/     \__/     \__/     \__/     \____
  ┼─────────────────────────────────────────────────────────────────────────────► Voltage
  │            ▲        ▲        ▲        ▲        ▲        ▲        ▲          (Vth)
             V_R1     V_R2     V_R3     V_R4     V_R5     V_R6     V_R7
            (1.0V)   (1.8V)   (2.6V)   (3.4V)   (4.2V)   (5.0V)   (5.8V)
```


for data reading, this chart represents 8 possible states, but a programmed cell only "responds" to voltage that targets its peak, and others peaks will remain "silent"
