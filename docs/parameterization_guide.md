# Parameterization and Scalability Guide

This document defines the two key parameters that allow the design to scale without architectural changes: **symbol count** and **link data width**. All RTL must be written against these parameters from day one so that scaling is a one-line change + re-synthesis.

---

## 1. Design Parameters Summary

| Parameter | Location | Default | Upgrade Target | What Changes |
|-----------|----------|---------|----------------|-------------|
| `NUM_SYMBOLS` | `hft_pkg.sv` | 4 | 8 or 16 | More per-symbol registers, wider round-robin, more AXI readout registers |
| `LINK_DATA_W` | `hft_pkg.sv` | 4 | 8 | Wider PMOD data bus, fewer beats per frame, 4 extra jumper wires |

Both parameters are independent. You can change one without affecting the other.

---

## 2. Symbol Count (`NUM_SYMBOLS`)

### 2.1 What the parameter controls

Every module that has per-symbol state uses `NUM_SYMBOLS` to size its arrays:

```systemverilog
// In hft_pkg.sv
localparam int NUM_SYMBOLS = 4;  // change to 8, 16, etc.
```

### 2.2 Modules affected

| Module | What scales | How |
|--------|-----------|-----|
| `market_sim.sv` | `mid_price[0:N-1]`, `spread[0:N-1]`, symbol round-robin counter | `generate` loop or array sized by NUM_SYMBOLS |
| `exchange_lite.sv` | `bid_price[0:N-1]`, `ask_price[0:N-1]` lookup | Indexed by `symbol_id` field from ORDER |
| `quote_book.sv` | `best_bid[0:N-1]`, `best_ask[0:N-1]`, `bid_size[0:N-1]`, `ask_size[0:N-1]` | Array sized by NUM_SYMBOLS |
| `feature_compute.sv` | `ema[0:N-1]` register per symbol | Array sized by NUM_SYMBOLS |
| `strategy_engine.sv` | No internal per-symbol state (operates on whichever symbol is presented) | No change needed |
| `risk_manager.sv` | Reads `position[0:N-1]` from position_tracker | Input port width scales |
| `position_tracker.sv` | `position[0:N-1]`, contributes to single `cash` accumulator | Array sized by NUM_SYMBOLS |
| `board_a_axi_regs.sv` | Init price/spread registers: 2 registers per symbol | Register count scales |
| `board_b_axi_regs.sv` | Position readout: 1 register per symbol. Optional: bid/ask/ema/spread readout: 4 registers per symbol | Register count scales |

### 2.3 Resource cost per symbol

| Resource | Per Symbol | 4 Symbols | 8 Symbols | 16 Symbols |
|----------|-----------|-----------|-----------|------------|
| FFs (quote_book) | 96 | 384 | 768 | 1,536 |
| FFs (EMA state) | 32 | 128 | 256 | 512 |
| FFs (position) | 16 | 64 | 128 | 256 |
| FFs (market_sim state) | 64 | 256 | 512 | 1,024 |
| AXI registers (Board A) | 2 (8 bytes) | 8 | 16 | 32 |
| AXI registers (Board B, basic) | 1 (4 bytes) | 4 | 8 | 16 |
| AXI registers (Board B, extended) | 5 (20 bytes) | 20 | 40 | 80 |
| **Total FFs** | **~208** | **~832** | **~1,664** | **~3,328** |
| **% of 141,120 FFs** | | **0.6%** | **1.2%** | **2.4%** |

No resource concern at any reasonable symbol count.

### 2.4 Impact on throughput

Quotes are generated round-robin: one symbol per timer tick. With `quote_interval` cycles between ticks:

```
Per-symbol quote rate = core_clk_freq / (quote_interval * NUM_SYMBOLS)
```

| NUM_SYMBOLS | quote_interval=4000 | quote_interval=400 | quote_interval=4 |
|-------------|--------------------|--------------------|------------------|
| 4 | 6.25K qps/sym | 62.5K qps/sym | 6.25M qps/sym (link-limited) |
| 8 | 3.125K qps/sym | 31.25K qps/sym | 3.125M qps/sym (link-limited) |
| 16 | 1.5625K qps/sym | 15.6K qps/sym | 1.5M qps/sym (link-limited) |

