
# QEMU + cocotb + Corundum Co-Simulation

Run the **unmodified mqnic Linux driver** against the **Corundum RTL** in
cocotb simulation, with **QEMU** acting as the PCIe Root Complex.

PCIe TLPs are exchanged between QEMU and the cocotb simulation over a ZMQ
transport, enabling a real guest OS running a real driver to drive real
RTL — useful for pre-silicon driver development, HW/SW co-debug, and
driver-directed RTL verification.

## Architecture

```
┌─────────────┐   TLPs/ZMQ   ┌───────────┐   RQ/RC/CQ/CC   ┌─────────────┐
│    QEMU     │◄────────────►│  cocotb   │◄───────────────►│  Corundum   │
│  (Root      │              │ CosimPort │   AXI-Stream    │     RTL     │
│  Complex +  │              │ + PCIe    │                 │             │
│  unmodified │              │  device   │                 │  + EthMac   │
│  mqnic drv) │              │  model    │                 │  loopback   │
└─────────────┘              └───────────┘                 └─────────────┘
```

**TLP flow:**
- **Config:** QEMU → CosimPort → device model config space (enumeration)
- **MMIO:** QEMU BAR access → MemRd/MemWr TLP → CQ → Corundum registers
- **DMA:** Corundum RQ → device model → CosimPort → QEMU guest memory
- **MSI-X:** Corundum MSI-X MemWr → device model → QEMU interrupt delivery

## Components

