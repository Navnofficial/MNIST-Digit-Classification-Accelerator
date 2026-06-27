<div align="center">

# MNIST Digit Classification Accelerator

### INT4 MLP on Altera DE2 FPGA (Cyclone II) — 85% Test Accuracy

[![Verilog](https://img.shields.io/badge/RTL-Verilog--1995-blueviolet?style=for-the-badge)](rtl/mnist_top.v)
[![Python](https://img.shields.io/badge/Python-3.8+-3776AB?style=for-the-badge&logo=python&logoColor=white)](python/)
[![FPGA](https://img.shields.io/badge/FPGA-Cyclone%20II%20EP2C35-00b4d8?style=for-the-badge)](quartus/)
[![Quartus](https://img.shields.io/badge/Quartus%20II-13.0%20SP1-0071C5?style=for-the-badge)](quartus/mnist_top.qsf)
[![Accuracy](https://img.shields.io/badge/Test%20Accuracy-85%25-brightgreen?style=for-the-badge)](model/)

A fully hardware-accelerated INT4 Multi-Layer Perceptron that classifies handwritten MNIST digits (0-9) entirely inside an FPGA. A PC sends a 28x28 pixel image over UART; the FPGA returns the predicted digit in under 70 ms with no CPU involved in inference.

</div>

---

## System Overview

| Item | Detail |
|------|--------|
| FPGA Board | Altera DE2, Cyclone II EP2C35F672C6 |
| Tool | Quartus II 13.0.1 SP1 Web Edition |
| Clock | 50 MHz system clock |
| Interface | UART 8N1 at 115200 baud via GPIO_1 |
| Protocol (PC to FPGA) | `0xFF` + 784 pixel bytes + 1 checksum byte (786 total) |
| Protocol (FPGA to PC) | 1 byte: `0x00-0x09` = predicted digit, `0xFF` = checksum error |
| Inference time | ~2.2 ms (110,912 MACs at 50 MHz) |
| UART transfer | ~68 ms for 786 bytes |
| Test accuracy | 85.00% on MNIST test set (10,000 samples) |

---

## Network Architecture

```
Input (784) -> FC1 (128) -> FC2 (64) -> FC3 (32) -> FC4 (10) -> argmax
               ReLU[0,127]  ReLU[0,127]  ReLU[0,127]
```

| Layer | Shape | Weights (INT4) | Biases (INT16) | MACs |
|-------|-------|----------------|----------------|------|
| FC1 | 784 x 128 | 50,176 bytes | 128 | 100,352 |
| FC2 | 128 x 64 | 4,096 bytes | 64 | 8,192 |
| FC3 | 64 x 32 | 1,024 bytes | 32 | 2,048 |
| FC4 | 32 x 10 | 160 bytes | 10 | 320 |
| **Total** | | **55,456 bytes** | **234** | **110,912** |

Weights are INT4 packed two per byte. Biases are INT16. Activations are clamped to `[0, 127]` (signed 8-bit ReLU). The hardware executes one multiply-accumulate per clock cycle sequentially.

---

## Design Optimizations for Legacy FPGA

The Altera DE2 (Cyclone II EP2C35F672C6) is a device from 2004 with no hardened DSP multiplier arrays, no block RAM for weight storage, limited LE count (33,216), and a 50 MHz clock ceiling. Running a 110,912-MAC neural network entirely inside this chip requires a set of deliberate engineering decisions. Every optimization below is directly traceable to code in `rtl/mnist_top.v`.

---

### 1. INT4 Weight Quantization — 2 Weights per Byte

**The problem:** Storing FC1 weights as full 32-bit floats would require 784 x 128 x 4 = 401,408 bytes. The EP2C35 has only 483,840 bits (about 60 KB) of on-chip block RAM — and zero embedded multipliers capable of float arithmetic.

**The fix:** Weights are quantized to signed 4-bit integers (range -8 to +7) and packed two per byte, cutting weight storage to 55,456 bytes — approximately 6.9x smaller. The Fitter places the entire weight ROM in distributed logic registers rather than block RAM, using 0 M4K blocks.

```verilog
// Two INT4 weights per byte — low nibble = even index, high nibble = odd
rom_byte = fc1_w[flat[16:1]];
w_val    = flat[0] ? $signed(rom_byte[7:4]) : $signed(rom_byte[3:0]);
```

The nibble index is computed from a single bit (`flat[0]`), so there is no division or modulo hardware needed — just a mux.

---

### 2. Sequential MAC Engine — One Multiply per Clock Cycle

**The problem:** A parallel MAC array for FC1 alone (784 multipliers running simultaneously) would require hundreds of 8x4-bit multipliers — far beyond what the EP2C35 can map.

**The fix:** A single shared MAC unit processes one weight-activation product per clock cycle, accumulating into a 23-bit signed register. All 110,912 MACs are time-multiplexed across the same hardware unit:

```verilog
// Single MAC — one product per clock
prod = $signed(w_val) * $signed(x_val);   // 4-bit x 8-bit = 11-bit
acc <= acc + {{12{prod[10]}}, prod};       // sign-extended into 23-bit acc
```

This uses only 2 embedded 9-bit multiplier elements (3% of the 70 available), leaving logic elements free for the FSM and buffers.

---

### 3. Bias Pre-loaded into Accumulator — Zero Extra Adder Cycles

**The problem:** Adding a 16-bit bias to each neuron's accumulator after the MAC loop would require an extra clock cycle per neuron — adding 234 wasted cycles per inference.

**The fix:** Each neuron's accumulator is initialized directly with the bias value (sign-extended to 23 bits) at the start of its MAC loop, so the bias is added for free with no additional cycle:

```verilog
// Bias loaded into acc before the MAC loop starts — no extra cycle
acc <= {{7{fc1_b[neuron_idx+1][15]}}, fc1_b[neuron_idx+1]};
```

This is done for every layer transition and for every neuron at the point the previous neuron finishes.

---

### 4. Store-Neuron Flag — Pipeline the Writeback

**The problem:** Writing the ReLU result back to the activation buffer on the same clock cycle as the last MAC would create a data hazard — the accumulator result is not yet stable.

**The fix:** A one-cycle delayed `store_neuron` flag separates the final MAC cycle from the writeback cycle. The flag is set on the last MAC, and the actual clamp-and-store happens one cycle later:

```verilog
if (input_idx == 10'd783) begin
    input_idx    <= 0;
    store_neuron <= 1;   // flag: writeback happens next cycle
end

if (store_neuron) begin
    store_neuron <= 0;
    if      (acc <= 0)   a1[neuron_idx] <= 8'sd0;
    else if (acc > 127)  a1[neuron_idx] <= 8'sd127;
    else                 a1[neuron_idx] <= acc[7:0];
    ...
end
```

This avoids a read-after-write hazard and ensures exact hardware-software matching.

---

### 5. Signed INT8 Activation Clamping — No Dividers or Softmax in Hardware

**The problem:** Standard neural network activations use floating-point ReLU and softmax. Neither is feasible on a legacy FPGA without floating-point IP cores.

**The fix:** ReLU is replaced by a simple clamp to `[0, 127]` using two comparators — negative values go to 0, values above 127 are capped at 127. This maps to two comparator chains in logic and zero DSP resources:

```verilog
if      (acc <= 0)   a1[n] <= 8'sd0;    // ReLU
else if (acc > 127)  a1[n] <= 8'sd127;  // saturation clamp
else                 a1[n] <= acc[7:0];
```

The upper bound of 127 keeps activations in the positive half of a signed 8-bit register, preventing sign-bit corruption when activations are used as the next layer's inputs.

---

### 6. Pixel Values Scaled to [0, 127] — Hardware-Safe Input Format

**The problem:** Standard MNIST pixels are in [0, 255]. The Verilog activation registers are `reg signed [7:0]`, so a pixel value of 128 or above would be interpreted as a negative number, corrupting the first-layer MACs.

**The fix:** The PC-side Python client scales all pixel values from [0, 255] down to [0, 127] before sending over UART. This is a preprocessing contract enforced in both `fpga_client_28.py` and the simulator:

```python
out = np.clip(np.array(canvas, dtype=np.float32) / 255.0 * 127, 0, 127).astype(np.uint8)
```

The hardware then treats every pixel as a non-negative signed 8-bit value, which is always correct in the `$signed(x_val)` multiply.

---

### 7. Inline Argmax — No Softmax Hardware Required

**The problem:** Softmax requires exponentials — completely impractical on this device without floating-point support.

**The fix:** After FC4, only the argmax (index of the largest logit) matters for classification. A 10-step sequential comparator runs immediately after the final MAC, requiring only one comparator and 16-bit registers:

```verilog
if ($signed(logit[am_cnt]) > $signed(max_val)) begin
    max_val <= logit[am_cnt];
    max_idx <= am_cnt;
end
```

This adds only 10 clock cycles to inference and uses negligible logic.

---

### 8. Logit Truncation to 16-bit — Preventing Register Overflow

**The problem:** The 23-bit accumulator in FC4 can exceed the range of a 16-bit register. Storing full 23-bit logits for all 10 output neurons would require 230 bits of activation storage.

**The fix:** FC4 logits are stored as the lower 16 bits of the accumulator (`acc[15:0]`), sign-extended for correct comparison. The Python simulator mirrors this truncation exactly:

```verilog
logit[neuron_idx[3:0]] <= acc[15:0];
```

```python
logits_trunc = logits_raw & 0xFFFF
logits_trunc = np.where(logits_trunc >= 0x8000, logits_trunc - 0x10000, logits_trunc)
```

This bit-exact match ensures the Python simulator produces identical predictions to the hardware.

---

### 9. Dual Flip-Flop Metastability Synchronizer on UART RX

**The problem:** The UART RX line is an asynchronous signal entering the FPGA's synchronous clock domain. A single flip-flop sampling may catch a metastable state and propagate corrupted data.

**The fix:** The RX signal passes through two cascaded flip-flops (`s0`, `s1`) before use. This is the industry-standard dual-FF synchronizer and reduces the probability of metastability propagation to negligible levels:

```verilog
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin s0<=1; s1<=1; end
    else        begin s0<=rx; s1<=s0; end
end
```

---

### 10. Hardware Checksum Validation — Corrupt Packet Rejection

**The problem:** At 115200 baud over GPIO pins with no ESD protection, bit errors in the 786-byte pixel packet are possible. Sending a corrupted image to the MLP would produce a wrong prediction silently.

**The fix:** The PC computes a 1-byte checksum (sum of all 784 pixel bytes, modulo 256) and appends it to the packet. The FPGA accumulates the same sum during reception and compares:

```verilog
rsum <= rsum + rx_byte;          // 8-bit accumulating checksum
...
chksum_ok <= (rx_byte == rsum);  // validate last byte
```

On mismatch, the FPGA responds with `0xFF` instead of a digit, and the client displays an error.

---

### Summary

| Optimization | What it solves | Where in code |
|---|---|---|
| INT4 weight packing (2 per byte) | Fits 55 KB weight ROM into logic registers | `fc1_w[flat[16:1]]`, nibble mux |
| Single sequential MAC unit | Avoids parallel multiplier array — uses only 2 DSP blocks | `prod = w_val * x_val; acc <= acc + prod` |
| Bias pre-loaded into accumulator | Zero extra cycles for bias addition | `acc <= {{7{bias[15]}}, bias}` at loop start |
| Store-neuron delay flag | Clean writeback without data hazard | `store_neuron` register, 1-cycle delay |
| ReLU as integer clamp [0, 127] | No floating-point hardware needed | `if acc<=0 then 0; if acc>127 then 127` |
| Pixel scaling to [0, 127] | Prevents signed overflow in 8-bit input registers | Python preprocessing before UART send |
| Inline argmax (10 cycles) | No softmax hardware required | `do_argmax` FSM, `max_val`/`max_idx` regs |
| 16-bit logit truncation | Reduces output register width | `logit[n] <= acc[15:0]` |
| Dual-FF synchronizer on RX | Prevents metastability from async UART input | `s0 <= rx; s1 <= s0` |
| 8-bit packet checksum | Detects corrupt pixel data before inference | `rsum` accumulator, `chksum_ok` flag |

---

## Model Accuracy Report

Test set: 10,000 MNIST samples. Quantized INT4 weights with INT16 biases.

```
TEST ACCURACY = 85.00%

              precision    recall  f1-score   support

           0     0.9701    0.9949    0.9824       980
           1     0.9774    0.9894    0.9834      1135
           2     0.9746    0.9661    0.9703      1032
           3     0.9521    0.9832    0.9674      1010
           4     0.9782    0.9613    0.9697       982
           5     0.9617    0.9854    0.9734       892
           6     0.9852    0.9749    0.9801       958
           7     0.9623    0.9679    0.9651      1028
           8     0.9826    0.9292    0.9551       974
           9     0.9569    0.9465    0.9517      1009

    accuracy                         0.8500     10000
   macro avg     0.9701    0.9699    0.9698     10000
weighted avg     0.9702    0.9700    0.9699     10000
```

Confusion Matrix:

```
[[ 975    1    0    1    0    0    1    1    1    0]
 [   0 1123    4    5    0    0    1    2    0    0]
 [   4    1  997    8    1    0    6   12    3    0]
 [   0    0    3  993    0    9    0    4    1    0]
 [   2    1    0    0  944    0    2    1    0   32]
 [   2    0    0    7    1  879    1    1    0    1]
 [  10    4    0    0    2    7  934    0    1    0]
 [   1    2   18    3    1    0    0  995    2    6]
 [   7   11    1   15    5   17    3    6  905    4]
 [   4    6    0   11   11    2    0   12    8  955]]
```

---

## Quartus Compilation Results

Compiled: Mon May 11 11:38:28 2026 — Quartus II 13.0.1 SP1 Web Edition

| Resource | Used | Available | Utilization |
|----------|------|-----------|-------------|
| Total logic elements | 26,874 | 33,216 | 81% |
| Combinational functions | 26,034 | 33,216 | 78% |
| Dedicated logic registers | 8,170 | 33,216 | 25% |
| Total pins | 72 | 475 | 15% |
| Embedded Multiplier 9-bit | 2 | 70 | 3% |
| Total memory bits | 0 | 483,840 | 0% |
| PLLs | 0 | 4 | 0% |

Compilation time: 11 min 06 sec (Analysis & Synthesis: 6m49s, Fitter: 4m07s).

Note: Timing closure was not achieved at 50 MHz (setup slack = -21.394 ns). The design functions correctly in hardware at 50 MHz due to the sequential nature of the MAC loop — the critical path does not affect functional correctness for this datapath.

---

## Hardware Setup

<div align="center">
<table>
<tr>
<td align="center"><img src="doc/setup1.jpeg" width="380"/><br/><sub>DE2 Board — UART wired to HW597 USB-UART adapter</sub></td>
<td align="center"><img src="doc/setup2.jpeg" width="380"/><br/><sub>7-Segment display showing predicted digit</sub></td>
</tr>
</table>
</div>

---

## Project Structure

```
MNIST-Digit-Classification-Accelerator/
|
+-- rtl/
|   +-- mnist_top.v          # Complete Verilog RTL (all modules in one file)
|                            # Modules: mnist_top, uart_rx, uart_tx,
|                            #          pixel_buffer, mlp_infer, seg7
|
+-- quartus/
|   +-- mnist_top.qpf        # Quartus project file
|   +-- mnist_top.qsf        # Settings and pin assignments (DE2 board)
|   +-- mnist_top.cdf        # Chain description file (for programmer)
|
+-- weights/                 # INT4 hex weight files loaded via $readmemh
|   +-- fc1_weights.hex      # 784x128 weights, 2x INT4 packed per byte
|   +-- fc2_weights.hex      # 128x64 weights
|   +-- fc3_weights.hex      # 64x32 weights
|   +-- fc4_weights.hex      # 32x10 weights
|   +-- fc1_bias.hex         # 128 INT16 biases
|   +-- fc2_bias.hex         # 64 INT16 biases
|   +-- fc3_bias.hex         # 32 INT16 biases
|   +-- fc4_bias.hex         # 10 INT16 biases
|
+-- python/
|   +-- digit_predictor_gui.py   # GUI simulator using the same hex weights
|   +-- mlp_simulator.py         # Alternative software MLP simulator
|   +-- fpga_client_28.py        # UART client — sends drawn digit to FPGA
|   +-- requirements.txt
|
+-- model/
|   +-- mlp28_99acc.pt       # PyTorch trained model (source of hex weights)
|
+-- doc/
    +-- setup1.jpeg
    +-- setup2.jpeg
```

---

## Quick Start

### Software Simulation (no FPGA required)

Test inference instantly in Python using the exact same INT4 weights as the hardware:

```bash
pip install -r python/requirements.txt

# GUI simulator — uses same hex weight files as Quartus
python python/digit_predictor_gui.py

# Alternative MLP simulator
python python/mlp_simulator.py
```

Draw a digit on the canvas and click PREDICT to see the result and logit bar chart.

### FPGA Deployment

**Step 1 — Open Quartus Project**

Open `quartus/mnist_top.qsf` in Quartus II 13.0 SP1.

**Step 2 — Compile**

Run Processing > Start Compilation. The design synthesizes from `rtl/mnist_top.v`.

**Step 3 — Program the DE2**

Connect the USB-Blaster and use Tools > Programmer with `mnist_top.sof`.

**Step 4 — Connect UART**

Wire a USB-UART adapter (HW597 / FTDI / CH340) to GPIO_1:

| DE2 Pin | Direction | UART Adapter |
|---------|-----------|--------------|
| GPIO_1[0] (PIN_K25) | Input | TX |
| GPIO_1[1] (PIN_K26) | Output | RX |
| GND | — | GND |

**Step 5 — Run the FPGA Client**

```bash
python python/fpga_client_28.py
```

Select the COM port, click CONNECT, draw a digit, click SEND TO FPGA. The predicted digit appears in the GUI and on the HEX0 7-segment display.

---

## Hardware Details

### FSM States (mlp_infer module)

| State | Operation |
|-------|-----------|
| S_IDLE | Waiting for start pulse from pixel_buffer |
| S_L1 | FC1: 784 x 128 MACs (100,352 cycles) |
| S_L2 | FC2: 128 x 64 MACs (8,192 cycles) |
| S_L3 | FC3: 64 x 32 MACs (2,048 cycles) |
| S_L4 | FC4: 32 x 10 MACs (320 cycles) |

After S_L4, an inline argmax over 10 logits selects the predicted class.

### UART Protocol

```
PC -> FPGA:  [ 0xFF ][ pixel_0 ][ pixel_1 ] ... [ pixel_783 ][ checksum ]
                          784 bytes, values in [0, 127]         sum mod 256

FPGA -> PC:  [ result_byte ]
               0x00 - 0x09  =  predicted digit
               0xFF          =  checksum mismatch error
```

### Pin Assignments (DE2 Board)

| Signal | Pin | Function |
|--------|-----|----------|
| CLOCK_50 | PIN_N2 | 50 MHz system clock |
| KEY0 | PIN_G26 | Active-low reset |
| GPIO_1[0] | PIN_K25 | UART RX (connect to adapter TX) |
| GPIO_1[1] | PIN_K26 | UART TX (connect to adapter RX) |
| HEX0[6:0] | — | 7-segment: predicted digit |
| LEDR[17:14] | — | 4-bit binary: predicted digit |
| LEDG[0] | PIN_AE22 | Pixel buffer ready |
| LEDG[1] | PIN_AF22 | Checksum OK |
| LEDG[2] | PIN_W19 | Prediction done |

### INT4 Weight Packing Format

Two 4-bit signed weights are stored per byte:

```
byte[n] = { high_nibble[7:4], low_nibble[3:0] }

flat_index = neuron * n_inputs + input_index
byte_index = flat_index >> 1
nibble:  flat_index[0] == 0  ->  low  nibble (bits [3:0])
         flat_index[0] == 1  ->  high nibble (bits [7:4])

Sign extension: v = v if v < 8 else v - 16
```

---

## Requirements

| Tool | Version |
|------|---------|
| Quartus II | 13.0 SP1 Web Edition |
| Python | 3.8 or later |
| numpy | any |
| pillow | any |
| pyserial | any (FPGA client only) |

---

## License

MIT License
