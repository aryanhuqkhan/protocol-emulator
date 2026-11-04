![](../../workflows/gds/badge.svg) ![](../../workflows/docs/badge.svg) ![](../../workflows/test/badge.svg) ![](../../workflows/fpga/badge.svg)

# Jane Street Programmable Protocol Emulator ASIC

A tiny, deterministic processor for implementing digital hardware protocols in firmware.

> [!IMPORTANT]
> This project is under development. Unless a result is explicitly marked **measured**, the architecture, protocol support, performance figures, and milestones in this document are proposals or targets—not completed features.

## Overview

Hardware protocols such as UART, SPI, and I²C are usually handled in one of two ways. Most microcontrollers include dedicated peripheral blocks: fixed digital circuits that handle one protocol each, with a baud-rate generator, shift register, and state machine wired in at design time. These are fast and precise, but they can only speak the protocols the chip designer chose to include. The alternative is bit-banging, where a general-purpose CPU toggles pins from software with carefully counted delays. That is flexible, but it consumes CPU time and becomes unreliable at higher speeds or when the processor is busy.

This protocol emulator takes a middle path: a tiny programmable processor with an instruction set designed for reading pins, writing pins, shifting data, counting cycles, and meeting exact timing. Protocols are implemented in firmware rather than fixed logic, so the same chip can run UART today and a different protocol tomorrow—even one created after fabrication—within the chip's timing, memory, electrical, and I/O limits. Because the processor is dedicated to pin-level work, it aims to preserve much of the timing precision of dedicated hardware without tying up a host CPU.

The design is inspired by programmable I/O engines such as the Raspberry Pi RP2040 PIO and deterministic real-time cores such as the TI PRU, while remaining small enough to explore as an open-source ASIC on Tiny Tapeout.

## Project goals

- Execute protocol firmware with deterministic, cycle-countable timing.
- Read, write, and change the direction of GPIO pins with low latency.
- Support compact loops, branches, shifts, sampling, and timed waits.
- Fit a useful processor, memories, and I/O control into a Tiny Tapeout design budget.
- Demonstrate representative implementations of UART, SPI, I²C, CAN, USB, and Ethernet, subject to physical-layer and clocking constraints.
- Provide reproducible cocotb tests and clearly separate measured results from design targets.
- Make the CPU, instruction set, firmware, and verification environment easy to study and extend.

## Non-goals and boundaries

The first version is not intended to replace every dedicated peripheral or to implement every physical layer directly. Protocols such as CAN, USB, and Ethernet require electrical signaling, analog behavior, clock recovery, line coding, or external transceivers that may be outside the scope of the Tiny Tapeout digital I/O interface. For those protocols, the initial goal is to implement and verify the digital link behavior that can be represented safely, and to document any required external PHY or transceiver.

This project also does not claim that all six protocols can run simultaneously. Firmware size, pin count, clock rate, and per-protocol timing will determine what can be supported in a given build.

## Status

| Area | Current status | Completion evidence |
|---|---|---|
| Protocol requirements | Planned/documented | Reviewed timing and electrical requirements |
| CPU architecture | Proposed | Synthesizable RTL and architecture tests |
| Instruction set | Proposed | Assembler plus instruction-level conformance tests |
| UART firmware | Planned | Loopback and error-injection tests |
| SPI firmware | Planned | Multi-mode controller/target tests |
| I²C firmware | Planned | Address, ACK/NACK, clock-stretching, and arbitration tests |
| CAN support | Research target | Bit-timing tests with an external transceiver model |
| USB support | Research target | Packet-level tests with documented PHY assumptions |
| Ethernet support | Research target | MAC/packet-level tests with documented PHY assumptions |
| ASIC implementation | Planned | LibreLane timing, area, DRC, and GDS results |
| FPGA prototype | Planned | Hardware test report and captured waveforms |

## Protocol requirements

The values below are design inputs, not claims of current support. Each firmware implementation must define its clock assumptions, acceptable timing error, pin mapping, external hardware, and test coverage.

### UART

UART is the simplest first implementation because it uses asynchronous serial data and does not require a shared clock line. There is no single universal UART protocol standard; voltage levels such as TTL/CMOS, RS-232, or RS-485 must be treated separately from the byte framing implemented by the emulator.

