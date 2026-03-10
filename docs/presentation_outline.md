# Design Review Presentation Outline

**Audience**: Professor + 2 PhD students  
**Goal**: Demonstrate that the project is well-defined, technically sound, and achievable in 10 weeks  
**Estimated duration**: 15--20 minutes + Q&A

---

## Slide 1: Title

**Deterministic Dual-FPGA Low-Latency Trading Engine**

- ECE 554 Capstone
- Team: [names]
- Platform: 2x AMD AUP-ZU3 (Zynq UltraScale+ XCZU3EG)

---

## Slide 2: Problem Statement

**Talking points:**
- Software trading systems suffer from OS jitter (10--100 us unpredictable latency)
- Software-based latency measurement is unreliable (measured by the same OS causing the jitter)
- We want to demonstrate that FPGAs can process market data and make trading decisions with **zero jitter** and **cycle-accurate measurement**

**Key phrase**: "The OS is both the problem and the measuring instrument -- we eliminate both by moving everything to hardware."

---

## Slide 3: What We're Building (high level)

```
  Board A                    Board B                  Laptop
  ┌────────────┐  2 cables  ┌────────────┐  USB-UART ┌──────────┐
  │ Exchange   │◄──────────►│ Trader     │──────────►│Dashboard │
  │ + Market   │  (PMOD)    │ + Risk     │           │ (8 panels│
  │ Simulator  │            │ + Metrics  │           │  20 Hz)  │
  └────────────┘            └────────────┘           └──────────┘
```

- Board A generates synthetic market quotes, matches orders, returns fills
- Board B runs a trading strategy, enforces risk limits, measures latency in hardware
- Laptop displays real-time metrics (throughput, latency histogram, PnL, positions)

**Key phrase**: "A complete closed-loop trading system where every operation from quote reception to order generation happens in a deterministic, measurable hardware pipeline."

---

## Slide 4: Why This Is Hard / What Makes It Interesting

**Technical challenges we solve:**

1. **Custom board-to-board link** -- source-synchronous mesochronous 50 MHz streaming bus over PMOD, with CDC and hardware flow control
2. **Fixed-point DSP pipeline** -- EMA computation, price comparison, and PnL tracking in Q16.16 fixed-point using DSP48E2 slices
3. **Hardware latency measurement** -- cycle-counter timestamps embedded in messages, round-trip latency computed entirely in PL, 16-bin histogram updated every fill
4. **Real-time risk management** -- position, rate, and loss limits enforced in 1 clock cycle (10 ns) with zero possibility of software override
5. **Parameterized scalability** -- symbol count and link width are compile-time parameters; architecture scales without structural changes

**Anticipated question**: "How is this different from just two boards sending data back and forth?"  
**Answer**: "The data has meaning -- it's a realistic market protocol (quotes, orders, fills). The processing pipeline makes real decisions (when to trade, how much risk to take). And the measurement is hardware-accurate, not software-estimated. The communication is purposeful, not just a throughput test."

---

## Slide 5: Design Requirements (quantitative)

| Requirement | Target | Why It Matters |
|-------------|--------|----------------|
| Pipeline latency | <= 10 cycles (100 ns) | Proves deterministic hardware processing |
| Pipeline jitter | 0 cycles | Proves no OS/software in critical path |
| Link throughput | 1.47M+ frames/sec (4-bit), 2.78M+ (8-bit) | Handles burst stress regime |
| Frame loss | 0 | Hardware backpressure prevents data loss |
| Latency resolution | 10 ns (1 clock cycle) | Cycle-accurate measurement |
| Sustained operation | >= 10 min per regime | Proves reliability, not just a demo flicker |
| Risk check latency | 1 cycle (10 ns) | Risk is hardware-enforced, not advisory |

**Anticipated question**: "How do you validate the 0-jitter claim?"  
**Answer**: "The latency histogram. If the pipeline is truly deterministic, all latency samples fall in the same histogram bin. Any jitter would spread samples across bins. This is self-proving."

---

## Slide 6: System Architecture (block diagram)

Show the detailed Board A and Board B block diagrams from the design spec (Section 5.3 and 5.4).

**Talking points:**
- Board A has 3 data-plane modules: market_sim, exchange_lite, tx_arbiter
- Board B has 7 data-plane modules forming a pipeline: demux -> quote_book -> features -> strategy -> risk -> order_manager, plus fill processing (position + histogram)
- PS (ARM) is used ONLY for configuration and telemetry readout -- never in the data path
- All inter-module connections use valid/ready handshake (AXI-Stream style)

**Anticipated question**: "Why not use the PS for the trading logic?"  
**Answer**: "The PS runs Linux. Even with real-time patches, interrupt latency is 1--10 us and non-deterministic. Our PL pipeline is 80 ns and exactly the same every time. The PS is relegated to slow-path tasks: loading config at startup and reading counters at 20 Hz."

---

## Slide 7: Board-to-Board Link