Total system quote rate stays the same (set by quote_interval). Each symbol just gets a proportionally smaller share. For the demo this is fine -- the dashboard shows total throughput, and per-symbol positions all move independently.

### 2.5 Coding pattern

Every per-symbol array must use `NUM_SYMBOLS`, never a hardcoded number:

```systemverilog
// CORRECT
logic [31:0] mid_price [NUM_SYMBOLS];
for (genvar s = 0; s < NUM_SYMBOLS; s++) begin : gen_sym
    // per-symbol logic
end

// WRONG
logic [31:0] mid_price [4];         // hardcoded -- do not do this
logic [31:0] mid_price [0:3];       // hardcoded -- do not do this
```

The `symbol_id` field in messages is 8 bits (supports up to 256 symbols). Modules must bounds-check: `if (symbol_id < NUM_SYMBOLS)` before indexing arrays.

### 2.6 AXI register map scaling

The register map uses a base-plus-offset pattern so that adding symbols just extends the address range:

**Board A config registers (per-symbol init prices):**
```
Offset = 0x10 + symbol_id * 0x08
  +0x00: SYM[s]_INIT_MID
  +0x04: SYM[s]_INIT_SPREAD
```

For 4 symbols: 0x10..0x2F (32 bytes). For 16 symbols: 0x10..0x8F (128 bytes).

**Board B status registers (per-symbol readout):**
```
Basic (position only):
  Offset = 0x58 + symbol_id * 0x04
  +0x00: POS_SYM[s]

Extended (full per-symbol telemetry):
  Offset = 0x100 + symbol_id * 0x10
  +0x00: SYM[s]_BID
  +0x04: SYM[s]_ASK
  +0x08: SYM[s]_EMA
  +0x0C: SYM[s]_SPREAD
```

For 4 symbols (extended): 0x100..0x13F (64 bytes). For 16 symbols: 0x100..0x1FF (256 bytes). All well within the AXI-Lite address space.

### 2.7 Dashboard scaling

The telemetry JSON automatically includes all symbols:

```python
# In telemetry_server.py
positions = [regs.read(0x58 + s*4) for s in range(NUM_SYMBOLS)]
```

The dashboard's position bar chart auto-scales to however many symbols are in the JSON array. No dashboard code changes needed when NUM_SYMBOLS changes.

---

## 3. Link Data Width (`LINK_DATA_W`)

### 3.1 What the parameter controls

The number of PMOD data pins used per direction. Determines how many clock beats are needed to serialize a 128-bit frame.

```systemverilog
// In hft_pkg.sv
localparam int LINK_DATA_W      = 4;  // change to 8 for upgraded link
localparam int FRAME_WIDTH      = 128;
localparam int BEATS_PER_FRAME  = FRAME_WIDTH / LINK_DATA_W;  // 32 or 16
```

### 3.2 Physical pin mapping

**4-bit mode (default -- 2 standard PMOD cables only):**

```
Cable 1 (PmodA header, A→B direction):
  JA[0] (J12) = data[0]     Board A output, Board B input
  JA[1] (H12) = data[1]     Board A output, Board B input
  JA[2] (H11) = data[2]     Board A output, Board B input
  JA[3] (G10) = data[3]     Board A output, Board B input
  JA[4] (K13) = tx_valid    Board A output, Board B input
  JA[5] (K12) = rx_ready    Board B output, Board A input
  JA[6] (J11) = spare
  JA[7] (J10) = spare

Cable 2 (PmodB header, B→A direction):
  JB[0] (E12) = data[0]     Board B output, Board A input
  JB[1] (D11) = data[1]     Board B output, Board A input
  JB[2] (B11) = data[2]     Board B output, Board A input
  JB[3] (A10) = data[3]     Board B output, Board A input
  JB[4] (C11) = tx_valid    Board B output, Board A input
  JB[5] (B10) = rx_ready    Board A output, Board B input
  JB[6] (A12) = spare
  JB[7] (A11) = spare

Hardware needed: 2x standard 12-pin PMOD ribbon cables
```

**8-bit mode (upgrade -- 2 PMOD cables + 4 jumper wires):**