| Component | Location | Notes |
|-----------|----------|-------|
| QEMU cosim PCIe device | [fork](https://github.com/syedsk/qemu) | Forwards config/MMIO as TLPs over ZMQ; GPL-2.0 |
| cocotbext-pcie CosimPort | [fork](https://github.com/syedsk/cocotbext-pcie/tree/qemu-cosim) | Based on v0.2.14 (Corundum-compatible) |
| Corundum testbench | [fork](https://github.com/syedsk/corundum/tree/qemu_cosim) | `fpga/common/tb/mqnic_core_pcie_us_qemu_cosim/` |

## Prerequisites

- Python 3.11
- cocotb **1.8.1** (not 2.x)
- cocotbext-pcie **0.2.14** (the fork with CosimPort)
- cocotbext-axi, cocotbext-eth, pyzmq
- Icarus Verilog (or Verilator)
- QEMU built with the `cocotb-pcie-endpoint` device
  ([fork](https://github.com/syedsk/qemu))
- An Ubuntu cloud image for the guest (e.g. `jammy-server-cloudimg-amd64.img`)

## Setup

### 1. Python environment

```bash
python3 -m venv corundum-sim-venv
source corundum-sim-venv/bin/activate

pip install "cocotb==1.8.1"
pip install cocotbext-axi cocotbext-eth pyzmq scapy

# Install the cosim-capable cocotbext-pcie
git clone https://github.com/syedsk/cocotbext-pcie.git
cd cocotbext-pcie && git checkout qemu-cosim
pip install -e . && cd ..
```

### 2. Build QEMU with the cosim device

```bash
../configure \
    --target-list=x86_64-softmmu \
    --enable-virtfs \
    --disable-gtk \
    --disable-sdl \
    --disable-gcrypt \
    --disable-curl \
    --disable-capstone \
    --disable-cap-ng \
    --enable-vnc \
    --enable-cap-ng \
    --disable-libusb \
    --disable-bzip2 \
    --enable-slirp

make -j$(nproc)
```

## Running the Co-Simulation

The cocotb simulation must start **first** (it binds the ZMQ sockets);
then QEMU connects.

### 3. Start the cocotb simulation

```bash
cd corundum/fpga/common/tb/mqnic_core_pcie_us_qemu_cosim
pytest -o log_cli=True --log-cli-level=INFO -s test_mqnic_qemu_cosim.py
```

Wait until the cosim server is listening (ports 5555 / 6000 / 6001 by
default). The simulation then idles, waiting for QEMU.

### 4. Launch QEMU

```bash
qemu-system-x86_64 \
    -machine q35,accel=kvm -cpu host -smp 4 -m 4G \
    -drive file=jammy-server-cloudimg-amd64.img,format=qcow2,if=virtio \
    -drive file=seed.img,format=raw,if=virtio \
    -device pcie-root-port,id=rp0,slot=0,bus=pcie.0 \
    -device cocotb-pcie-endpoint,bus=rp0,\
np_port=5555,push_port=6000,pull_port=6001,\
bar0_size=16777216,bar0_type=1,bar0_prefetch=on \
    -nographic -serial mon:stdio
```

BAR config for Corundum: BAR0 = 16 MiB, 64-bit (`bar0_type=1`),
prefetchable. MSI-X lives inside BAR0, so no separate MSI-X BAR.

### 5. Build and load the mqnic driver (in the guest)

```bash
sudo apt-get update
sudo apt-get install -y linux-headers-$(uname -r) build-essential

# Build from the same Corundum tree used for the RTL
cd corundum/modules/mqnic
make

# Load dependency, then the driver
sudo modprobe i2c-algo-bit
sudo insmod mqnic.ko

dmesg | grep mqnic     # should show enumeration + "Registered device mqnic0"
ip link                # a new interface (e.g. enp1s0np0) appears
```

### 6. Test traffic (loopback)

```bash
sudo ip addr add 192.168.100.1/24 dev enp1s0np0
sudo ip link set enp1s0np0 up

# Generate traffic (ARP/ping). The cocotb MAC loopback echoes TX -> RX.
ping -I enp1s0np0 -c 10 192.168.100.99

# Verify the datapath: both TX and RX counters should increment.
ip -s link show enp1s0np0
```

> "Destination Host Unreachable" is expected — there is no real peer to
> answer ARP. The proof of a working datapath is that **both TX and RX
> packet counters increment** in `ip -s link show`.

## Testbench Environment Variables

The cocotb testbench honors:

| Variable | Default | Description |
|----------|---------|-------------|
| `COSIM_HOST` | `localhost` | cosim server hostname |
| `COSIM_PCIE_WIDTH` | `512` | PCIe AXI-Stream data width for the build |
| `COSIM_LOG_LEVEL` | `INFO` | testbench log level |
| `COSIM_MAX_US` | `0` | max sim time in µs (0 = run forever) |

## QEMU Device BAR Configuration

The `cocotb-pcie-endpoint` device takes BAR layout via properties, so the
same binary works for different endpoints without recompiling:

| Property | Meaning |
|----------|---------|
| `barN_size` | BAR size in bytes (0 = unused) |
| `barN_type` | 0 = mem32, 1 = mem64, 2 = io |
| `barN_prefetch` | `on` / `off` (memory BARs) |

Example (Corundum): `bar0_size=16777216,bar0_type=1,bar0_prefetch=on`
A 64-bit BAR consumes two BAR slots, so leave the next `barN_size=0`.

## Troubleshooting

### Driver doesn't bind — device shows `1234:00ff`
cocotbext-pcie defaults `device_id` to `0x00ff`. Set it to match the
mqnic driver's PCI table (`0x1001`) in the testbench:
```python
self.dev.functions[0].device_id = 0x1001
```

### FW ID block reads `0x00000000` / probe fails with error `-5`
QEMU's BAR0 must **forward MMIO to the cosim**, not return local data, and
must be sized to match Corundum's Control BAR (16 MiB, 64-bit,
prefetchable):
```
bar0_size=16777216,bar0_type=1,bar0_prefetch=on
```

### `insmod: ERROR ... Unknown symbol i2c_bit_add_bus`
Load the dependency first:
```bash
sudo modprobe i2c-algo-bit
```
### `ModuleNotFoundError: No module named 'cocotb.log'`
cocotb version mismatch — use cocotb 1.8.x, not 2.x:
```bash
pip install "cocotb==1.8.1"
```

### `TypeError: object.__init__() takes exactly one argument`
cocotbext-pcie version mismatch with the testbench. Use cocotbext-pcie
**0.2.14** (the Corundum-compatible version / this fork).

### Simulation appears to "hang" after reset
Expected — the harness idles in a `while True` loop waiting for QEMU to
connect and drive TLPs. Start QEMU to proceed.

## Status

End-to-end verified: the unmodified mqnic driver enumerates the device,
brings up queues/scheduler/ports, registers a network interface, and moves
packets bidirectionally through the cocotb MAC loopback (TX and RX counters
increment).

## Known Limitations

- **Speed:** cycle-accurate RTL simulation is slow (~1 µs sim per second of
  wall time); suitable for functional/debug work, not performance testing.
- **Time sync:** QEMU and the simulation run in loosely-coupled time.
- **Version coupling:** requires cocotb 1.8.x and cocotbext-pcie 0.2.14.
- **Loopback only:** the MAC loopback echoes frames verbatim; there is no
  packet-answering peer, so ARP/ping do not receive real replies.
- Since QEMU (the Root Complex) is effectively an always-ready consumer, the
 `CosimPort` advertises **effectively unlimited credits** via `fc_state`.
  This keeps DMA flowing without modeling real backpressure.

## License

- QEMU device code (`qemu-device/`): **GPL-2.0** (links QEMU)
- Testbench and documentation: **BSD-2-Clause-Views** (matches Corundum and
  cocotbext-pcie)

## Acknowledgments

Built on [Corundum](https://github.com/corundum/corundum) and
[cocotbext-pcie](https://github.com/alexforencich/cocotbext-pcie) by
Alex Forencich. This project adds the QEMU co-simulation bridge enabling
unmodified drivers to run against the Corundum RTL.



