
# QEMU ↔ cocotb Multi-Bus Co-Simulation

A framework for running **unmodified guest drivers** against **RTL** in
cocotb simulation, with **QEMU** providing the host side across multiple
bus protocols (PCIe, I3C, SPI, Ethernet, …).

QEMU emulates the CPU + bus host; a thin cosim device forwards bus
transactions over a transport (ZMQ) to a cocotb simulation running the
real RTL via the matching cocotbext protocol model.

## Supported / In-Progress Targets

| Bus | Arch | QEMU Machine | cocotbext Repo | RTL DUT | Status | Docs |
|-----|------|--------------|----------------|---------|--------|------|
| PCIe | x86 | q35 | cocotbext-pcie | Corundum (mqnic NIC) | ✅ Working | [corundum.md](docs/corundum.md) |
| PCIe | x86 | q35 | cocotbext-pcie | NVMe (gzip compression) | ✅ Working | [nvme.md](docs/nvme.md) |
| I3C | riscv | virt | cocotbext-i3c + cocotbext-axi | CHIPS Alliance I3C target | 🚧 In progress | — |

✅ Working · 🚧 In progress · 📋 Planned

## How It Works

```
┌──────────────┐  bus txns   ┌───────────┐  protocol   ┌──────────────┐
│    QEMU      │◄───────────►│  cocotb   │◄───────────►│     RTL      │
│ (CPU + bus   │  over ZMQ   │  cosim    │   signals   │  (DUT via    │
│  host +      │             │  device + │             │  cocotbext   │
│  real driver)│             │  cocotbext│             │  model)      │
└──────────────┘             └───────────┘             └──────────────┘
```
## QEMU cocotb-pcie-endpoint

A generic QEMU PCIe device that bridges a guest-visible endpoint to an
external cocotb RTL simulation. It forwards all PCIe activity — config
space, MMIO, DMA, and MSI-X — as TLPs over ZMQ to the cocotb side, where the
RTL services them via a cocotbext-pcie device model. The same binary fronts
any PCIe endpoint (Corundum, NVMe, …); the endpoint's IDs and BAR layout
come from device properties + the cocotb device model.
See https://github.com/syedsk/qemu/

### Properties

| Property | Default | Description |
|----------|---------|-------------|
| `np_port` | 5555 | ZMQ port — non-posted requests / completions |
| `push_port` | 6000 | ZMQ port — outbound posted TLPs (QEMU → cocotb) |
| `pull_port` | 6001 | ZMQ port — inbound posted TLPs (cocotb → QEMU) |
| `barN_size` | 0 | BAR N size in bytes (0 = unused), N = 0..5 |
| `barN_type` | 0 | BAR N type: 0=mem32, 1=mem64, 2=io |
| `barN_prefetch` | false | BAR N prefetchable (memory BARs) |

A 64-bit BAR consumes two slots; leave the next `barN_size=0`. BAR sizes
must match the cocotb model's `configure_bar()`.

### Examples

Corundum: `bar0_size=16777216,bar0_type=1,bar0_prefetch=on`
NVMe:     `bar0_size=16384,bar0_type=0,bar1_size=16384,bar1_type=0`