```
Cable 1 (PmodA header, A→B data):
  JA[0] (J12) = data[0]     Board A output, Board B input
  JA[1] (H12) = data[1]     Board A output, Board B input
  JA[2] (H11) = data[2]     Board A output, Board B input
  JA[3] (G10) = data[3]     Board A output, Board B input
  JA[4] (K13) = data[4]     Board A output, Board B input
  JA[5] (K12) = data[5]     Board A output, Board B input
  JA[6] (J11) = data[6]     Board A output, Board B input
  JA[7] (J10) = data[7]     Board A output, Board B input

Cable 2 (PmodB header, B→A data):
  JB[0] (E12) = data[0]     Board B output, Board A input
  JB[1] (D11) = data[1]     Board B output, Board A input
  JB[2] (B11) = data[2]     Board B output, Board A input
  JB[3] (A10) = data[3]     Board B output, Board A input
  JB[4] (C11) = data[4]     Board B output, Board A input
  JB[5] (B10) = data[5]     Board B output, Board A input
  JB[6] (A12) = data[6]     Board B output, Board A input
  JB[7] (A11) = data[7]     Board B output, Board A input

Jumper wires (JAB pins, control signals):
  JAB[0] (F12) = a2b_valid   Board A output, Board B input
  JAB[1] (G11) = b2a_valid   Board B output, Board A input
  JAB[2] (E10) = a2b_ready   Board B output, Board A input
  JAB[3] (D11*) = b2a_ready  Board A output, Board B input

  JAB[4] (F10) = spare
  JAB[5] (F11) = spare

Hardware needed: 2x standard 12-pin PMOD cables + 4x female-female Dupont jumper wires

* NOTE: JAB[3] ball assignment must be verified from the board XDC file.
  D11 appears in the pinout table for both JB[1] and potentially JAB[3].
  Check the official board constraints file to confirm the correct ball for JAB[3].
```

### 3.3 Performance comparison

| Metric | LINK_DATA_W = 4 | LINK_DATA_W = 8 |
|--------|-----------------|-----------------|
| Beats per frame | 32 | 16 |
| Frame serialization time | 640 ns | 320 ns |
| Inter-frame gap | 40 ns (2 beats) | 40 ns (2 beats) |
| Max frame rate | ~1.47M fps | ~2.78M fps |
| Round-trip wire time | ~1.36 us | ~0.68 us |
| Wiring complexity | 2 standard cables | 2 cables + 4 jumpers |

### 3.4 Modules affected

Only two modules change behavior based on `LINK_DATA_W`:

| Module | What changes |
|--------|-------------|
| `link_tx.sv` | Shift register outputs `DATA_W` bits per beat. Beat counter counts to `BEATS_PER_FRAME - 1`. Output port is `[DATA_W-1:0]`. |
| `link_rx.sv` | Captures `DATA_W` bits per beat into shift register. Beat counter counts to `BEATS_PER_FRAME - 1`. Input port is `[DATA_W-1:0]`. |

No other module is affected. The link presents complete 128-bit frames to the rest of the system regardless of how they were serialized.

### 3.5 Coding pattern for link modules

```systemverilog
module link_tx #(
    parameter int DATA_W = hft_pkg::LINK_DATA_W,
    parameter int FRAME_W = hft_pkg::FRAME_WIDTH,
    parameter int BEATS = FRAME_W / DATA_W
)(
    input  logic                clk,
    input  logic                rst,
    input  logic [FRAME_W-1:0]  frame_in,
    input  logic                frame_in_valid,
    output logic                frame_in_ready,
    input  logic                remote_ready,
    output logic [DATA_W-1:0]   pmod_data,
    output logic                pmod_valid
);

    logic [$clog2(BEATS)-1:0] beat_count;
    logic [FRAME_W-1:0] shift_reg;
    logic tick;  // 50 MHz clock enable (toggles every core_clk cycle)

    // Serialization: shift DATA_W bits per beat
    // MSB first: pmod_data = shift_reg[FRAME_W-1 -: DATA_W]
    // After each beat: shift_reg <<= DATA_W

endmodule
```

```systemverilog
module link_rx #(
    parameter int DATA_W = hft_pkg::LINK_DATA_W,
    parameter int FRAME_W = hft_pkg::FRAME_WIDTH,
    parameter int BEATS = FRAME_W / DATA_W
)(
    input  logic                clk,
    input  logic                rst,
    input  logic [DATA_W-1:0]   pmod_data,
    input  logic                pmod_valid,
    output logic                pmod_ready,
    output logic [FRAME_W-1:0]  frame_out,
    output logic                frame_out_valid
);

    logic [$clog2(BEATS)-1:0] beat_count;
    logic [FRAME_W-1:0] assemble_reg;

    // Assembly: shift in DATA_W bits per beat
    // assemble_reg <= {assemble_reg[FRAME_W-DATA_W-1:0], synced_data}
    // After BEATS beats: frame_out = assemble_reg, frame_out_valid = 1

endmodule
```

