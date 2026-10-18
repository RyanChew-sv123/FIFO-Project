# Synchronous FIFO with a Self-Checking SystemVerilog Testbench

A small, parameterized synchronous FIFO written in SystemVerilog, with a self-checking testbench that runs three scenarios and prints PASS or FAIL for each check.

I built this to learn SystemVerilog and basic verification ideas in a few days, coming from a non-engineering background. It is a learning project, not production code.

## Files

| File | What it is |
|---|---|
| `design.sv` | The FIFO (the design under test) |
| `testbench.sv` | The self-checking testbench |

## Design

The FIFO is 8 entries deep and 16 bits wide by default. Both are parameters, so they can be changed without editing the logic.

| Parameter | Default | Meaning |
|---|---|---|
| `DEPTH` | 8 | Number of entries. Should be a power of 2. |
| `DATA_WIDTH` | 16 | Bits per entry |
| `ADDR_SIZE` | `$clog2(DEPTH)` | Bits needed to select an entry |
| `PTR_SIZE` | `ADDR_SIZE + 1` | Pointer width: address bits plus one lap bit |

### How it works

- **Pointers with a lap bit.** The read and write pointers are one bit wider than the address. The lower bits select an entry, and the top bit flips each time the pointer wraps around.
- **`empty`** is high when the two pointers are fully equal.
- **`full`** is high when the address bits match but the lap bits differ, meaning the write pointer is exactly one lap ahead.
- **`full` and `empty` are calculated from the pointers, not stored in registers**, so they can never fall out of step with the pointers.
- **Registered output.** `data_out` only changes when a valid read happens, one clock edge after `read_en` is set. It holds its last value until the next read, and is not cleared by reset. `empty` tells you whether there is anything valid to read.
- Writes are ignored when the FIFO is full, and reads are ignored when it is empty.

## Testbench

The testbench changes its inputs on the **falling** clock edge, and the FIFO samples them on the **rising** edge. This keeps inputs stable half a cycle before they are sampled and avoids race conditions.

A `check` task compares each output with the expected value and prints PASS or FAIL. A counter tracks failures, and the run ends with `TEST PASSED` or `TEST FAILED`.

| Scenario | What it does | What it checks |
|---|---|---|
| 1 | Write 1, 10, 100, then read all three | Items come out in the order they went in, `data_out` stays unknown until the first read, and `empty` returns to 1 |
| 2 | Write 1, read, write 2, read, write 3, read | Alternating writes and reads work, `empty` toggles correctly, and `data_out` holds its value through reset |
| 3 | Fill all 8 entries with powers of 2 (1 to 128), attempt a 9th write (256), then read everything back | `full` goes high after the 8th write, the 9th write is ignored, and the 8 original values come back in order |

Each scenario starts with a reset so earlier scenarios cannot affect later ones.

## How to run

1. Open [EDA Playground](https://www.edaplayground.com) and log in.
2. Set the language to **SystemVerilog/Verilog**.
3. Choose a simulator (I used **[Icarus Verilog 12.0]**).
4. Paste `testbench.sv` into the left pane and `design.sv` into the right pane.
5. Tick **Open EPWave after run**, then click **Run**.

Expected log: three scenario headers with `PASS:` lines, ending in `TEST PASSED`.

## Results

All 33 checks pass across the three scenarios.

<!-- Add your own waveform screenshot here, for example: -->
<!-- ![Waveform](waveform.png) -->

## What I learned

- Why the pointers need an extra lap bit: without it, the pointers being equal could mean either empty or full.
- Why `full` and `empty` are better calculated from the pointers than stored.
- The difference between registered logic (`always_ff`, changes only on a clock edge) and combinational logic (`assign`, follows its inputs immediately).
- Why a testbench should drive inputs away from the clock edge the design samples on.
- The difference between a first-word fall-through output and a registered output, and why the testbench checks have to move one clock later for the registered style.
- Why non-power-of-2 depths do not work with this pointer scheme.

<!-- Replace or add to this list with things that actually surprised you or went wrong. -->

## Limitations and next steps

Not tested yet:

- Reading and writing in the same clock cycle
- Reading from an empty FIFO and writing to a full FIFO, checking that nothing changes
- Wrap-around while the FIFO is only partly full
- Random data and random timing

Next, I plan to:

1. Add the missing directed tests above.
2. Add SystemVerilog assertions (for example, `full` and `empty` never high together).
3. Add functional coverage.
4. Rebuild the testbench in UVM.

## How this was built

I had no SystemVerilog or UVM background. I used an AI assistant to help me learn the concepts and to draft the code, then ran it, studied each part, and tested and modified it myself.

## Author

[RYAN CHEW] · [https://www.linkedin.com/in/chee-hoo-c-bb68a9123/?isSelfProfile=true]

May refer to the EDAPlayground code here: https://edaplayground.com/x/VGYu