| Requirement | Initial target or consideration |
|---|---|
| Signals | TX and RX; optional RTS and CTS |
| Roles | Transmitter and receiver |
| Framing | Configurable data bits, optional parity, and configurable stop bits |
| Bit order | Least-significant bit first |
| Baud rates | Begin with 9,600 to 115,200 baud; characterize higher rates from measured clock timing |
| Receive sampling | Detect the start edge and sample near the center of each bit |
| Timing | Constant-cycle transmit loop; bounded receive-sampling error |
| Errors | Framing, parity, break, and receive-overrun detection where practical |
| Electrical layer | Direct CMOS-level testing or an external RS-232/RS-485 transceiver |
| Verification | TX/RX loopback, baud mismatch, parity error, framing error, and back-to-back frames |

### SPI

SPI is synchronous and has no single universal standard. Compatibility depends on clock polarity, clock phase, word width, bit order, chip-select behavior, and maximum clock rate.

| Requirement | Initial target or consideration |
|---|---|
| Signals | SCLK, COPI/MOSI, CIPO/MISO, and one or more chip-select signals |
| Roles | Controller first; peripheral/target after input timing is characterized |
| Modes | Mode 0 through Mode 3 (CPOL/CPHA combinations) |
| Word sizes | Start with 8 bits; allow firmware-defined widths |
| Bit order | MSB-first and LSB-first |
| Chip select | Programmable setup, hold, and inter-word behavior |
| Clock generation | Symmetric, cycle-counted high and low phases where possible |
| Full duplex | Sample input and update output on the configured edges |
| Frequency | Determine from instruction latency and post-layout clock frequency |
| Verification | All four modes, multiple word sizes, continuous transfers, and randomized data |

### I²C

I²C uses open-drain signaling, so the design must never actively drive SDA or SCL high. A released line is pulled high externally. Correct bidirectional-output-enable behavior is therefore a core requirement.

| Requirement | Initial target or consideration |
|---|---|
| Signals | Open-drain SDA and SCL with external pull-up resistors |
| Roles | Controller first; target mode as a later milestone |
| Addressing | 7-bit initially; consider 10-bit later |
| Speeds | Standard-mode (100 kbit/s) first, then Fast-mode (400 kbit/s) if timing permits |
| Bus events | START, repeated START, STOP, ACK, and NACK |
| Clock stretching | Release and sample SCL rather than assuming it rose immediately |
| Arbitration | Detect loss of arbitration when transmitting a released-high bit |
| Filtering | Define whether short spikes are rejected in hardware or firmware |
| Bus recovery | Optional sequence of SCL pulses followed by STOP |
| Verification | Address/data ACK and NACK, repeated START, stretching, arbitration, and stuck bus |

### CAN

CAN requires an external differential transceiver for normal bus operation. The programmable core would handle bit timing and the data-link behavior exposed on the transceiver's digital TX/RX interface. CAN FD is outside the first milestone unless resources and measured timing permit it.

| Requirement | Initial target or consideration |
|---|---|
| External hardware | CAN transceiver between logic-level TX/RX and CANH/CANL |
| Initial scope | Classical CAN data-link behavior; CAN FD deferred |
| Bit rates | Research 125 kbit/s, 250 kbit/s, 500 kbit/s, and 1 Mbit/s |
| Bit timing | Configurable synchronization, propagation, phase segments, and sample point |
| Bus behavior | Dominant/recessive monitoring and non-destructive arbitration |
| Framing | Standard 11-bit identifier first; extended 29-bit identifier later |
| Coding | Bit stuffing after five equal bits and de-stuffing on receive |
| Integrity | CRC, ACK slot, frame checks, and error flags |
| Fault handling | Error counters and bus-off behavior may require hardware assistance or extra memory |
| Verification | Arbitration, stuffing, CRC errors, ACK errors, resynchronization, and overload/error frames |

### USB

USB is a stretch target. Full USB compliance includes analog electrical requirements, line-state detection, precise signaling, packet timing, and host/device state machines. The first step is to determine which digital behaviors can be demonstrated using an external PHY or a carefully constrained low/full-speed interface.

| Requirement | Initial target or consideration |
|---|---|
| Initial scope | Packet-engine experiment; not a claim of compliant USB operation |
| Roles | Device-side proof of concept before host support |
| Speeds | Low-speed (1.5 Mbit/s) research target; full-speed (12 Mbit/s) only if measured timing permits |
| Physical layer | External USB PHY preferred; direct D+/D− signaling requires separate electrical validation |
| Coding | NRZI encode/decode and bit stuffing/de-stuffing |
| Packet fields | SYNC, PID, address/endpoint, payload, CRC5/CRC16, and EOP |
| Transactions | Token, data, and handshake packet sequencing |
| Timing | Response deadlines may require dedicated assists rather than firmware alone |
| State | Reset, idle, and basic enumeration research milestones |
| Verification | Known packet vectors, stuffing boundaries, CRC faults, malformed packets, and reset behavior |

