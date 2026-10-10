
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
