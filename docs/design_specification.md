# Dual-FPGA Deterministic Trading Engine

## Complete Design Specification

**Project**: ECE 554 Capstone  
**Platform**: 2x AMD AUP-ZU3 (Zynq UltraScale+ XCZU3EG-2SFVC784E)  
**Revision**: A1 -- March 2026

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Proposed Solution](#2-proposed-solution)
3. [Design Requirements](#3-design-requirements)
4. [Demo Approach](#4-demo-approach)
5. [Hardware System Specification](#5-hardware-system-specification)
6. [Software System Specification](#6-software-system-specification)
7. [Submodule Catalog and Interconnect](#7-submodule-catalog-and-interconnect)
8. [Control Logic and Operational Flow](#8-control-logic-and-operational-flow)
9. [Vivado Build Strategy](#9-vivado-build-strategy)
10. [Testing Plan](#10-testing-plan)
11. [Risk Analysis](#11-risk-analysis)
12. [Bill of Materials](#12-bill-of-materials)
13. [Appendix A: XDC Constraints](#appendix-a-xdc-constraints)
14. [Appendix B: Message Format Bit Maps](#appendix-b-message-format-bit-maps)
15. [Appendix C: AXI-Lite Register Maps](#appendix-c-axi-lite-register-maps)

---

## 1. Problem Statement

### 1.1 The Core Challenge

Modern electronic trading demands **deterministic, sub-microsecond decision-making**. General-purpose CPUs cannot guarantee this: OS scheduling, cache misses, and interrupts introduce unpredictable jitter (typically 10--100 us), making software-based latency measurement unreliable and software-based trading non-deterministic.

We build a **closed-loop, dual-FPGA trading testbed** that demonstrates:

- **Hardware-deterministic processing** -- every market quote processed and responded to in a fixed, predictable number of clock cycles with zero jitter.
- **Accurate hardware-based measurement** -- latency, throughput, and risk metrics computed entirely in FPGA fabric, free from OS/software contamination.
- **Realistic market stress testing** -- the system behaves differently under four market regimes (calm, volatile, burst, adversarial), and all behavioral changes are measurable in real time.

### 1.2 Why Two Separate FPGAs?

A single FPGA looping back to itself would be trivially simple and unrealistic. Two physically separate boards force real engineering problems:

- **Physical-layer link design** -- a deterministic, high-throughput serial bus across a cable.
- **Clock domain crossing** -- two independent oscillators require proper CDC.
- **Separation of concerns** -- Exchange and Trader are distinct entities with distinct logic, mirroring the real world.
- **Demo credibility** -- judges see two physical boards with a visible cable carrying live data.

### 1.3 What This Is NOT

- Not a full stock exchange (no deep order book, no multiple participants).
- Not algorithmic trading research (strategy is intentionally simple; the point is the hardware pipeline).
- Not a networking project (the link is custom PL-to-PL, not Ethernet/TCP).

The project is an **FPGA systems engineering demonstration**: custom interconnect, real-time pipelines, hardware measurement, and a polished demo.

---

## 2. Proposed Solution

### 2.1 Concept

| Board | Role | Real-World Analogy |
|-------|------|--------------------|
| Board A | Exchange-Lite + Market Simulator | A simplified NASDAQ |
| Board B | Trader (strategy + risk + telemetry) | A trading firm's FPGA engine |
| Laptop | Dashboard (read-only display) | Operations monitoring screen |

Board A continuously generates synthetic market quotes (bid/ask prices for multiple symbols) and sends them to Board B over a custom PMOD streaming link. Board B processes quotes through a deterministic pipeline -- feature computation, strategy, risk checks -- and sends orders back. Board A matches orders against current prices and returns fill or reject responses. Board B measures round-trip latency in hardware and exposes all metrics to a laptop dashboard via UART.

### 2.2 Data Lifecycle

```
Board A                          Cable               Board B
--------                     (2x PMOD)               --------
Market Sim generates quote ─────────────────────────> Quote book update
                                                      Feature compute (EMA)
                                                      Strategy decision
                                                      Risk check
                             <───────────────────────── Order sent
Exchange matches order
Fill (or reject) generated ─────────────────────────> Fill received
                                                      Position + PnL update
                                                      Latency measured
                                                      Stats to dashboard
```

### 2.3 Why This Solution

- **Meaningful closed loop** -- quotes, orders, and fills form a realistic market data cycle with clear semantics.
- **Demoable** -- judges see live throughput, latency histograms, PnL, and regime changes in real time.
- **Scalable complexity** -- baseline achievable in 10 weeks; stretch goals exist.
- **Teaching value** -- touches CDC, fixed-point DSP, AXI-Lite, state machines, PYNQ, and physical I/O.

---

## 3. Design Requirements

Each requirement is traced from a **functional need** to a **technical approach** to a **quantitative acceptance metric**.

### 3.1 Processing Speed

| Layer | Description |
|-------|-------------|
| **Functional** | Board B must process market quotes and generate orders in real-time with no buffering delay |
| **Technical** | Fully pipelined RTL in PL. No software, OS, cache, or interrupts in the critical path. Valid/ready handshake between stages. |
| **Quantitative** | Internal pipeline latency: **<= 10 clock cycles = 100 ns** at 100 MHz. Jitter: **0 ns** (deterministic). |

### 3.2 Throughput

| Layer | Description |
|-------|-------------|
| **Functional** | System must handle high quote rates including burst stress scenarios |
| **Technical** | 4-bit parallel PMOD link at 50 MHz effective data rate. 128-bit frames. 64-deep FIFOs absorb bursts. Hardware backpressure prevents loss. |
| **Quantitative** | Max sustained: **~1.47 million frames/sec per direction**. Demo target: >= 100K quotes/sec (CALM), >= 1M quotes/sec (BURST). |

### 3.3 Board-to-Board Connectivity

| Layer | Description |
|-------|-------------|
| **Functional** | Full-duplex, deterministic link between two physically separate FPGA boards |
| **Technical** | Custom streaming bus over 2x standard 12-pin PMOD ribbon cables. 4-bit data + valid + ready per direction. Mesochronous clocking (both boards use local 100 MHz, data output at 50 MHz rate via clock enable). |
| **Quantitative** | Wire latency per 128-bit frame: **680 ns** (34 link beats at 20 ns each). Frame loss: **0** (hardware backpressure). Bit error rate: **0** (LVCMOS33 digital signaling over 30 cm cable). |

### 3.4 Latency Measurement Accuracy

| Layer | Description |
|-------|-------------|
| **Functional** | Measure true round-trip trading latency in hardware, not contaminated by OS jitter |
| **Technical** | Board B embeds 16-bit cycle-counter timestamp in each ORDER. Board A echoes it in FILL. Board B computes `latency = current_cycle - echoed_timestamp` entirely in PL. 16-bin histogram in PL registers. |
| **Quantitative** | Resolution: **10 ns** (1 cycle at 100 MHz). Histogram: 16 bins x 32 cycles. Scalar stats: min, max, sum, count. p50/p99 computed on laptop from bins. |

### 3.5 Laptop Telemetry

| Layer | Description |
|-------|-------------|
| **Functional** | Real-time dashboard showing all trading metrics for demo audience |
| **Technical** | Board B PS reads AXI-Lite registers at 20 Hz, sends JSON over USB-UART (FTDI FT2232) to laptop. Laptop runs Python Plotly Dash web dashboard. |
| **Quantitative** | Refresh: **20 Hz** (50 ms). Panels: **8** (throughput, latency histogram, position, PnL, regime, risk rejects, link health, scalar stats). Baud: 115200 (upgradeable to 921600). |

### 3.6 Determinism

| Layer | Description |
|-------|-------------|
| **Functional** | Given same LFSR seed and regime sequence, system produces identical results |
| **Technical** | LFSR PRNG with configurable seed. All logic synchronous. No variable-latency operations. Proper CDC at link boundary only. |
| **Quantitative** | Pipeline latency variance: **0 cycles**. Identical seed + regime + parameters = identical trade sequence. |

### 3.7 Stress Resilience

| Layer | Description |
|-------|-------------|
| **Functional** | System stable under all 4 stress regimes with no data loss |
| **Technical** | FIFO backpressure. Error counters. 4 configurable regimes. |
| **Quantitative** | Sustained run: **>= 10 minutes** per regime. Frame loss: **0**. FIFO max fill: **< 75%**. |

### 3.8 Risk Management

| Layer | Description |
|-------|-------------|
| **Functional** | Trader enforces position, rate, and loss limits before every order |
| **Technical** | Three independent checks in 1 pipeline stage. Any failure suppresses the order and increments reject counter. |
| **Quantitative** | Risk check latency: **1 cycle (10 ns)**. Position limit: +/-100 shares (configurable). Rate limit: 1000 orders/ms (configurable). Loss halt: -$16.00 (configurable). |

---

## 4. Demo Approach

### 4.1 Physical Setup

```
     ┌─────────────────┐    2x PMOD cables    ┌─────────────────┐
     │   Board A        │◄────────────────────►│   Board B        │
     │   (Exchange)      │                      │   (Trader)        │
     │   AUP-ZU3 #1     │                      │   AUP-ZU3 #2     │
     └─────────────────┘                      └────────┬────────┘
                                                       │ USB-C UART
                                                       ▼
                                               ┌───────────────┐
                                               │    Laptop      │
                                               │  (Dashboard)   │
                                               └───────────────┘
```

### 4.2 Dashboard Panels (8 total)

1. **Throughput gauges** -- quotes/sec, orders/sec, fills/sec
2. **Latency histogram** -- 16-bin bar chart with p50/p99/max annotated
3. **Position bars** -- per-symbol position (green = long, red = short)
4. **PnL line chart** -- running profit/loss over time
5. **Regime indicator** -- current stress mode label + color
6. **Risk reject counter** -- jumps when limits hit
7. **Link health** -- error count (should be 0)
8. **Scalar stats** -- p50, p99, max latency in nanoseconds

### 4.3 Scripted Demo Flow

| Step | Action | What Audience Sees |
|------|--------|--------------------|
| 1 | Power on boards, PYNQ boots | LEDs come on, dashboard connects |
| 2 | Run config script on Board A (CALM regime) | Quotes flowing on dashboard |
| 3 | Enable trading on Board B (switch or script) | Orders + fills begin, PnL moves |
| 4 | Switch to VOLATILE (flip switch on Board A) | Spread widens, PnL swings larger |
| 5 | Switch to BURST | Quote rate spikes to ~1M+/sec |
| 6 | Switch to ADVERSARIAL | Risk rejects spike, position limits hit |
| 7 | Return to CALM | System stabilizes, proves resilience |
| 8 | Highlight link errors = 0 | Proves zero frame loss under all regimes |

---

## 5. Hardware System Specification

### 5.1 Target Device Resources

From the XCZU3EG-2SFVC784E:

| Resource | Available | Our Est. Usage (per board) | Utilization |
|----------|-----------|---------------------------|-------------|
| CLB LUTs | 70,560 | ~5,000 | ~7% |
| CLB Flip-Flops | 141,120 | ~4,000 | ~3% |
| Block RAM (18Kb) | 432 | ~8 | ~2% |
| DSP48E2 Slices | 360 | ~5 | ~1.4% |
| UltraRAM (Mb) | 7.6 | 0 | 0% |

Massive headroom. No resource concerns.

### 5.2 Clock Architecture

From the AUP-ZU3 reference manual, the clocking scheme:

- **PS Reference Clock**: 33.33 MHz oscillator -> PS PLL -> FCLK outputs to PL
- **PL System Clock**: 100 MHz LVDS from TI CDCE6214 clock generator on pins D6 (P) / D7 (N)

**Our design uses PS FCLK0 = 100 MHz** as the single core clock for all PL logic. This is the standard PYNQ approach and eliminates clock domain crossings between AXI-Lite and our data-plane. The FCLK0 frequency is configured in the Zynq PS IP block within the Vivado block design.

The ONLY clock domain crossing in the entire design is at the PMOD link RX boundary, where incoming data (from the other board's clock domain) is synchronized via 2-FF synchronizers.

```
Clock Domains (per board):

  ┌──────────────────────────────────────────┐
  │  PS FCLK0 = 100 MHz                      │
  │  Drives: ALL PL logic                    │
  │    - Pipeline modules                     │
  │    - AXI-Lite interconnect                │
  │    - Counters, FIFOs, state machines     │
  │    - LED/button/switch logic              │
  │    - Link TX (with 50 MHz clock enable)  │
  │    - Link RX (after 2-FF synchronizer)   │
  └──────────────────────────────────────────┘

  Only CDC boundary: incoming PMOD data/valid
  synchronized via 2-FF synchronizers into core_clk domain
```

### 5.3 Board A: Exchange-Lite + Market Simulator

```
                           ┌─────────────────────────────────────────────┐
                           │              Board A (PL)                   │
                           │                                             │
  ┌──────────┐   AXI-Lite  │   ┌──────────┐    ┌──────────────┐         │
  │  PS ARM  │◄───────────►│   │  axi_regs │───►│  market_sim  │         │
  │  (PYNQ)  │             │   │           │    │  (LFSR price │         │
  └──────────┘             │   │           │    │   evolution)  │         │
                           │   └──────────┘    └──────┬───────┘         │
                           │       │                   │ quote frames    │
                           │       │ config     ┌──────▼───────┐         │
                           │       │            │  quote_fifo   │         │
                           │       │            │  (64 x 128b)  │         │
                           │   ┌───▼─────┐     └──────┬───────┘         │
                           │   │  ctrl   │            │                  │
                           │   │(btn/sw/ │     ┌──────▼───────┐         │
                           │   │  led)   │     │  tx_arbiter   │◄─┐     │
                           │   └─────────┘     │(fills>quotes) │  │     │
                           │                   └──────┬───────┘  │     │
                           │                          │ frames    │     │
  PmodA pins ◄─────────────│───────────────── link_tx ┘           │     │
  (to Board B)             │                                      │     │
                           │                          fill frames │     │
  PmodB pins ──────────────│──► link_rx ──► exchange_lite ────────┘     │
  (from Board B)           │                 (order matching)           │
                           │                  ▲                         │
                           │                  │ bid/ask state           │
                           │                  │ (from market_sim)       │
                           └──────────────────┼─────────────────────────┘
```

**Data flows:**
1. `market_sim` generates QUOTE frames using LFSR-driven price random walk
2. Quotes enter `quote_fifo`, then `tx_arbiter`, then `link_tx` out PmodA
3. Orders arrive on PmodB into `link_rx`, fed to `exchange_lite`
4. `exchange_lite` compares limit price vs current bid/ask, produces FILL or REJECT
5. Fills enter `fill_fifo`, then `tx_arbiter` (priority over quotes) to `link_tx`

### 5.4 Board B: Trader

```
                           ┌──────────────────────────────────────────────────┐
                           │               Board B (PL)                       │
                           │                                                  │
  ┌──────────┐   AXI-Lite  │   ┌──────────┐                                  │
  │  PS ARM  │◄───────────►│   │  axi_regs │──► config to pipeline modules   │
  │  (PYNQ)  │             │   │           │◄── status/counters/histogram     │
  │  sends   │             │   └──────────┘                                  │
  │  JSON    │             │       │                                          │
  │  over    │             │   ┌───▼─────┐                                   │
  │  UART    │             │   │  ctrl   │  (btn/sw/led)                     │
  └──────────┘             │   └─────────┘                                   │
                           │                                                  │
  PmodA pins ──────────────│──► link_rx ──► msg_demux ──┬── (QUOTE) ──►      │
  (from Board A)           │                            │  quote_book         │
                           │                            │      │              │
                           │                            │  feature_compute    │
                           │                            │  (mid, spread, EMA) │
                           │                            │      │              │
                           │                            │  strategy_engine    │
                           │                            │  (mean-reversion)   │
                           │                            │      │              │
                           │                            │  risk_manager       │
                           │                            │  (pos/rate/loss)    │
                           │                            │      │              │
                           │                            │  order_manager      │
                           │                            │      │              │
  PmodB pins ◄─────────────│──────────── link_tx ◄──────┘      │              │
  (to Board A)             │                                   │              │
                           │                    ┌── (FILL) ────┘              │
                           │                    │                             │
                           │              position_tracker                    │
                           │              (position + cash)                   │
                           │                    │                             │
                           │              latency_histogram                   │
                           │              (16 bins + stats)                   │
                           └──────────────────────────────────────────────────┘
```

**Pipeline stages (quote arrival to order departure):**

| Stage | Module | Cycles | Operation |
|-------|--------|--------|-----------|
| 1 | msg_demux | 1 | Route by msg_type |
| 2 | quote_book | 1 | Update bid/ask registers |
| 3 | feature_compute | 1 | mid = (bid+ask)>>1, spread = ask-bid |
| 4 | feature_compute | 2 | EMA multiply-accumulate (uses DSP48) |
| 5 | strategy_engine | 1 | Compare deviation vs threshold |
| 6 | risk_manager | 1 | 3 parallel limit checks |
| 7 | order_manager | 1 | Build ORDER frame, assign ID + timestamp |
| **Total** | | **8** | **80 ns at 100 MHz** |

### 5.5 Board-to-Board Link Layer

#### 5.5.1 Physical Layer

The AUP-ZU3 has a single Pmod+ connector providing two standard 12-pin PMOD headers (PmodA and PmodB) plus 6 extra Pmod+ signals (JAB). Each standard PMOD header has 8 signal pins at **LVCMOS33** (3.3V I/O bank).

**Critical constraint from reference manual**: PMOD signals support speeds **up to approximately 50 MHz**.

We use **two standard 12-pin PMOD ribbon cables** -- one per direction:

**Cable 1 -- PmodA (Quotes + Fills: Board A -> Board B):**

| PMOD Pin | Signal Name | FPGA Ball | Board A Dir | Board B Dir | Function |
|----------|-------------|-----------|-------------|-------------|----------|
| JA_[0] | link_a2b_data[0] | J12 | OUTPUT | INPUT | Data nibble bit 0 |
| JA_[1] | link_a2b_data[1] | H12 | OUTPUT | INPUT | Data nibble bit 1 |
| JA_[2] | link_a2b_data[2] | H11 | OUTPUT | INPUT | Data nibble bit 2 |
| JA_[3] | link_a2b_data[3] | G10 | OUTPUT | INPUT | Data nibble bit 3 |
| JA_[4] | link_a2b_valid | K13 | OUTPUT | INPUT | High during frame TX |
| JA_[5] | link_a2b_ready | K12 | INPUT | OUTPUT | Backpressure from B |
| JA_[6] | spare_a0 | J11 | -- | -- | Reserved |
| JA_[7] | spare_a1 | J10 | -- | -- | Reserved |

**Cable 2 -- PmodB (Orders: Board B -> Board A):**

| PMOD Pin | Signal Name | FPGA Ball | Board A Dir | Board B Dir | Function |
|----------|-------------|-----------|-------------|-------------|----------|
| JB_[0] | link_b2a_data[0] | E12 | INPUT | OUTPUT | Data nibble bit 0 |
| JB_[1] | link_b2a_data[1] | D11 | INPUT | OUTPUT | Data nibble bit 1 |
| JB_[2] | link_b2a_data[2] | B11 | INPUT | OUTPUT | Data nibble bit 2 |
| JB_[3] | link_b2a_data[3] | A10 | INPUT | OUTPUT | Data nibble bit 3 |
| JB_[4] | link_b2a_valid | C11 | INPUT | OUTPUT | High during frame TX |
| JB_[5] | link_b2a_ready | B10 | OUTPUT | INPUT | Backpressure from A |
| JB_[6] | spare_b0 | A12 | -- | -- | Reserved |
| JB_[7] | spare_b1 | A11 | -- | -- | Reserved |

#### 5.5.2 Clocking Strategy -- Mesochronous (No Forwarded Clock)

Both boards run their PL at 100 MHz from their own PS FCLK0. Data is output at an **effective 50 MHz rate** using a clock-enable toggle (data held for 2 core_clk cycles = 20 ns, within the ~50 MHz PMOD speed limit).

No forwarded clock pin is needed. The receiver synchronizes the incoming `valid` signal through a 2-FF synchronizer and uses it to detect frame boundaries. Data is sampled on the core_clk edge where synchronized valid is stable.

**Why this works at 50 MHz data rate / 100 MHz sample rate:**
- Data changes every 20 ns (50 MHz)
- Receiver samples every 10 ns (100 MHz)
- After 2-FF synchronizer (20 ns latency), synchronized valid transitions are clean
- Receiver uses rising edge of synchronized valid to start a frame counter
- Frame is 32 nibbles; both clocks are ~100 MHz so cumulative drift over 32 data beats (640 ns) is < 0.032 cycles at 50 ppm -- negligible
- Inter-frame gap (>= 2 data beats = 40 ns) absorbs any accumulated drift

#### 5.5.3 Frame Protocol

```
TX side (100 MHz core_clk, 50 MHz data rate via tick enable):

  tick: _/‾\_/‾\_/‾\_/‾\_/‾\_ ...  (toggles every core_clk, = 50 MHz)

  On each tick=1:
    - Drive data[3:0] with next nibble of 128-bit frame (MSB first)
    - Hold data stable until next tick=1 (2 core_clk cycles)

  valid: ___/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\___
              |<------------ 32 data beats ------------->| gap
              N0   N1   N2   N3  ...  N30  N31   idle

  N0  = frame[127:124]  (MSB nibble first)
  N31 = frame[3:0]      (LSB nibble last)
  Gap = valid low for >= 2 data beats (40 ns minimum)
```

```
RX side (100 MHz core_clk):

  1. 2-FF synchronize incoming valid and data[3:0]
  2. Detect rising edge of synchronized valid -> start frame capture
  3. Generate internal 50 MHz tick (phase-aligned to detected edge)
  4. Sample synchronized data on each tick for 32 beats
  5. Assemble 128-bit frame in shift register
  6. After 32 beats: frame complete, push to output FIFO
  7. Assert ready = !(output_fifo_almost_full)
```

#### 5.5.4 Throughput and Latency

| Metric | Value | Derivation |
|--------|-------|------------|
| Data rate per direction | 200 Mbps | 4 bits x 50 MHz |
| Frame size | 128 bits | Fixed for all message types |
| Beats per frame | 32 | 128 / 4 |
| Time per frame | 640 ns | 32 x 20 ns |
| Inter-frame gap | 40 ns min | 2 beats x 20 ns |
| Max frame rate | ~1.47M/sec | 1 / (640 + 40) ns |
| Link utilization at 100K qps | ~6.8% | 100K / 1.47M |
| Link utilization at 1M qps | ~68% | 1M / 1.47M |

### 5.6 Number Representation

| Type | Width | Format | Range | Example |
|------|-------|--------|-------|---------|
| Price | 32 bits | Unsigned Q16.16 | 0 -- 65535.9999 | $150.25 = 0x0096_4000 |
| PnL / Signed Price | 32 bits | Signed Q16.16 | -32768 -- +32767.9999 | -$5.50 = 0xFFFA_8000 |
| Cash Accumulator | 48 bits | Signed Q32.16 | +/- 2 billion | Prevents overflow on accumulation |
| Quantity | 16 bits | Unsigned integer | 0 -- 65535 | 100 shares = 0x0064 |
| Symbol ID | 8 bits | Unsigned integer | 0 -- 255 | 0 = "SYM0" |
| Order ID | 16 bits | Unsigned integer | 0 -- 65535 | Wrapping counter |
| Timestamp | 16 bits | Unsigned integer | 0 -- 65535 | Low 16 bits of cycle counter |

**Fixed-point arithmetic rules:**
- Addition/subtraction: direct (same format)
- price x qty: `result_48 = price_32 * qty_16` (uses 1 DSP48E2 slice). Result is Q32.16 for cash accumulation
- EMA: `ema_new = (alpha * sample + (65536 - alpha) * ema_old) >> 16` (uses 2 DSP48E2 slices)

---

## 6. Software System Specification

### 6.1 PS Software (PYNQ on ARM Cortex-A53)

The Processing System runs PYNQ Linux. We do **NOT** implement a soft-core processor (MicroBlaze/RISC-V) in PL. The hard ARM PS handles all slow-path tasks:

```
┌─────────────────────────────────────────────────┐
│  PYNQ Linux (on ARM Cortex-A53)                 │
│                                                  │
│  ┌────────────────────────────────────────────┐ │
│  │  Overlay Manager (pynq.Overlay)            │ │
│  │  - Loads .bit + .hwh into PL               │ │
│  └────────────────────────────────────────────┘ │
│                                                  │
│  ┌────────────────────────────────────────────┐ │
│  │  MMIO Register Access (pynq.MMIO)          │ │
│  │  - Reads/writes AXI-Lite mapped registers  │ │
│  └────────────────────────────────────────────┘ │
│                                                  │
│  ┌────────────────────────────────────────────┐ │
│  │  Board A: config_exchange.py               │ │
│  │  - Set LFSR seed, initial prices, regime   │ │
│  │  - Write start bit to begin simulation     │ │
│  └────────────────────────────────────────────┘ │
│                                                  │
│  ┌────────────────────────────────────────────┐ │
│  │  Board B: telemetry_server.py              │ │
│  │  - Read all status registers at 20 Hz      │ │
│  │  - Format as JSON, print to UART stdout    │ │
│  └────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

**Board A PS script** loads the overlay, writes config registers (regime, quote interval, prices, LFSR seed), and sets the start bit. Regime can be changed live via register write or overridden by physical switches.

**Board B PS script** loads the overlay, writes config registers (strategy threshold, EMA alpha, risk limits), then enters a 20 Hz polling loop: reads all status registers, formats as a JSON line, prints to stdout (which flows to the FTDI UART). The laptop captures this serial stream.

### 6.2 UART Telemetry Protocol

Board B PS sends one JSON line every 50 ms to the FTDI UART (COM port):

```json
{"qps":125000,"ops":5000,"fps":4950,"rej":42,"pos":[10,-5,0,0],"cash_lo":1234,"cash_hi":0,"hist":[100,500,300,50,10,0,0,0,0,0,0,0,0,0,0,0],"lat_min":28,"lat_max":487,"lat_cnt":4950,"link_err":0}
```

The laptop reads this via pyserial at 115200 baud (sufficient for ~200 bytes every 50 ms = 4 KB/s, well within 11.5 KB/s capacity of 115200 baud).

### 6.3 Laptop Dashboard

Python application using **Plotly Dash** (web-based, runs at localhost:8050).

```
┌──────────────────────────────────────────────────┐
│  Laptop                                           │
│                                                   │
│  ┌─────────────┐    ┌──────────────────────────┐ │
│  │  pyserial    │───►│  JSON line parser         │ │
│  │  (COM port)  │    │  + rate computation       │ │
│  └─────────────┘    │  (delta counter / delta t)│ │
│                      └──────────┬───────────────┘ │
│                                 │                  │
│                      ┌──────────▼───────────────┐ │
│                      │  Plotly Dash Web App      │ │
│                      │  8 panels, 20 Hz refresh  │ │
│                      │  http://localhost:8050     │ │
│                      └──────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

Rate computation: counters are monotonic. `rate = (counter_now - counter_prev) / delta_time`.

p50/p99: walk histogram bins cumulatively. p50 = bin where cumulative >= 50% of total. p99 = bin where cumulative >= 99%.

---

## 7. Submodule Catalog and Interconnect

### 7.1 Complete Module List (22 RTL modules)

#### Shared / Common (3 modules)

| # | Module | Description | Key Resources |
|---|--------|-------------|---------------|
| 1 | `hft_pkg.sv` | Package: types, constants, message structs | -- |
| 2 | `lfsr32.sv` | 32-bit Galois LFSR pseudo-random generator | 32 FFs |
| 3 | `debounce.sv` | Button debouncer (20 ms filter, ~2M cycle shift register) | ~20 FFs |

#### Link Layer (2 modules, instantiated on both boards)

| # | Module | Description | Key Resources |
|---|--------|-------------|---------------|
| 4 | `link_tx.sv` | 128-bit to 4-bit serializer with 50 MHz clock enable, valid/ready | ~200 LUTs |
| 5 | `link_rx.sv` | 4-bit to 128-bit deserializer, 2-FF sync on valid, frame assembly | ~250 LUTs |

#### Board A -- Data Plane (3 modules)

| # | Module | Description | Key Resources |
|---|--------|-------------|---------------|
| 6 | `market_sim.sv` | LFSR price random walk, per-symbol state, quote frame builder | ~500 LUTs, 1 DSP |
| 7 | `exchange_lite.sv` | Compares order limit_price vs bid/ask, generates FILL or REJECT | ~300 LUTs |
| 8 | `tx_arbiter.sv` | Strict-priority mux: fill_fifo before quote_fifo | ~100 LUTs |

#### Board A -- Control Plane (3 modules)

| # | Module | Description | Key Resources |
|---|--------|-------------|---------------|
| 9 | `board_a_axi_regs.sv` | AXI-Lite slave: config + status registers | ~400 LUTs |
| 10 | `board_a_ctrl.sv` | Button edge detect, switch sampling, LED drivers | ~100 LUTs |
| 11 | `board_a_top.sv` | Top-level structural wiring | -- |

#### Board B -- Data Plane (7 modules)

| # | Module | Description | Key Resources |
|---|--------|-------------|---------------|
| 12 | `msg_demux.sv` | Routes frames by msg_type (QUOTE vs FILL) | ~50 LUTs |
| 13 | `quote_book.sv` | 4-symbol register file: latest bid/ask/sizes | ~200 FFs |
| 14 | `feature_compute.sv` | mid, spread, EMA (fixed-point multiply-accumulate) | ~400 LUTs, 2 DSP |
| 15 | `strategy_engine.sv` | Mean-reversion: compare deviation vs threshold | ~200 LUTs |
| 16 | `risk_manager.sv` | 3 parallel checks (position, rate, loss), gate output | ~300 LUTs |
| 17 | `order_manager.sv` | Assign order_id (wrapping counter), capture timestamp, build frame | ~200 LUTs |
| 18 | `position_tracker.sv` | Signed position per symbol, 48-bit cash accumulator | ~300 LUTs, 1 DSP |

#### Board B -- Measurement (1 module)

| # | Module | Description | Key Resources |
|---|--------|-------------|---------------|
| 19 | `latency_histogram.sv` | 16 bins x 32-bit, min/max/sum/count registers | ~300 LUTs |

#### Board B -- Control Plane (3 modules)

| # | Module | Description | Key Resources |
|---|--------|-------------|---------------|
| 20 | `board_b_axi_regs.sv` | AXI-Lite slave: config + status + histogram readout | ~500 LUTs |
| 21 | `board_b_ctrl.sv` | Button edge detect, switch sampling, LED/RGB drivers | ~150 LUTs |
| 22 | `board_b_top.sv` | Top-level structural wiring | -- |

#### Testbenches (4 files)

| # | Testbench | Tests |
|---|-----------|-------|
| 23 | `tb_link.sv` | link_tx + link_rx loopback, 1000 random frames |
| 24 | `tb_board_a.sv` | Full Board A with synthetic orders |
| 25 | `tb_board_b.sv` | Full Board B with synthetic quotes + fills |
| 26 | `tb_system.sv` | Board A + Board B with behavioral link model |

### 7.2 Module Interface Definitions

Every data-plane module uses a **valid/ready handshake**:
```
Producer drives:  out_frame[127:0], out_valid
Consumer drives:  out_ready
Transfer when:    out_valid && out_ready (both high on same rising edge)
```

#### link_tx

```
Inputs:
  clk                    -- 100 MHz core clock
  rst                    -- synchronous reset
  frame_in[127:0]        -- frame to transmit
  frame_in_valid         -- frame available in FIFO
  remote_ready           -- synchronized rx_ready from remote board
Outputs:
  frame_in_ready         -- can accept a new frame
  pmod_data[3:0]         -- to PMOD data pins
  pmod_valid             -- to PMOD valid pin
Internal:
  tick                   -- toggles every core_clk cycle (50 MHz enable)
  nibble_counter[4:0]    -- 0..31 within a frame
  frame_shift[127:0]     -- shift register for serialization
```

#### link_rx

```
Inputs:
  clk                    -- 100 MHz core clock
  rst                    -- synchronous reset
  pmod_data[3:0]         -- from PMOD data pins
  pmod_valid             -- from PMOD valid pin
Outputs:
  frame_out[127:0]       -- assembled frame
  frame_out_valid        -- frame ready (one cycle pulse)
  pmod_ready             -- to PMOD ready pin (!fifo_almost_full)
  error_count[31:0]      -- framing error counter
Internal:
  valid_sync[1:0]        -- 2-FF synchronizer for pmod_valid
  data_sync[3:0]         -- 2-FF synchronizer for pmod_data (each bit)
  nibble_counter[4:0]    -- 0..31 frame assembly
  frame_assemble[127:0]  -- shift register
  tick                   -- internal 50 MHz enable, phase-locked to valid edge
```

#### market_sim

```
Inputs:
  clk, rst
  start                  -- from ctrl register
  regime[1:0]            -- current regime
  quote_interval[31:0]   -- cycles between quote rounds
  num_symbols[2:0]       -- 1..4
  sym_init_mid[0:3]      -- initial mid prices (Q16.16)
  sym_init_spread[0:3]   -- initial spreads (Q16.16)
  lfsr_seed[31:0]        -- PRNG seed (loaded on reset)
Outputs:
  quote_frame[127:0]     -- QUOTE message
  quote_valid, quote_ready (input)
  bid_price[0:3]         -- current bid per symbol (shared with exchange_lite)
  ask_price[0:3]         -- current ask per symbol
  quotes_sent[31:0]      -- counter
```

#### exchange_lite

```
Inputs:
  clk, rst
  order_frame[127:0]     -- from link_rx
  order_valid, order_ready (output)
  bid_price[0:3]         -- from market_sim
  ask_price[0:3]         -- from market_sim
Outputs:
  fill_frame[127:0]      -- FILL or REJECT message
  fill_valid, fill_ready (input)
  orders_received[31:0]  -- counter
  fills_sent[31:0]       -- counter
  rejects_sent[31:0]     -- counter
```

#### feature_compute

```
Inputs:
  clk, rst
  bid_price[31:0]        -- from quote_book
  ask_price[31:0]        -- from quote_book
  symbol_id[7:0]         -- which symbol updated
  quote_valid            -- new quote flag
  ema_alpha[15:0]        -- Q0.16 smoothing factor (configurable)
Outputs:
  mid_price[31:0]        -- (bid + ask) >> 1
  spread[31:0]           -- ask - bid
  ema[31:0]              -- exponential moving average
  deviation[31:0]        -- mid - ema (signed Q16.16)
  feature_valid          -- features ready
  symbol_id_out[7:0]     -- pass-through
```

#### strategy_engine

```
Inputs:
  clk, rst
  deviation[31:0]        -- signed Q16.16 from feature_compute
  bid_price[31:0]        -- from quote_book
  ask_price[31:0]        -- from quote_book
  symbol_id[7:0]
  feature_valid
  threshold[31:0]        -- configurable (Q16.16)
  base_qty[15:0]         -- order size
Outputs:
  order_side             -- 0=BUY, 1=SELL
  order_price[31:0]      -- limit price
  order_qty[15:0]
  order_symbol[7:0]
  signal_valid           -- trade signal (low = no trade)
```

#### risk_manager

```
Inputs:
  clk, rst
  signal_valid, order_side, order_price, order_qty, order_symbol
  position[0:3]          -- current signed position per symbol
  total_pnl[31:0]        -- from position_tracker
  max_position[15:0]     -- configurable
  max_order_rate[15:0]   -- configurable
  max_loss[31:0]         -- configurable (Q16.16)
  trading_enable         -- from switch
Outputs:
  approved_valid         -- order passes all checks
  order_side, order_price, order_qty, order_symbol (pass-through)
  risk_rejects[31:0]     -- counter
  risk_halt              -- max loss breached, all trading stopped
```

#### latency_histogram

```
Inputs:
  clk, rst
  fill_valid             -- fill received
  ts_echo[15:0]          -- echoed timestamp from fill
  cycle_counter[15:0]    -- free-running counter[15:0]
Outputs (readable via AXI-Lite):
  hist_bins[0:15]        -- 16 x 32-bit bin counters
  lat_min[15:0]          -- minimum observed latency
  lat_max[15:0]          -- maximum observed latency
  lat_sum[47:0]          -- sum (for average)
  lat_count[31:0]        -- sample count
```

---

## 8. Control Logic and Operational Flow

### 8.1 Do We Need a Processor in PL?

**No.** The Zynq UltraScale+ already contains a hard ARM Cortex-A53 (the PS). We use it for configuration and telemetry. All real-time data-plane logic is pure RTL state machines and combinational pipelines. No MicroBlaze, no RISC-V, no custom instruction set.

The base overlay's "Pmod IO Processors" (MicroBlaze instances) are NOT used. We replace the base overlay entirely with our own design.

### 8.2 Button / Switch / LED Mapping

#### Board A

| I/O | Pin(s) | FPGA Ball(s) | IOSTD | Function |
|-----|--------|-------------|-------|----------|
| SW[1:0] | PL_USER_SW0, SW1 | AB1, AF1 | LVCMOS12 | Regime: 00=CALM, 01=VOLATILE, 10=BURST, 11=ADVERSARIAL |
| SW[2] | PL_USER_SW2 | AE3 | LVCMOS12 | Override: 0=regime from PS register, 1=regime from switches |
| SW[3:7] | PL_USER_SW3..7 | AC2,AC1,AD6,AD1,AD2 | LVCMOS12 | Reserved |
| BTN[0] | PL_USER_PB0 | AB6 | LVCMOS33 | Start market simulator (edge-detected) |
| BTN[1] | PL_USER_PB1 | AB7 | LVCMOS33 | Stop market simulator |
| BTN[2] | PL_USER_PB2 | AB2 | LVCMOS33 | Reset all counters and state |
| BTN[3] | PL_USER_PB3 | AC6 | LVCMOS33 | Reserved |
| LED[3:0] | PL_USER_LED0..3 | AF5,AE7,AH2,AE5 | LVCMOS12 | Regime indicator (binary + blink when running) |
| LED[7:4] | PL_USER_LED4..7 | AH1,AE4,AG1,AF2 | LVCMOS12 | Link activity (flash on frame TX/RX) |
| RGB0 | PL_LEDRGB0_R/G/B | AD7,AD9,AE9 | LVCMOS12 | Regime color: green=CALM, yellow=VOLATILE, red=BURST, purple=ADVERSARIAL |
| RGB1 | PL_LEDRGB1_R/G/B | AG9,AE8,AF8 | LVCMOS12 | Link status: green=healthy, red=errors |

#### Board B

| I/O | Pin(s) | FPGA Ball(s) | IOSTD | Function |
|-----|--------|-------------|-------|----------|
| SW[0] | PL_USER_SW0 | AB1 | LVCMOS12 | Trading enable: 0=armed (observe), 1=live trading |
| SW[1:7] | PL_USER_SW1..7 | AF1,... | LVCMOS12 | Reserved |
| BTN[0] | PL_USER_PB0 | AB6 | LVCMOS33 | Start pipeline (edge-detected) |
| BTN[1] | PL_USER_PB1 | AB7 | LVCMOS33 | Stop pipeline |
| BTN[2] | PL_USER_PB2 | AB2 | LVCMOS33 | Reset counters, positions, histogram |
| BTN[3] | PL_USER_PB3 | AC6 | LVCMOS33 | Reserved |
| LED[3:0] | PL_USER_LED0..3 | AF5,AE7,AH2,AE5 | LVCMOS12 | Order activity (flash per order) |
| LED[7:4] | PL_USER_LED4..7 | AH1,AE4,AG1,AF2 | LVCMOS12 | Fill activity (flash per fill) |
| RGB0 | PL_LEDRGB0_R/G/B | AD7,AD9,AE9 | LVCMOS12 | PnL: green=profit, red=loss, off=flat |
| RGB1 | PL_LEDRGB1_R/G/B | AG9,AE8,AF8 | LVCMOS12 | Risk: green=OK, yellow=near limit (>80%), red=halted |

### 8.3 Debounce and Edge Detection

All 4 buttons pass through `debounce.sv`: a 20-bit shift register clocked at 100 MHz. Output changes only when all 20 samples agree (~200 us debounce window, well above mechanical bounce). After debounce, a 1-cycle rising-edge pulse is generated. The `board_x_ctrl` module ORs this pulse with the corresponding AXI-Lite register write, so either hardware button or PS software can trigger the same action.

### 8.4 Board A Operational States

```
     ┌─────────┐  reset_done   ┌──────┐
     │  RESET  │──────────────►│ IDLE │
     └─────────┘               └───┬──┘
         ▲                         │ start (btn OR register)
         │                         ▼
         │ reset              ┌─────────┐
         ├────────────────────│ RUNNING │◄──────┐
         │                    └────┬────┘       │
         │                         │ stop       │ start
         │                         ▼            │
         │                    ┌─────────┐       │
         └────────────────────│ STOPPED │───────┘
                              └─────────┘

RESET:   Clear all counters, FIFOs, LFSR. Load initial prices.
IDLE:    Waiting for start. PS writes config registers.
RUNNING: market_sim generating quotes. exchange_lite accepting orders.
         Regime selectable via switch or register at any time.
STOPPED: Quotes paused. Exchange still drains pending orders/fills.
```

### 8.5 Board B Operational States

```
     ┌─────────┐  reset_done   ┌──────┐
     │  RESET  │──────────────►│ IDLE │
     └─────────┘               └───┬──┘
         ▲                         │ link_up (quotes arriving)
         │                         ▼
         │ reset              ┌─────────┐
         ├────────────────────│ ARMED   │◄──────────────────┐
         │                    └────┬────┘                    │
         │                         │ trading_enable=1 & start│
         │                         ▼                         │
         │                    ┌──────────┐  stop or disable  │
         ├────────────────────│ TRADING  │──────────────────►│
         │                    └────┬─────┘                   │
         │                         │ pnl < -max_loss
         │                         ▼
         │                    ┌──────────┐
         └────────────────────│ HALTED   │
                              └──────────┘

IDLE:    No quotes received yet. Waiting for link.
ARMED:   Quotes flowing. Features computed. Orders NOT sent (observing).
TRADING: Full pipeline active. Orders generated and sent.
HALTED:  Max loss breached. Orders suppressed. Must reset to resume.
```

---

## 9. Vivado Build Strategy

### 9.1 Block Design Structure (same for both boards)

```
┌────────────────────────────────────────────────────────┐
│  Vivado Block Design                                    │
│                                                         │
│  ┌──────────────────────────┐                          │
│  │  Zynq UltraScale+ PS IP  │                          │
│  │  - FCLK0 = 100 MHz       │                          │
│  │  - M_AXI_HPM0_LPD        │                          │
│  │  - UART enabled           │                          │
│  └────────────┬─────────────┘                          │
│               │ AXI Master                              │
│  ┌────────────▼─────────────┐                          │
│  │  Processor System Reset   │                          │
│  └────────────┬─────────────┘                          │
│               │                                         │
│  ┌────────────▼─────────────┐                          │
│  │  AXI Interconnect         │                          │
│  │  (1 master, 1 slave)      │                          │
│  └────────────┬─────────────┘                          │
│               │ S00_AXI                                 │
│  ┌────────────▼─────────────┐                          │
│  │  board_x_custom_ip        │ ◄── Our packaged IP     │
│  │  (AXI-Lite Slave)         │     containing ALL       │
│  │                           │     RTL modules          │
│  │  External ports:          │                          │
│  │   - pmoda[7:0]            │                          │
│  │   - pmodb[7:0]            │                          │
│  │   - sw[7:0]               │                          │
│  │   - btn[3:0]              │                          │
│  │   - led[7:0]              │                          │
│  │   - rgb[11:0]             │                          │
│  └───────────────────────────┘                          │
└────────────────────────────────────────────────────────┘
```

### 9.2 IP Packaging Approach

1. Use Vivado's **"Create and Package New IP"** wizard to generate an AXI4-Lite slave template
2. Vivado auto-generates the AXI handshake logic and register template
3. We edit the template to add our custom register read/write behavior
4. We instantiate all data-plane RTL modules inside the IP wrapper
5. External PMOD, LED, switch, button ports are exposed as top-level IP ports

### 9.3 PYNQ Overlay Packaging

The Vivado build produces two files needed by PYNQ:
- `.bit` -- the FPGA bitstream
- `.hwh` -- the hardware handoff (describes address map, IP blocks, etc.)

Both files are placed on the MicroSD card. PYNQ's `Overlay()` class loads them:

```python
from pynq import Overlay
ol = Overlay('board_a.bit')  # loads .bit and parses .hwh
```

---

## 10. Testing Plan

### 10.1 Unit Testing (Simulation)

Every module gets a self-checking testbench in SystemVerilog. Simulation tool: Vivado XSIM (included) or ModelSim.

| Module | Stimulus | Key Checks | Edge Cases |
|--------|----------|------------|------------|
| `lfsr32` | Provide seed, clock 100 cycles | Never outputs zero; sequence matches known polynomial output | seed=0 (prevented), seed=all-ones |
| `link_tx` | Feed 5 known frames | Correct nibble sequence (MSB first), valid high for exactly 64 core_clk cycles (32 beats x 2), gap between frames | Back-to-back frames, remote_ready deassert mid-idle |
| `link_rx` | Drive pins with known frame pattern | Assembled 128-bit frame matches expected value, frame_out_valid pulses once per frame | Glitch on valid (should not corrupt), valid staying low (no spurious frames) |
| `link_tx + link_rx` | Loopback: tx pins -> rx pins | 1000 random frames sent = 1000 identical frames received | Max rate (back-to-back), intermittent pauses, ready toggling |
| `market_sim` | Set CALM regime, run 100 quote cycles | Prices within expected range, spread > 0, seq_num incrementing, correct symbol round-robin | Regime change mid-run, all 4 regimes, symbol count = 1 |
| `exchange_lite` | BUY order at ask price | FILL with fill_price = ask_price, order_id echoed | BUY below ask (REJECT), SELL above bid (REJECT), exact boundary |
| `tx_arbiter` | Simultaneous quote + fill | Fill sent first | Only quotes, only fills, alternating |
| `msg_demux` | Mixed QUOTE + FILL frames | Quotes to quote port, fills to fill port | Unknown msg_type (discard + error increment) |
| `feature_compute` | Known bid/ask sequence | mid exact, EMA converges within tolerance | Large price jump, spread=0, alpha=0, alpha=65535 |
| `strategy_engine` | Deviation > threshold | Correct BUY/SELL signal, correct price/qty | At threshold exactly, deviation=0 |
| `risk_manager` | Position at limit | Order rejected, counter incremented | Position at max-1 (approve), rate at max (reject), PnL at -max_loss |
| `order_manager` | Approved signal | Correct ORDER frame, order_id incrementing, timestamp captured | Rapid back-to-back (ID wrap), no signal (no output) |
| `position_tracker` | BUY fill then SELL fill | Position and cash correct | Large qty, alternating buy/sell back to zero, signed overflow |
| `latency_histogram` | Fills with known ts_echo values | Correct bin incremented, min/max updated | latency=0, latency > max bin (overflow bin), timestamp wrap |

### 10.2 Integration Testing (Simulation)

| Test | Setup | Duration | Pass Criteria |
|------|-------|----------|---------------|
| Board A standalone | market_sim + exchange_lite looped (orders from a stimulus module) | 10K quotes | Counters consistent, no FIFO overflow, fills match expected |
| Board B standalone | Synthetic quote generator replaces link_rx | 5K quotes | Orders match expected strategy, positions track correctly |
| Full system | Board A top + Board B top, behavioral link (wire model) | 50K quotes, all 4 regimes | Zero frame loss, PnL matches golden model, histogram non-empty |

### 10.3 Hardware Bring-Up (9 phases)

Each phase is gated -- do not proceed to the next until the current phase passes.

| Phase | Goal | Method | Pass Criteria |
|-------|------|--------|---------------|
| **1: LED Blinker** | Verify Vivado flow + PYNQ boot | Build minimal overlay with clock divider -> LED toggle | LED blinks at expected rate |
| **2: AXI-Lite Smoke** | Verify PS <-> PL register access | Add AXI-Lite slave with 1 R/W register. Read/write from Python MMIO. | Write 0xDEADBEEF, read back 0xDEADBEEF |
| **3: Board A Standalone** | Verify market_sim + exchange | Load full Board A overlay. Check counters via MMIO. Optional: ILA on quote frames. | quotes_sent incrementing, correct regime behavior |
| **4: Board B Standalone** | Verify trader pipeline | Load Board B overlay with internal synthetic quote generator (hardcoded). | orders_sent > 0, position != 0, histogram populated |
| **5: Link Smoke** | Verify PMOD data transfer | Board A sends incrementing counter frames. Board B checks sequence. | rx_frame_count = tx_frame_count, zero sequence errors |
| **6: One-Way Quotes** | Verify quote reception | Board A sends real quotes. Board B receives and updates quote_book. | Quote counters match on both boards |
| **7: Closed Loop** | Verify full trading cycle | Enable orders on Board B. Fills return from Board A. | fill_count > 0, position != 0, PnL updating |
| **8: Dashboard** | Verify laptop telemetry | Connect USB-UART. Run telemetry_server.py + dashboard.py. | All 8 panels updating with plausible data |
| **9: Stress Test** | Verify stability under all regimes | Cycle CALM -> VOLATILE -> BURST -> ADVERSARIAL, 10 min each. | link_errors = 0, no FIFO overflow, stable operation |

### 10.4 Stress Testing Protocol

| Regime | Duration | Metrics to Watch | Expected Behavior |
|--------|----------|-----------------|-------------------|
| CALM | 10 min | ~100K qps, steady PnL, few risk rejects | Stable, predictable |
| VOLATILE | 10 min | Same throughput, wider PnL swings, more rejects | PnL oscillates more |
| BURST | 10 min | ~1M+ qps, FIFO fill levels, no overflow | Throughput gauge maxes, FIFO fill < 75% |
| ADVERSARIAL | 10 min | High risk rejects, rapid PnL changes, position limits hit | Risk system heavily engaged |
| RAPID SWITCH | 10 min | Switch regime every 30 sec | No glitches on transitions |

### 10.5 Acceptance Criteria Matrix

| Criterion | Target | How Verified |
|-----------|--------|-------------|
| Closed loop runs | Quotes -> Orders -> Fills cycle completes | All counters incrementing on dashboard |
| Determinism | 0 cycles pipeline jitter | Histogram shows single-bin concentration |
| Throughput | >= 1M quotes/sec (BURST) | Dashboard throughput gauge |
| Latency | < 3 us round-trip typical | Histogram p99 |
| Reliability | 0 frames lost in 10 min | link_error_count = 0 |
| Risk enforcement | Position never exceeds limit | max(abs(position)) <= max_position |
| Telemetry | 8 panels update at >= 10 Hz | Visual confirmation |
| Stress | Zero errors across all 4 regimes | link_error_count = 0 after full suite |

---

## 11. Risk Analysis

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| PMOD signal integrity at 50 MHz | Low | High | Manual confirms 50 MHz support. Use LVCMOS33, short cables. If issues: reduce to 25 MHz (halves throughput, still sufficient). |
| Mesochronous sampling metastability | Low | High | 2-FF synchronizers on all inputs. 50 MHz data rate with 100 MHz sampling gives 2x oversampling margin. Fallback: add forwarded clock. |
| AXI-Lite integration issues | Medium | Medium | Use Vivado IP wizard template. Test Phase 2 early. |
| Fixed-point overflow in PnL | Low | Medium | 48-bit cash accumulator. Add saturation logic. |
| FIFO overflow under BURST | Low | High | 64-deep FIFOs + backpressure. BURST rate (~1.5M fps) approaches link max; monitor fill levels. |
| PYNQ overlay packaging errors | Medium | Medium | Follow PYNQ docs. Test Phase 1-2 early. |
| Time pressure (2 people, 10 weeks) | High | High | Strict phase-gated milestones. Cut stretch goals early if behind. |
| Board thermal issues under sustained load | Low | Medium | Manual notes passive heatsink included. Design uses < 10% resources, so power is low. |

---

## 12. Bill of Materials

### Required

| Item | Qty | Purpose |
|------|-----|---------|
| AUP-ZU3 board | 2 | Main compute platforms |
| MicroSD card (32GB, UHS-I) | 2 | PYNQ boot image + scripts |
| USB-C PD power supply (9V/3A+) | 2 | Board power (must support USB-C PD negotiation) |
| USB-C cable (data, for PROG UART port) | 2 | JTAG programming + UART serial to laptop |
| Standard 12-pin PMOD ribbon cable (2x6, 2.54mm) | 2 | Board-to-board link (one per direction) |

### Optional

| Item | Qty | Purpose |
|------|-----|---------|
| Female-female jumper wires (2.54mm) | 20 | Backup if PMOD cables have issues |
| External monitor + adapter | 1 | Demo display for laptop dashboard |
| USB-C to Ethernet adapter | 2 | Stretch: alternative telemetry path |

---

## Appendix A: XDC Constraints

**These constraints are identical for Board A and Board B** (same physical board). The RTL controls pin direction via the top-level port declarations (input vs output).

```tcl
## =============================================================================
## AUP-ZU3 Pin Constraints for HFT Capstone
## =============================================================================

## -- PMOD A (A->B link on Board A, B receives on Board B) --------------------
## IOSTANDARD: LVCMOS33
set_property -dict {PACKAGE_PIN J12 IOSTANDARD LVCMOS33} [get_ports {pmoda[0]}]
set_property -dict {PACKAGE_PIN H12 IOSTANDARD LVCMOS33} [get_ports {pmoda[1]}]
set_property -dict {PACKAGE_PIN H11 IOSTANDARD LVCMOS33} [get_ports {pmoda[2]}]
set_property -dict {PACKAGE_PIN G10 IOSTANDARD LVCMOS33} [get_ports {pmoda[3]}]
set_property -dict {PACKAGE_PIN K13 IOSTANDARD LVCMOS33} [get_ports {pmoda[4]}]
set_property -dict {PACKAGE_PIN K12 IOSTANDARD LVCMOS33} [get_ports {pmoda[5]}]
set_property -dict {PACKAGE_PIN J11 IOSTANDARD LVCMOS33} [get_ports {pmoda[6]}]
set_property -dict {PACKAGE_PIN J10 IOSTANDARD LVCMOS33} [get_ports {pmoda[7]}]

## -- PMOD B (B->A link on Board B, A receives on Board A) --------------------
## IOSTANDARD: LVCMOS33
set_property -dict {PACKAGE_PIN E12 IOSTANDARD LVCMOS33} [get_ports {pmodb[0]}]
set_property -dict {PACKAGE_PIN D11 IOSTANDARD LVCMOS33} [get_ports {pmodb[1]}]
set_property -dict {PACKAGE_PIN B11 IOSTANDARD LVCMOS33} [get_ports {pmodb[2]}]
set_property -dict {PACKAGE_PIN A10 IOSTANDARD LVCMOS33} [get_ports {pmodb[3]}]
set_property -dict {PACKAGE_PIN C11 IOSTANDARD LVCMOS33} [get_ports {pmodb[4]}]
set_property -dict {PACKAGE_PIN B10 IOSTANDARD LVCMOS33} [get_ports {pmodb[5]}]
set_property -dict {PACKAGE_PIN A12 IOSTANDARD LVCMOS33} [get_ports {pmodb[6]}]
set_property -dict {PACKAGE_PIN A11 IOSTANDARD LVCMOS33} [get_ports {pmodb[7]}]

## -- Switches (directly readable, directly active-high) ----------------------
## IOSTANDARD: LVCMOS12
set_property -dict {PACKAGE_PIN AB1 IOSTANDARD LVCMOS12} [get_ports {sw[0]}]
set_property -dict {PACKAGE_PIN AF1 IOSTANDARD LVCMOS12} [get_ports {sw[1]}]
set_property -dict {PACKAGE_PIN AE3 IOSTANDARD LVCMOS12} [get_ports {sw[2]}]
set_property -dict {PACKAGE_PIN AC2 IOSTANDARD LVCMOS12} [get_ports {sw[3]}]
set_property -dict {PACKAGE_PIN AC1 IOSTANDARD LVCMOS12} [get_ports {sw[4]}]
set_property -dict {PACKAGE_PIN AD6 IOSTANDARD LVCMOS12} [get_ports {sw[5]}]
set_property -dict {PACKAGE_PIN AD1 IOSTANDARD LVCMOS12} [get_ports {sw[6]}]
set_property -dict {PACKAGE_PIN AD2 IOSTANDARD LVCMOS12} [get_ports {sw[7]}]

## -- Pushbuttons (active-high, directly active-high) -------------------------
## IOSTANDARD: LVCMOS33
set_property -dict {PACKAGE_PIN AB6 IOSTANDARD LVCMOS33} [get_ports {btn[0]}]
set_property -dict {PACKAGE_PIN AB7 IOSTANDARD LVCMOS33} [get_ports {btn[1]}]
set_property -dict {PACKAGE_PIN AB2 IOSTANDARD LVCMOS33} [get_ports {btn[2]}]
set_property -dict {PACKAGE_PIN AC6 IOSTANDARD LVCMOS33} [get_ports {btn[3]}]

## -- White LEDs (directly active-high) ---------------------------------------
## IOSTANDARD: LVCMOS12
set_property -dict {PACKAGE_PIN AF5 IOSTANDARD LVCMOS12} [get_ports {led[0]}]
set_property -dict {PACKAGE_PIN AE7 IOSTANDARD LVCMOS12} [get_ports {led[1]}]
set_property -dict {PACKAGE_PIN AH2 IOSTANDARD LVCMOS12} [get_ports {led[2]}]
set_property -dict {PACKAGE_PIN AE5 IOSTANDARD LVCMOS12} [get_ports {led[3]}]
set_property -dict {PACKAGE_PIN AH1 IOSTANDARD LVCMOS12} [get_ports {led[4]}]
set_property -dict {PACKAGE_PIN AE4 IOSTANDARD LVCMOS12} [get_ports {led[5]}]
set_property -dict {PACKAGE_PIN AG1 IOSTANDARD LVCMOS12} [get_ports {led[6]}]
set_property -dict {PACKAGE_PIN AF2 IOSTANDARD LVCMOS12} [get_ports {led[7]}]

## -- RGB LEDs (accent, active-high) ------------------------------------------
## IOSTANDARD: LVCMOS12
set_property -dict {PACKAGE_PIN AD7 IOSTANDARD LVCMOS12} [get_ports {rgb0_r}]
set_property -dict {PACKAGE_PIN AD9 IOSTANDARD LVCMOS12} [get_ports {rgb0_g}]
set_property -dict {PACKAGE_PIN AE9 IOSTANDARD LVCMOS12} [get_ports {rgb0_b}]
set_property -dict {PACKAGE_PIN AG9 IOSTANDARD LVCMOS12} [get_ports {rgb1_r}]
set_property -dict {PACKAGE_PIN AE8 IOSTANDARD LVCMOS12} [get_ports {rgb1_g}]
set_property -dict {PACKAGE_PIN AF8 IOSTANDARD LVCMOS12} [get_ports {rgb1_b}]

## -- Timing (PMOD link is <50 MHz, treat as slow I/O) -----------------------
set_false_path -to [get_ports {pmoda[*]}]
set_false_path -from [get_ports {pmoda[*]}]
set_false_path -to [get_ports {pmodb[*]}]
set_false_path -from [get_ports {pmodb[*]}]
set_false_path -from [get_ports {sw[*]}]
set_false_path -from [get_ports {btn[*]}]
set_false_path -to [get_ports {led[*]}]
set_false_path -to [get_ports {rgb*}]
```

**Note on SW[5] pin (AD6)**: This pin assignment was read from the reference manual pinout table image. Verify against the official board XDC file from Real Digital if available.

---

## Appendix B: Message Format Bit Maps

All messages are exactly **128 bits (16 bytes)**.

### QUOTE (Board A -> Board B) -- msg_type = 4'h1

```
Bit Range   Width  Field         Description
---------   -----  -----------   -------------------------------------------
[127:124]    4     msg_type      4'h1 = QUOTE
[123:116]    8     symbol_id     Instrument index (0..3 for demo)
[115:114]    2     regime        00=CALM, 01=VOLATILE, 10=BURST, 11=ADVERSARIAL
[113:112]    2     reserved      2'b00
[111:80]    32     bid_price     Best bid (Q16.16)
[79:48]     32     ask_price     Best ask (Q16.16)
[47:32]     16     bid_size      Shares at bid
[31:16]     16     ask_size      Shares at ask
[15:0]      16     seq_num       Monotonic counter (per-symbol)
```

### ORDER (Board B -> Board A) -- msg_type = 4'h2

```
Bit Range   Width  Field         Description
---------   -----  -----------   -------------------------------------------
[127:124]    4     msg_type      4'h2 = ORDER
[123:116]    8     symbol_id     Instrument index
[115]        1     side          0=BUY, 1=SELL
[114:112]    3     reserved      3'b000
[111:80]    32     limit_price   Max buy / min sell price (Q16.16)
[79:64]     16     quantity      Shares to trade
[63:48]     16     order_id      Wrapping counter, assigned by Board B
[47:32]     16     timestamp     cycle_counter[15:0] at ORDER send time
[31:0]      32     reserved      32'h0
```

### FILL (Board A -> Board B) -- msg_type = 4'h3

```
Bit Range   Width  Field         Description
---------   -----  -----------   -------------------------------------------
[127:124]    4     msg_type      4'h3 = FILL
[123:116]    8     symbol_id     Instrument index
[115]        1     side          Echoed from ORDER
[114:112]    3     status        000=FILLED, 001=REJECTED
[111:80]    32     fill_price    Execution price (Q16.16), 0 if rejected
[79:64]     16     fill_qty      Shares filled, 0 if rejected
[63:48]     16     order_id      Echoed from ORDER
[47:32]     16     ts_echo       Echoed timestamp (for round-trip latency)
[31:0]      32     reserved      32'h0
```

---

## Appendix C: AXI-Lite Register Maps

### Board A Registers (base address assigned by Vivado, placeholder: 0x4000_0000)

| Offset | R/W | Width | Name | Description |
|--------|-----|-------|------|-------------|
| 0x00 | RW | 32 | CTRL | [0]=start, [1]=reset, [3:2]=regime (if sw_override=0) |
| 0x04 | RW | 32 | QUOTE_INTERVAL | Cycles between quote rounds |
| 0x08 | RW | 32 | NUM_SYMBOLS | Active symbols (1..4) |
| 0x0C | RW | 32 | LFSR_SEED | 32-bit PRNG seed |
| 0x10 | RW | 32 | SYM0_INIT_MID | Symbol 0 initial mid price (Q16.16) |
| 0x14 | RW | 32 | SYM0_INIT_SPREAD | Symbol 0 initial spread (Q16.16) |
| 0x18 | RW | 32 | SYM1_INIT_MID | Symbol 1 initial mid price |
| 0x1C | RW | 32 | SYM1_INIT_SPREAD | Symbol 1 initial spread |
| 0x20 | RW | 32 | SYM2_INIT_MID | Symbol 2 |
| 0x24 | RW | 32 | SYM2_INIT_SPREAD | Symbol 2 |
| 0x28 | RW | 32 | SYM3_INIT_MID | Symbol 3 |
| 0x2C | RW | 32 | SYM3_INIT_SPREAD | Symbol 3 |
| 0x40 | RO | 32 | STATUS | [0]=running, [1]=link_up |
| 0x44 | RO | 32 | QUOTES_SENT | Counter |
| 0x48 | RO | 32 | ORDERS_RCVD | Counter |
| 0x4C | RO | 32 | FILLS_SENT | Counter |
| 0x50 | RO | 32 | REJECTS_SENT | Counter |
| 0x54 | RO | 32 | LINK_ERRORS | Counter |

### Board B Registers (base address assigned by Vivado, placeholder: 0x4000_0000)

| Offset | R/W | Width | Name | Description |
|--------|-----|-------|------|-------------|
| 0x00 | RW | 32 | CTRL | [0]=start, [1]=reset |
| 0x04 | RW | 32 | STRATEGY_SEL | 0=mean-reversion (V1 only) |
| 0x08 | RW | 32 | THRESHOLD | Strategy threshold (Q16.16) |
| 0x0C | RW | 32 | EMA_ALPHA | EMA smoothing (Q0.16, e.g. 6554=0.1) |
| 0x10 | RW | 32 | BASE_QTY | Shares per trade |
| 0x14 | RW | 32 | MAX_POSITION | Risk: max abs position per symbol |
| 0x18 | RW | 32 | MAX_ORDER_RATE | Risk: max orders per window |
| 0x1C | RW | 32 | MAX_LOSS | Risk: loss halt threshold (Q16.16) |
| 0x40 | RO | 32 | STATUS | [0]=running, [1]=link_up, [2]=risk_halt |
| 0x44 | RO | 32 | QUOTES_RCVD | Counter |
| 0x48 | RO | 32 | ORDERS_SENT | Counter |
| 0x4C | RO | 32 | FILLS_RCVD | Counter |
| 0x50 | RO | 32 | RISK_REJECTS | Counter |
| 0x54 | RO | 32 | LINK_ERRORS | Counter |
| 0x58 | RO | 32 | POS_SYM0 | Position symbol 0 (signed 16-bit, sign-extended) |
| 0x5C | RO | 32 | POS_SYM1 | Position symbol 1 |
| 0x60 | RO | 32 | POS_SYM2 | Position symbol 2 |
| 0x64 | RO | 32 | POS_SYM3 | Position symbol 3 |
| 0x68 | RO | 32 | CASH_LO | Cash accumulator bits [31:0] |
| 0x6C | RO | 32 | CASH_HI | Cash accumulator bits [47:32] (signed) |
| 0x80 | RO | 32 | HIST_BIN_0 | Latency histogram bin 0 (0--31 cycles) |
| 0x84 | RO | 32 | HIST_BIN_1 | Bin 1 (32--63 cycles) |
| ... | ... | ... | ... | ... |
| 0xBC | RO | 32 | HIST_BIN_15 | Bin 15 (>= 480 cycles, overflow) |
| 0xC0 | RO | 32 | LAT_MIN | Min latency in cycles |
| 0xC4 | RO | 32 | LAT_MAX | Max latency in cycles |
| 0xC8 | RO | 32 | LAT_SUM_LO | Sum of latencies [31:0] |
| 0xCC | RO | 32 | LAT_SUM_HI | Sum of latencies [47:32] |
| 0xD0 | RO | 32 | LAT_COUNT | Number of latency samples |

---

## Appendix D: Market Simulator Detail

### LFSR Polynomial

32-bit Galois LFSR with polynomial `x^32 + x^22 + x^2 + x + 1` (feedback taps at bits 31, 21, 1, 0). Maximal length: 2^32 - 1 cycles before repeat. Seed loaded from AXI-Lite register on reset.

### Price Evolution

```
raw_step    = lfsr_out[4:0]                          // 5-bit unsigned [0..31]
signed_step = $signed({1'b0, raw_step}) - 16         // signed [-16..+15]
step_scaled = signed_step * step_size[regime]         // regime-dependent scale

mid_price[s] = mid_price[s] + step_scaled
spread[s]    = base_spread[regime] + lfsr_out[18:16]  // slight spread variation
bid_price[s] = mid_price[s] - (spread[s] >> 1)
ask_price[s] = mid_price[s] + (spread[s] >> 1)
```

### Regime Parameters

| Regime | step_size (Q16.16) | base_spread (Q16.16) | quote_interval (cycles) | Effective Rate |
|--------|-------------------|---------------------|------------------------|----------------|
| CALM | 0x0000_0100 (~0.004) | 0x0000_2000 (~0.125) | 4000 (40 us) | ~100K qps (4 sym) |
| VOLATILE | 0x0000_1000 (~0.0625) | 0x0000_8000 (~0.5) | 2000 (20 us) | ~200K qps |
| BURST | 0x0000_0100 (~0.004) | 0x0000_2000 (~0.125) | 4 (40 ns) | ~1.4M qps (near link max) |
| ADVERSARIAL | 0x0000_4000 (~0.25) | 0x0001_0000 (~1.0) | 400 (4 us) | ~1M qps |

Quote generation is round-robin across symbols: each timer tick produces one quote for the next symbol, cycling 0 -> 1 -> 2 -> 3 -> 0.

---

## Appendix E: Strategy and Risk Detail

### Mean-Reversion Strategy

```
mid       = (bid + ask) >> 1
deviation = mid - ema                    // signed Q16.16

if (deviation > +threshold):
    signal    = SELL
    order_price = bid_price              // sell at bid (aggressive)
    order_qty   = base_qty

else if (deviation < -threshold):
    signal    = BUY
    order_price = ask_price              // buy at ask (aggressive)
    order_qty   = base_qty

else:
    signal = NONE                        // no trade
```

### EMA Update (Fixed-Point)

```
// alpha: Q0.16 (0..65535 representing 0.0 to ~1.0)
// Default: alpha = 6554 (~0.1)

mult_new = alpha * mid_price;                    // 16 x 32 -> 48-bit
mult_old = (16'hFFFF - alpha + 1) * ema_old;     // 16 x 32 -> 48-bit
ema_new  = (mult_new + mult_old) >> 16;          // take [47:16] -> 32-bit
```

### Risk Checks (parallel, 1 cycle)

```
Check 1 -- Position limit:
  new_pos = position[symbol] + (side==BUY ? +qty : -qty)
  pass_1  = (abs(new_pos) <= max_position)

Check 2 -- Order rate:
  pass_2  = (orders_in_window < max_order_rate)
  // Window: shift register of depth window_cycles; increment on send, decrement on shift-out

Check 3 -- Loss limit:
  pass_3  = (total_pnl > -max_loss)
  // Uses signed comparison on Q16.16 value

approved = pass_1 & pass_2 & pass_3 & trading_enable & !risk_halt
```

---

## Appendix F: Directory Structure

```
ECE554_Capstone_HFT/
  docs/
    design_specification.md    <-- this document
  rtl/
    common/
      hft_pkg.sv
      lfsr32.sv
      debounce.sv
    link/
      link_tx.sv
      link_rx.sv
    board_a/
      market_sim.sv
      exchange_lite.sv
      tx_arbiter.sv
      board_a_axi_regs.sv
      board_a_ctrl.sv
      board_a_top.sv
    board_b/
      msg_demux.sv
      quote_book.sv
      feature_compute.sv
      strategy_engine.sv
      risk_manager.sv
      order_manager.sv
      position_tracker.sv
      latency_histogram.sv
      board_b_axi_regs.sv
      board_b_ctrl.sv
      board_b_top.sv
  tb/
    tb_link.sv
    tb_board_a.sv
    tb_board_b.sv
    tb_system.sv
  constraints/
    aup_zu3.xdc
  vivado/
    build_board_a.tcl
    build_board_b.tcl
  sw/
    board_a/
      config_exchange.py
    board_b/
      telemetry_server.py
    dashboard/
      dashboard.py
      requirements.txt
  README.md
```