### Ethernet

Ethernet is also a stretch target and requires an external PHY for real 10/100BASE-T signaling. A practical first scope is a small MAC-like packet engine connected to a digital PHY interface, or a reduced-rate educational model.

| Requirement | Initial target or consideration |
|---|---|
| Initial scope | MAC framing experiment; external PHY required for twisted-pair Ethernet |
| Candidate interface | Research MII/RMII-style digital interface subject to available pins and clocking |
| Data rate | 10 Mbit/s first if feasible; 100 Mbit/s likely needs hardware assistance |
| Transmit framing | Preamble, start-frame delimiter, destination/source, length/type, payload, padding, and FCS |
| Receive behavior | Frame detection, length checks, filtering, and FCS verification |
| Integrity | Ethernet CRC-32 generation and checking |
| Timing | Interpacket gap and PHY-interface setup/hold requirements |
| Duplex | Full-duplex first; collision detection/backoff deferred |
| Buffering | Streaming design or small FIFO because a complete frame may exceed local memory |
| Verification | Minimum/maximum frames, padding, CRC faults, back-to-back frames, and malformed lengths |

## Standards and references

The implementation should be checked against the latest applicable specification before compliance is claimed.

- UART electrical interfaces: TIA/EIA-232 and TIA/EIA-485, where applicable.
- SPI: device datasheets and the original Motorola SPI convention; SPI has no single universal protocol standard.
- I²C: NXP, *UM10204 I2C-bus specification and user manual*.
- CAN: Bosch, *CAN Specification 2.0*; ISO 11898 family for current requirements.
- USB: USB-IF, *Universal Serial Bus Specification* and relevant class specifications.
- Ethernet: IEEE 802.3 and the selected external PHY datasheet.
- Tiny Tapeout: [Tiny Tapeout documentation](https://tinytapeout.com/).
- Related programmable-I/O concepts: [RP2040 documentation](https://www.raspberrypi.com/documentation/microcontrollers/rp2040.html) and [TI PRU documentation](https://www.ti.com/tool/PRU-SWPKG).

## Proposed CPU architecture

The processor is intentionally small and deterministic. The following is a working architecture proposal to be tested against protocol timing before it is frozen.

### Datapath and state

| Component | Proposed design |
|---|---|
| Instruction width | Fixed 16-bit instructions for predictable fetch and decode |
| Data width | 16-bit datapath, with efficient 8-bit operations for serial protocols |
| Registers | Small general-purpose register file plus PC, flags, and cycle/timing state |
| Program storage | Small instruction memory initialized with protocol firmware |
| Data storage | Small scratch RAM or register-backed storage for state and short buffers |
| Execution | Single issue, in order, with documented cycle counts |
| GPIO | Direct input, output, and output-enable access through dedicated instructions |
| Timing | Cycle-counted delay/wait operations and optional absolute deadline counter |
| Host interface | Minimal command/status and firmware-loading path if area and pins permit |

### Candidate instruction groups

| Group | Candidate operations | Protocol purpose |
|---|---|---|
| Data movement | `MOV`, `LDI`, `LOAD`, `STORE` | State, constants, and buffers |
| Arithmetic/logic | `ADD`, `SUB`, `AND`, `OR`, `XOR`, `CMP` | Counters, masks, CRC support |
| Shifts | `SHL`, `SHR`, rotate, shift-through-carry | Serial transmit/receive and CRCs |
| Flow control | `JMP`, conditional branch, counted loop, call/return if affordable | Compact state machines |
| Pin input | Read all pins, read masked pins, wait for pin state/edge | Sampling and synchronization |
| Pin output | Write pins, set/clear masked pins, write-and-shift | Low-latency waveform generation |
| Direction | Set/clear output enable by mask | I²C open-drain and shared buses |
| Timing | Delay N cycles, wait until deadline, capture cycle count | Deterministic bit periods |
| Optional assists | CRC step, bit-stuff step, FIFO access | Higher-rate CAN/USB/Ethernet research |

The final encoding should favor the operations that appear in measured protocol kernels. Expensive general-purpose features should not displace timing-critical I/O operations unless benchmarks justify them.

## Seven architecture considerations

1. **Determinism.** Every instruction needs a documented latency. Branches, memory access, and pin operations must have predictable timing, or firmware cannot guarantee protocol edges.
2. **I/O latency.** Reading a pin, updating an output, and changing output enable should take as few cycles as possible. Read-modify-write hazards must be avoided for masked GPIO operations.
3. **Code density.** Program memory is expensive in a tiny ASIC. Immediate values, loops, branches, and combined shift/I/O operations should encode common protocol kernels compactly.
4. **Clock rate and timing resolution.** The clock must provide enough cycles per protocol bit for sampling, decisions, and edge placement. Post-layout maximum frequency—not RTL simulation alone—sets the real limit.
5. **Memory and buffering.** UART and SPI can stream with little storage, while USB and Ethernet packets can exceed a tiny local RAM. Interfaces need backpressure, streaming, or external buffering.
6. **Hardware assists versus programmability.** CRC, bit stuffing, FIFOs, and deadline timers can raise throughput, but every assist costs area and reduces generality. Add them only when profiling shows the CPU cannot meet a required deadline.
7. **Safe electrical behavior.** Firmware-visible output-enable control must support open-drain buses and reset to a safe, high-impedance state. External voltage-level, differential, and analog requirements remain the responsibility of appropriate transceivers or PHYs.

## Timing model

For a clock frequency `f_clk`, one processor cycle lasts:

```text
T_cycle = 1 / f_clk
```

If a firmware bit loop takes `C_bit` cycles, its nominal bit rate is:

```text
bit_rate = f_clk / C_bit
```

For receive sampling, the analysis must also include start-edge detection latency, branch-path variation, synchronizer delay, accumulated baud mismatch, and clock tolerance. Each firmware implementation should ship with a timing worksheet derived from its assembled instruction trace and checked against waveform measurements.

## Benchmark plan

No benchmark should be reported as achieved until the associated artifact is published. The table below defines what to measure.

| Benchmark | Metric | Method | Status |
|---|---|---|---|
| Core area | Standard-cell area and Tiny Tapeout tile utilization | LibreLane reports after hardening | Not measured |
| Maximum clock | Worst-corner post-route frequency and slack | Static timing analysis | Not measured |
| Instruction throughput | Instructions per cycle and branch penalties | Architecture tests plus waveform traces | Not measured |
| GPIO response | Cycles from sampled input change to output action | Directed cocotb timing test | Not measured |
| Delay accuracy | Programmed versus observed edge interval | Randomized timing tests | Not measured |
| Program size | Instructions/bytes for each protocol kernel | Assembler map output | Not measured |
| UART | Maximum verified baud rate and tolerated mismatch | Loopback with swept baud/error injection | Not measured |
| SPI | Maximum verified SCLK for each CPOL/CPHA mode | Controller/target model with randomized transfers | Not measured |
| I²C | Verified bus modes, stretching, and arbitration response | Behavioral bus model | Not measured |
| CAN | Maximum verified bit rate and sample-point error | Transceiver-level digital model | Not measured |
| USB | Verified packet rate and response latency | Packet-level PHY model | Not measured |
| Ethernet | Sustainable frame rate and interpacket gap | PHY/MAC stream model | Not measured |
| Power | Dynamic and leakage estimate for representative firmware | Post-layout activity and power analysis | Not measured |

### Initial engineering targets

These are starting targets for design trade-offs, not measured specifications.

| Item | Initial target |
|---|---|
| Core execution | One cycle for common ALU, shift, branch, and GPIO operations where practical |
| Timing variation | No data-dependent variation inside a selected protocol bit path |
| GPIO action | Input sample or masked output update in one instruction |
| Firmware | UART, SPI, and I²C kernels fit in the selected program memory together or as separate images |
| Verification | Every implemented instruction and protocol state has automated tests |
| Build | Reproducible lint, simulation, synthesis, timing, and documentation workflows |

## Verification strategy

Verification will be layered so failures can be localized quickly.

1. **Instruction-level tests** verify reset state, register operations, flags, branches, shifts, memory, GPIO, output enable, and exact instruction timing.
2. **Assembler tests** verify encoding, labels, range errors, and known instruction vectors.
3. **Firmware unit tests** run protocol kernels against behavioral peripherals and check every edge and sample point.
4. **Randomized protocol tests** vary payloads, clock ratios, pauses, errors, and boundary conditions.
5. **Gate-level tests** rerun critical timing-independent checks on the synthesized netlist.
6. **Post-layout analysis** records area, slack, clock limits, and any firmware timing assumptions affected by implementation.
7. **Hardware tests** compare captured FPGA or ASIC waveforms with the same reference models used in simulation.

The intended tool flow uses Verilog/SystemVerilog RTL, cocotb, and the Tiny Tapeout build environment. Tests should fail when timing is outside the documented tolerance, not merely when decoded bytes differ.

## Suggested repository layout

```text
.
├── docs/                 # Project and architecture documentation
├── firmware/             # Protocol programs and timing notes
├── src/                  # Synthesizable RTL
├── test/                 # cocotb tests and behavioral models
├── tools/                # Assembler and trace utilities
├── info.yaml             # Tiny Tapeout project metadata
└── README.md
```

## Development roadmap

### Phase 1: Requirements and timing

- Freeze Tiny Tapeout pin and clock assumptions.
- Turn each protocol table into cycle-level timing requirements.
- Identify which protocols require external transceivers or PHYs.
- Select representative devices and behavioral models for testing.

### Phase 2: CPU and toolchain

- Finalize the minimal instruction set after writing sample protocol kernels.
- Implement the processor, GPIO block, program memory, and scratch storage.
- Build a small assembler and instruction trace/disassembly tool.
- Verify every instruction and document exact cycle counts.

### Phase 3: Core protocol firmware

- Implement and verify UART transmit/receive.
- Implement and verify SPI modes 0–3.
- Implement and verify I²C controller operation, including open-drain behavior and clock stretching.
- Publish measured code size and maximum verified rates.

### Phase 4: Advanced protocol research

- Prototype Classical CAN bit timing, arbitration, stuffing, and CRC with an external transceiver model.
- Evaluate whether USB packet deadlines can be met in firmware or require dedicated assists.
- Evaluate a streaming Ethernet MAC experiment with an external PHY interface.
- Document resource or timing limits even when a protocol is not feasible.

### Phase 5: Physical implementation

- Run lint, synthesis, and Tiny Tapeout/LibreLane hardening.
- Close timing and record area, frequency, and power estimates.
- Run gate-level regression and review safe reset/output-enable behavior.
- Prototype on FPGA where practical, then prepare shuttle submission artifacts.

## Tiny Tapeout Verilog Project Template

- [Read the documentation for the project](docs/info.md)

### What is Tiny Tapeout?

Tiny Tapeout is an educational project that aims to make it easier and cheaper than ever to get your digital and analog designs manufactured on a real chip.

To learn more and get started, visit [https://tinytapeout.com](https://tinytapeout.com).

### Set up your Verilog project

1. Add your Verilog files to the `src` folder.
2. Edit [info.yaml](info.yaml) and update information about your project, paying special attention to the `source_files` and `top_module` properties. If you are upgrading an existing Tiny Tapeout project, use the [online info.yaml migration tool](https://tinytapeout.github.io/tt-yaml-upgrade-tool/).
3. Edit [docs/info.md](docs/info.md) and add a description of your project.
4. Adapt the testbench to your design. See [test/README.md](test/README.md) for more information.

The GitHub Action will automatically build the ASIC files using [LibreLane](https://www.zerotoasiccourse.com/terminology/librelane/).

### Enable GitHub Actions to build the results page

- [Enabling GitHub Pages](https://tinytapeout.com/faq/#my-github-action-is-failing-on-the-pages-part)

### Resources

- [FAQ](https://tinytapeout.com/faq/)
- [Digital design lessons](https://tinytapeout.com/digital_design/)
- [Learn how semiconductors work](https://tinytapeout.com/siliwiz/)
- [Join the community](https://tinytapeout.com/discord)
- [Build your design locally](https://www.tinytapeout.com/guides/local-hardening/)

### What next?

- [Submit your design to the next shuttle](https://app.tinytapeout.com/).
- Edit [this README](README.md) as implementation details and measured results become available.
- Share your project on your social network of choice:
  - LinkedIn [#tinytapeout](https://www.linkedin.com/search/results/content/?keywords=%23tinytapeout) [@TinyTapeout](https://www.linkedin.com/company/100708654/)
  - Mastodon [#tinytapeout](https://chaos.social/tags/tinytapeout) [@matthewvenn](https://chaos.social/@matthewvenn)
  - X (formerly Twitter) [#tinytapeout](https://twitter.com/hashtag/tinytapeout) [@tinytapeout](https://twitter.com/tinytapeout)
  - Bluesky [@tinytapeout.com](https://bsky.app/profile/tinytapeout.com)

## Contributing

Contributions are welcome, especially protocol timing research, compact firmware kernels, instruction-set trade-off analysis, cocotb models, and reproducible physical-design results. Please keep claims evidence-based: label targets as targets, link measured results to logs or reports, and describe the test conditions needed to reproduce them.

## License

Add the repository's selected open-source license here and include the corresponding `LICENSE` file before distribution.
