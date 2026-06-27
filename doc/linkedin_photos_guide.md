# LinkedIn Post — Photos Checklist

Collect these photos/screenshots in this folder before posting.
Post them in the order listed below for best impact.

---

## Slide 1 — Hero Shot (MOST IMPORTANT — stops the scroll)

**What:** DE2 board powered on, 7-segment display showing a predicted digit (e.g. 3, 7, 5).
LEDs lit up on the board. UART adapter plugged in.

**How to take it:** Phone camera, close-up on the board. Make sure the digit on HEX0 is clearly visible.

**Save as:** `doc/slide1_fpga_hero.jpg`

---

## Slide 2 — Python GUI Screenshot

**What:** Your PC screen showing `digit_predictor_gui.py` — a digit drawn on the canvas,
the big prediction number displayed on the right, and the logit bar chart filled in.

**How to take it:** Windows Snipping Tool (Win + Shift + S) or full screenshot.

**Save as:** `doc/slide2_gui_prediction.png`

---

## Slide 3 — End-to-End Shot (GUI + Board together)

**What:** Your PC screen showing the GUI on one side, DE2 board visible in the background
or placed next to the monitor. Shows the full system working together.

**How to take it:** Phone photo of your desk setup — monitor + board in one frame.

**Save as:** `doc/slide3_full_system.jpg`

---

## Slide 4 — Quartus Compilation Success Screenshot

**What:** Quartus II flow report or compilation window showing:
- Flow Status: Successful
- Total logic elements: 26,874 / 33,216 (81%)
- Device: EP2C35F672C6

**How to take it:** Screenshot of the Quartus summary window or the flow report text.

**Save as:** `doc/slide4_quartus_compile.png`

---

## Slide 5 — UART Wiring Close-Up

**What:** Close-up of the HW597 / USB-UART adapter plugged into the GPIO_1 pins on the DE2 board.
Shows the hardware integration is real, not just simulation.

**How to take it:** Phone camera macro shot of the connector on the GPIO header.

**Save as:** `doc/slide5_uart_wiring.jpg`

---

## Slide 6 — Verilog Code Snippet (optional but good for VLSI audience)

**What:** A clean screenshot of the MAC section from `rtl/mnist_top.v`:

```verilog
rom_byte = fc1_w[flat[16:1]];
w_val    = flat[0] ? $signed(rom_byte[7:4]) : $signed(rom_byte[3:0]);
prod     = $signed(w_val) * $signed(x_val);
acc     <= acc + {{12{prod[10]}}, prod};
```

Open the file in VS Code with a dark theme, zoom in, screenshot.

**Save as:** `doc/slide6_verilog_mac.png`

---

## Bonus — Short Video (highest LinkedIn reach)

Record a 10-15 second video:
1. Draw a digit in the GUI
2. Click SEND TO FPGA
3. Show the 7-segment display updating with the result

Even a phone recording of the screen + board is fine.

**Save as:** `doc/demo_video.mp4`

---

## Posting Order

1. slide1_fpga_hero.jpg         (hero — first image shown)
2. slide2_gui_prediction.png    (software side)
3. slide3_full_system.jpg       (end-to-end proof)
4. slide4_quartus_compile.png   (credibility for VLSI/FPGA audience)
5. slide5_uart_wiring.jpg       (hardware integration detail)
6. slide6_verilog_mac.png       (depth — for technical viewers)