**Talking points:**
- 2x standard PMOD cables (4-bit mode) or + 4 jumper wires (8-bit mode)
- 50 MHz effective data rate (within PMOD spec of ~50 MHz max)
- No forwarded clock -- mesochronous design: both boards sample with local 100 MHz, data output held for 20 ns, 2-FF synchronizers on receiver
- Valid/ready flow control prevents frame loss under any throughput
- Parameterized: LINK_DATA_W = 4 or 8, one constant change

**Anticipated question**: "Why not use Ethernet?"  
**Answer**: "Ethernet on this board goes through USB-to-Ethernet adapters and the Linux network stack. That puts the OS in the data path -- exactly what we're trying to avoid. PMOD is PL-to-PL, fully deterministic, no software involvement. We can always add Ethernet as a secondary telemetry channel."

**Anticipated question**: "Why 50 MHz and not faster?"  
**Answer**: "The AUP-ZU3 reference manual specifies PMOD signal speeds up to approximately 50 MHz. We respect the board spec. At 50 MHz with 4-bit data, we get 1.47M frames/sec -- that's over 10x what we need for any regime except max burst, and 8-bit mode doubles it."

---

## Slide 8: Message Format

Show the 128-bit frame layout (QUOTE, ORDER, FILL) from Appendix B of the design spec.

**Talking points:**
- Universal 128-bit frame for all message types
- 4-bit msg_type header distinguishes QUOTE / ORDER / FILL
- ORDER carries a 16-bit timestamp (cycle counter) that gets echoed back in FILL -- this is how we measure round-trip latency without cross-board clock synchronization
- Q16.16 fixed-point prices: 16 integer bits + 16 fractional bits

**Anticipated question**: "Why not variable-length messages?"  
**Answer**: "Fixed 128-bit frames simplify serialization, deserialization, and FIFO sizing. There's no compression benefit at this scale -- all three message types fit comfortably in 128 bits with room for reserved fields."

---

## Slide 9: Trading Strategy and Risk

**Talking points:**
- Mean-reversion strategy: compute EMA of mid-price, trade when price deviates beyond threshold
- EMA update: `ema_new = (alpha * mid + (1-alpha) * ema_old)` in Q16.16 using 2 DSP48E2 slices
- Risk manager: 3 parallel checks in 1 cycle -- position limit, order rate limit, loss halt
- All parameters configurable via AXI-Lite registers (threshold, alpha, limits)
- Strategy is intentionally simple -- the point is the hardware pipeline, not the alpha

**Anticipated question**: "Could you implement a more complex strategy?"  
**Answer**: "Yes. The pipeline has a clean valid/ready interface between stages. You could replace strategy_engine with a more complex module (even a tiny neural network) without changing anything upstream or downstream. It's a stretch goal."

---

## Slide 10: Latency Measurement

**Talking points:**
- Board B has a free-running 16-bit cycle counter
- When an ORDER is sent, the counter value is embedded as a timestamp
- When the FILL returns, the echoed timestamp is subtracted from the current counter
- Result goes into a 16-bin hardware histogram (32-cycle bins = 320 ns per bin)
- Also tracked: min, max, sum, count
- p50 and p99 computed on laptop from histogram (cumulative distribution)
- **The laptop never measures latency** -- it only reads pre-computed hardware counters

**Key phrase**: "Latency is measured by the hardware that processes the data, not by the software that displays it."

---

## Slide 11: Dashboard

Describe the 8 panels. If possible, show a mockup or wireframe.

**Talking points:**
- Plotly Dash web app on laptop (localhost:8050)
- Board B PS sends JSON at 20 Hz over USB-UART
- Throughput computed as counter deltas over time
- Regime changes are immediately visible: throughput jumps, latency shifts, risk rejects spike

**Anticipated question**: "Why not display directly on a monitor connected to the board?"  
**Answer**: "The board has Mini DisplayPort on the PS, but building a video pipeline in PL would be significant scope creep. The laptop dashboard is faster to implement, easier to polish, and the audience can see it from further away."

---

## Slide 12: Stress Regimes

| Regime | What It Does | What Dashboard Shows |
|--------|-------------|---------------------|
| CALM | Small price moves, moderate rate | Steady PnL, low rejects |
| VOLATILE | Large price swings, wider spreads | PnL oscillates wildly |
| BURST | Maximum quote rate (~1M+/sec) | Throughput gauge spikes |
| ADVERSARIAL | Rapid spread changes, large jumps | Risk rejects spike, possible halt |

**Talking point**: "We switch regimes live during the demo, and the audience watches the dashboard respond in real time. This proves the system isn't just running a canned demo -- it's reacting to changing conditions."

---

## Slide 13: Scalability

**Talking points:**
- `NUM_SYMBOLS`: default 4, scalable to 8/16/256 with one parameter change
- `LINK_DATA_W`: default 4-bit, upgradable to 8-bit with one parameter change + 4 jumper wires
- Resource utilization at 4 symbols: ~7% LUTs, ~3% FFs, ~2% BRAM, ~1% DSP
- Headroom for stretch goals: neural net strategy, deeper order book, auto-regime cycling

