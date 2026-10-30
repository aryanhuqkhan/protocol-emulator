![](../../workflows/gds/badge.svg) ![](../../workflows/docs/badge.svg) ![](../../workflows/test/badge.svg) ![](../../workflows/fpga/badge.svg)
# Jane Street Protocol Emulator

Purpose:
Hardware protocols like UART, SPI, and I2C are usually handled in one of two ways. Most microcontrollers include dedicated peripheral blocks, fixed digital circuits that handle one protocol each, with a baud rate generator, shift register, and state machine wired in at design time. These are fast and precise, but they can only speak the protocols the chip designer chose to include. The alternative is bit-banging, where the CPU toggles pins from software with carefully counted delays. That is flexible, but it consumes CPU time and becomes unreliable at higher speeds or when the processor is busy.

A protocol emulator ASIC takes a middle path: a tiny programmable processor with an instruction set built for reading pins, writing pins, counting cycles, and hitting exact timing. Protocols are implemented in firmware rather than fixed logic, so the same chip can run UART today and a different protocol tomorrow, even one invented after the chip was fabricated, within its timing and I/O limits. Because the processor does nothing but pin-level work, it keeps the timing precision of dedicated hardware without tying up a main CPU.

This is a good design approach for several reasons. It gives flexibility without the usual cost in timing accuracy, so one small chip replaces many fixed blocks. It makes the hardware useful long after fabrication, which matters for an expensive, unchangeable ASIC. And it is a practical tool for hardware debugging and reverse engineering, where you often need to talk to, probe, or mimic unknown or custom devices. Existing designs like the RP2040’s PIO state machines and TI’s PRU cores show the idea works, and an open-source version would let others study, verify, and extend it.


# Tiny Tapeout Verilog Project Template

- [Read the documentation for project](docs/info.md)

## What is Tiny Tapeout?

Tiny Tapeout is an educational project that aims to make it easier and cheaper than ever to get your digital and analog designs manufactured on a real chip.

To learn more and get started, visit https://tinytapeout.com.

## Set up your Verilog project

1. Add your Verilog files to the `src` folder.
2. Edit the [info.yaml](info.yaml) and update information about your project, paying special attention to the `source_files` and `top_module` properties. If you are upgrading an existing Tiny Tapeout project, check out our [online info.yaml migration tool](https://tinytapeout.github.io/tt-yaml-upgrade-tool/).
3. Edit [docs/info.md](docs/info.md) and add a description of your project.
4. Adapt the testbench to your design. See [test/README.md](test/README.md) for more information.

The GitHub action will automatically build the ASIC files using [LibreLane](https://www.zerotoasiccourse.com/terminology/librelane/).

## Enable GitHub actions to build the results page

- [Enabling GitHub Pages](https://tinytapeout.com/faq/#my-github-action-is-failing-on-the-pages-part)

## Resources

- [FAQ](https://tinytapeout.com/faq/)
- [Digital design lessons](https://tinytapeout.com/digital_design/)
- [Learn how semiconductors work](https://tinytapeout.com/siliwiz/)
- [Join the community](https://tinytapeout.com/discord)
- [Build your design locally](https://www.tinytapeout.com/guides/local-hardening/)

## What next?

- [Submit your design to the next shuttle](https://app.tinytapeout.com/).
- Edit [this README](README.md) and explain your design, how it works, and how to test it.
- Share your project on your social network of choice:
  - LinkedIn [#tinytapeout](https://www.linkedin.com/search/results/content/?keywords=%23tinytapeout) [@TinyTapeout](https://www.linkedin.com/company/100708654/)
  - Mastodon [#tinytapeout](https://chaos.social/tags/tinytapeout) [@matthewvenn](https://chaos.social/@matthewvenn)
  - X (formerly Twitter) [#tinytapeout](https://twitter.com/hashtag/tinytapeout) [@tinytapeout](https://twitter.com/tinytapeout)
  - Bluesky [@tinytapeout.com](https://bsky.app/profile/tinytapeout.com)