### 3.6 XDC constraint handling

The XDC file can include both 4-bit and 8-bit pin assignments. Unused pins are simply not connected in the RTL (Vivado ignores constraints for ports that don't exist in the design).

Alternatively, use conditional generation in the top-level to expose different port widths:

```systemverilog
// In board_a_top.sv
module board_a_top (
    // PmodA: always 8 pins, but usage depends on LINK_DATA_W
    inout [7:0] pmoda,
    inout [7:0] pmodb,

    // JAB: only used when LINK_DATA_W == 8
    `ifdef LINK_8BIT
    output       jab_a2b_valid,
    input        jab_a2b_ready,
    input        jab_b2a_valid,
    output       jab_b2a_ready,
    `endif
    ...
);
```

Or more cleanly, always declare all ports and let the synthesizer optimize away unused ones.

---

## 4. Upgrade Roadmap

### Phase 1: Bring-up (Weeks 1--7)

```
NUM_SYMBOLS  = 4
LINK_DATA_W  = 4
Hardware:      2x standard PMOD cables
```

Focus on getting the closed loop working. Fewer symbols = simpler debug traces in ILA. 4-bit link = no custom wiring.

### Phase 2: Link upgrade (Week 8)

```
NUM_SYMBOLS  = 4
LINK_DATA_W  = 8
Hardware:      2x PMOD cables + 4x Dupont jumper wires on JAB pins
```

Change one constant, re-synthesize, add 4 jumper wires. Verify link still works (Phase 5 smoke test). Round-trip latency drops by ~680 ns.

### Phase 3: Symbol scaling (Week 9)

```
NUM_SYMBOLS  = 8 (or 16)
LINK_DATA_W  = 8
Hardware:      same as Phase 2
```

Change one constant, re-synthesize. Dashboard automatically shows more symbol bars. Verify all symbols trade independently.

### Phase 4: Demo day (Week 10)

```
NUM_SYMBOLS  = 8
LINK_DATA_W  = 8
```

Present the parameterized design. Mention to the committee: "our architecture scales to 256 symbols and 8-bit link width with single-parameter changes, demonstrated by our Phase 1 -> Phase 3 progression."

---

## 5. Package Definition (for reference)

The final `hft_pkg.sv` should contain these as the single source of truth:

```systemverilog
package hft_pkg;

    // === Scalable Parameters ===
    localparam int NUM_SYMBOLS     = 4;    // 4 for bring-up, 8+ for demo
    localparam int LINK_DATA_W     = 4;    // 4 for bring-up, 8 for final
    localparam int FRAME_WIDTH     = 128;  // fixed: all messages are 128 bits
    localparam int BEATS_PER_FRAME = FRAME_WIDTH / LINK_DATA_W;

    // === Type Definitions ===
    typedef logic [31:0]        price_t;      // Q16.16 unsigned
    typedef logic signed [31:0] pnl_t;        // Q16.16 signed
    typedef logic [15:0]        qty_t;        // unsigned integer
    typedef logic [7:0]         symbol_t;     // supports up to 256 symbols
    typedef logic [15:0]        order_id_t;   // wrapping counter
    typedef logic [15:0]        timestamp_t;  // low 16 bits of cycle counter

    // === Message Types ===
    typedef enum logic [3:0] {
        MSG_QUOTE  = 4'h1,
        MSG_ORDER  = 4'h2,
        MSG_FILL   = 4'h3
    } msg_type_e;

    // === Regime Encoding ===
    typedef enum logic [1:0] {
        REGIME_CALM        = 2'b00,
        REGIME_VOLATILE    = 2'b01,
        REGIME_BURST       = 2'b10,
        REGIME_ADVERSARIAL = 2'b11
    } regime_e;

endpackage
```

All modules `import hft_pkg::*;` and use these definitions. No module should ever hardcode `4` where it means "number of symbols" or "data width."