**Key phrase**: "We designed for extensibility, not just functionality."

---

## Slide 14: Testing Plan

Show the 9-phase bring-up plan from the design spec (Section 10.3).

**Talking points:**
- Phase-gated: each phase must pass before proceeding
- Phases 1--4 are single-board (can be done in parallel by 2 team members)
- Phase 5 is the first cross-board test (just counter frames, not real data)
- Phase 7 is the closed loop milestone
- Phase 9 is 40+ minutes of sustained stress testing

**Anticipated question**: "How do you verify correctness, not just 'it runs'?"  
**Answer**: "Three layers. First: self-checking simulation testbenches for every module. Second: counter comparison -- tx_count must equal rx_count, quotes_sent must equal quotes_received. Third: the latency histogram itself -- if frames were corrupted or lost, the histogram would show anomalies or the counters would mismatch."

---

## Slide 15: Timeline (10 weeks)

| Weeks | Person 1 (Link + Board A) | Person 2 (Board B + Dashboard) |
|-------|--------------------------|-------------------------------|
| 1--2 | hft_pkg, link_tx/rx, lfsr | msg_demux, quote_book, features |
| 3--4 | market_sim, exchange, arbiter | strategy, risk, order_manager |
| 5 | Board A top + testbenches | Board B top + testbenches (position, histogram) |
| 6 | Vivado build A, hardware Phase 2--3 | Vivado build B, hardware Phase 3--4 |
| 7 | Link bring-up (Phase 4--5) -- BOTH TOGETHER | |
| 8 | Close the loop (Phase 6--7) -- BOTH TOGETHER | |
| 9 | Stress testing (Phase 8--9) | Dashboard + telemetry script |
| 10 | Demo polish + presentation | Demo polish + presentation |

**Anticipated question**: "Is this realistic for 2 people?"  
**Answer**: "Yes. Each module is small (100--500 LUTs). The pipeline is linear, not a complex graph. The AXI-Lite template is auto-generated by Vivado. The hardest part is link bring-up in week 7, which is why both of us work on it together. We have 3 weeks of buffer between closed-loop (week 8) and demo (week 10)."

---

## Slide 16: Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| PMOD signal integrity | Manual confirms 50 MHz. Fallback: reduce to 25 MHz. |
| Mesochronous sampling | 2-FF synchronizers. Fallback: add forwarded clock. |
| AXI-Lite integration | Use Vivado auto-generated template. Test in Phase 2. |
| Time pressure | Phase-gated milestones. Cut stretch goals early if behind. |

---

## Slide 17: Summary / Ask

**Three things we want you to take away:**

1. This is a **complete, closed-loop system** with meaningful data semantics -- not just "two boards communicate"
2. Every metric is **measured in hardware** -- the numbers on the dashboard are real, not software estimates
3. The architecture is **parameterized and extensible** -- we can scale symbols and link width with single-constant changes

**What we need from you:**
- Approval to proceed with implementation
- Feedback on scope (too ambitious? too conservative?)
- Any concerns about the PMOD link approach

---

## Appendix: Anticipated Hard Questions

**Q: "What happens if one board crashes or resets mid-operation?"**  
A: The other board sees valid go low permanently. The link_rx module stays in IDLE. Counters stop incrementing. The dashboard shows the stall. We don't implement automatic recovery in V1 -- manual reset is required. This is realistic (real exchanges have halt procedures).

**Q: "Is this publishable / does it have research merit?"**  
A: The hardware latency measurement methodology (timestamp-echo in fill messages, entirely PL-based histogram) is a clean contribution. The parameterized exchange-trader testbed could be useful for other FPGA trading research. But the primary goal is an engineering demo, not a paper.

**Q: "Why not use high-speed serial transceivers (GTR) instead of PMOD?"**  
A: The AUP-ZU3's GTR transceivers are used by USB3 and DisplayPort. They're not routed to a general-purpose connector. PMOD is the only PL-connected inter-board interface available without custom PCB work. At 50 MHz x 4 bits (or 8 bits), PMOD provides 1.47--2.78M frames/sec -- more than sufficient.

**Q: "How do you handle clock drift between the two boards?"**  
A: Mesochronous design. Both boards run at ~100 MHz from independent oscillators (within ~50 ppm). Data is output at 50 MHz rate (held for 20 ns). The receiver's 2-FF synchronizer on the valid signal handles phase uncertainty. Over a 16-beat frame (320 ns at 8-bit width), cumulative drift is < 0.016 cycles -- negligible. The inter-frame gap absorbs any residual.

**Q: "What's your fallback if the PMOD link doesn't work?"**  
A: Three levels. First: reduce data rate to 25 MHz (double the hold time, halve throughput -- still 700K+ fps). Second: add a forwarded clock on a spare pin and use it via BUFG. Third: fall back to Ethernet via USB adapters for a functional demo (lower performance, OS in the path, but still demonstrates the concept).
