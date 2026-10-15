
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

## Example Run

### cocotb side (simulation)

```
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb                             Running on Icarus Verilog version 11.0 (stable)
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb                             Running tests with cocotb v1.7.2 from /proj/ipeng1/syeds/corundum-sim-venv/lib64/python3.11/site-packages/cocotb
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb                             Seeding Python random module with 1792083786
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.regression                  Found test test_mqnic_qemu_cosim.run_test_cosim
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.regression                  running run_test_cosim (1/1)
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00] Xilinx UltraScale+ PCIe hard IP core model
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00] cocotbext-pcie version 0.2.14
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00] Copyright (c) 2020 Alex Forencich
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00] https://github.com/alexforencich/cocotbext-pcie
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00] Incomplete configuration specified, attempting to select reasonable options
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00] Setting PCIe link width to x8
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00] Xilinx UltraScale+ PCIe hard IP core configuration:
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PCIe speed: gen 3
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PCIe link width: x8
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   User clock frequency: 250 MHz
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Alignment: dword
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable CQ straddling: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable CC straddling: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable RQ straddling: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable RC straddling: True
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable RC 4 TLP straddling: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF count: 1
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Max payload size: 1024
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable client tag: True
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable extended tag: True
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable parity: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable RX message interface: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable SR-IOV: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable extended configuration: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable PF0 MSI: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF0 MSI vector count: 32
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable PF1 MSI: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF1 MSI vector count: 1
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable PF2 MSI: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF2 MSI vector count: 1
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable PF3 MSI: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF3 MSI vector count: 1
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable PF0 MSIX: True
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF0 MSIX table size: 63
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF0 MSIX table BIR: 0
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF0 MSIX table offset: 0x00010000
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF0 MSIX PBA BIR: 0
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF0 MSIX PBA offset: 0x00018000
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable PF1 MSIX: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF1 MSIX table size: 0
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF1 MSIX table BIR: 0
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF1 MSIX table offset: 0x00000000
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF1 MSIX PBA BIR: 0
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF1 MSIX PBA offset: 0x00000000
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable PF2 MSIX: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF2 MSIX table size: 0
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF2 MSIX table BIR: 0
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF2 MSIX table offset: 0x00000000
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF2 MSIX PBA BIR: 0
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF2 MSIX PBA offset: 0x00000000
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   Enable PF3 MSIX: False
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF3 MSIX table size: 0
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF3 MSIX table BIR: 0
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF3 MSIX table offset: 0x00000000
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF3 MSIX PBA BIR: 0
INFO     cocotb:simulator.py:302      0.00ns INFO     ..cie.UltraScalePlusPcieDevice[00]   PF3 MSIX PBA offset: 0x00000000
INFO     cocotb:simulator.py:302 PCIe cosim listening on 5555
INFO     cocotb:simulator.py:302 PCIe cosim listening on 6000
INFO     cocotb:simulator.py:302 PCIe cosim listening on 6001
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_tx_inst.m_axis_tx      Ethernet MAC TX model
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_tx_inst.m_axis_tx      cocotbext-eth version 0.1.22
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_tx_inst.m_axis_tx      Copyright (c) 2020 Alex Forencich
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_tx_inst.m_axis_tx      https://github.com/alexforencich/cocotbext-eth
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_tx_inst.m_axis_tx      Reset de-asserted
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_tx_inst.m_axis_tx      Ethernet MAC TX model configuration
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_tx_inst.m_axis_tx        Byte size: 8 bits
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_tx_inst.m_axis_tx        Data width: 64 bits (8 bytes)
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_tx_inst.m_axis_tx        tkeep width: 8 bits
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_tx_inst.m_axis_tx        tuser width: 17 bits
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_tx_inst.m_axis_tx        ptp_time width: 96 bits
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_tx_inst.m_axis_tx      Reset de-asserted
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_rx_inst.s_axis_rx      Ethernet MAC RX model
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_rx_inst.s_axis_rx      cocotbext-eth version 0.1.22
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_rx_inst.s_axis_rx      Copyright (c) 2020 Alex Forencich
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_rx_inst.s_axis_rx      https://github.com/alexforencich/cocotbext-eth
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_rx_inst.s_axis_rx      Reset de-asserted
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_rx_inst.s_axis_rx      Ethernet MAC RX model configuration
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_rx_inst.s_axis_rx        Byte size: 8 bits
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_rx_inst.s_axis_rx        Data width: 64 bits (8 bytes)
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_rx_inst.s_axis_rx        tkeep width: 8 bits
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_rx_inst.s_axis_rx        tuser width: 97 bits
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_rx_inst.s_axis_rx        ptp_time width: 96 bits
INFO     cocotb:simulator.py:302      0.00ns INFO     cocotb.port_rx_inst.s_axis_rx      Reset de-asserted
INFO     cocotb:simulator.py:302      4.00ns INFO     cocotb.port_rx_inst.s_axis_rx      Reset asserted
INFO     cocotb:simulator.py:302      4.00ns INFO     cocotb.port_rx_inst.s_axis_rx      Reset asserted
INFO     cocotb:simulator.py:302      4.00ns INFO     cocotb.port_tx_inst.m_axis_tx      Reset asserted
INFO     cocotb:simulator.py:302      4.00ns INFO     cocotb.port_tx_inst.m_axis_tx      Reset asserted
INFO     cocotb:simulator.py:302    208.00ns INFO     cocotb.port_rx_inst.s_axis_rx      Reset de-asserted
INFO     cocotb:simulator.py:302    208.00ns INFO     cocotb.port_rx_inst.s_axis_rx      Reset de-asserted
INFO     cocotb:simulator.py:302    208.00ns INFO     cocotb.port_tx_inst.m_axis_tx      Reset de-asserted
INFO     cocotb:simulator.py:302    208.00ns INFO     cocotb.port_tx_inst.m_axis_tx      Reset de-asserted
INFO     cocotb:simulator.py:302  19150.00ns INFO     ..raScalePlusPcieFunction[00:00.0] Assigned PCIe ID 01:00.0
INFO     cocotb:simulator.py:302  19150.00ns INFO     ..raScalePlusPcieFunction[01:00.0] Config type 0 read, reg 0x000

...
INFO     cocotb:simulator.py:302 387020.00ns INFO     ..otb.mqnic_core_pcie_us.s_axis_cq TX CQ frame: UsPcieFrame(data=[0x00000004, 0x000000c0, 0x00000001, 0x00c000d1], byte_en=[0x0, 0x0, 0x0, 0x0], parity=[0xe, 0xf, 0xe, 0xf], first_be=0xf, last_be=0x0, discontinue=False, seq_num=0)
INFO     cocotb:simulator.py:302 387132.00ns INFO     ..otb.mqnic_core_pcie_us.m_axis_cc RX CC frame: UsPcieFrame(data=[0x00040004, 0x00000001, 0x000000d1, 0x00000100], byte_en=[], parity=[0x0, 0x0, 0x0, 0x0], first_be=0x0, last_be=0x0, discontinue=False, seq_num=0)
INFO     cocotb:simulator.py:302 387140.00ns INFO     ..raScalePlusPcieFunction[01:00.0] Config type 0 read, reg 0x018
INFO     cocotb:simulator.py:302 387150.00ns INFO     ..raScalePlusPcieFunction[01:00.0] Config type 0 read, reg 0x018
INFO     cocotb:simulator.py:302 387160.00ns INFO     ..raScalePlusPcieFunction[01:00.0] Config type 0 read, reg 0x018
INFO     cocotb:simulator.py:302 387170.00ns INFO     ..raScalePlusPcieFunction[01:00.0] Config type 0 write, reg 0x018 data 0xc03f0000
INFO     cocotb:simulator.py:302 387180.00ns INFO     ..raScalePlusPcieFunction[01:00.0] Config type 0 read, reg 0x018
INFO     cocotb:simulator.py:302 387190.00ns INFO     ..raScalePlusPcieFunction[01:00.0] Config type 0 read, reg 0x019
INFO     cocotb:simulator.py:302 387200.00ns INFO     ..raScalePlusPcieFunction[01:00.0] Config type 0 read, reg 0x018
INFO     cocotb:simulator.py:302 387200.00ns INFO     ..otb.mqnic_core_pcie_us.s_axis_cq TX CQ frame: UsPcieFrame(data=[0x0001000c, 0x000000c0, 0x00000801, 0x00c000d8, 0x00000001], byte_en=[0x0, 0x0, 0x0, 0x0, 0xf], parity=[0xb, 0xf, 0xc, 0xf, 0xe], first_be=0xf, last_be=0x0, discontinue=False, seq_num=0)
INFO     cocotb:simulator.py:302 387212.00ns INFO     ..otb.mqnic_core_pcie_us.s_axis_cq TX CQ frame: UsPcieFrame(data=[0x0001000c, 0x000000c0, 0x00000001, 0x00c00019], byte_en=[0x0, 0x0, 0x0, 0x0], parity=[0xb, 0xf, 0xe, 0xe], first_be=0xf, last_be=0x0, discontinue=False, seq_num=0)


```

### QEMU guest side (driver load)

```
ubuntu@ubuntu:~$ lspci -v
01:00.0 Non-VGA unclassified device: Device 1234:00ff
        Physical Slot: 0
        Flags: fast devsel
        Memory at c000000000 (64-bit, prefetchable) [size=16M]
        Capabilities: [40] Power Management version 3
        Capabilities: [60] MSI-X: Enable+ Count=64 Masked-
        Capabilities: [70] Express Endpoint, MSI 00
        
sudo modprobe i2c-algo-bit
sudo insmod mqnic.ko
[  115.160704] mqnic: loading out-of-tree module taints kernel.
[  115.161891] mqnic: module verification failed: signature and/or required key missing - tainting kernel
[  115.169249] mqnic 0000:01:00.0: mqnic PCI probe
[  115.169682] mqnic 0000:01:00.0:  Vendor: 0x1234
[  115.170038] mqnic 0000:01:00.0:  Device: 0x00ff
[  115.170412] mqnic 0000:01:00.0:  Subsystem vendor: 0x0000
[  115.170815] mqnic 0000:01:00.0:  Subsystem device: 0x0000
[  115.171250] mqnic 0000:01:00.0:  Class: 0x000000
[  115.171649] mqnic 0000:01:00.0:  PCI ID: 0000:01:00.0
[  115.193407] mqnic 0000:01:00.0:  Max payload size: 128 bytes
[  115.194423] mqnic 0000:01:00.0:  Max read request size: 512 bytes
[  115.195495] mqnic 0000:01:00.0:  Read completion boundary: 64 bytes
[  115.197447] mqnic 0000:01:00.0:  Link capability: gen 0 x0
[  115.198567] mqnic 0000:01:00.0:  Link status: gen 0 x0
[  115.199195] mqnic 0000:01:00.0:  Relaxed ordering: enabled
[  115.199866] mqnic 0000:01:00.0:  Phantom functions: disabled
[  115.201175] mqnic 0000:01:00.0:  Extended tags: enabled
[  115.201940] mqnic 0000:01:00.0:  No snoop: enabled
[  115.202868] mqnic 0000:01:00.0:  NUMA node: -1
[  115.203551] mqnic 0000:01:00.0:  PCI ID (bridge): 0000:00:02.0
[  115.204554] mqnic 0000:01:00.0:  Link capability (bridge): gen 4 x32
[  115.205861] mqnic 0000:01:00.0:  Link status (bridge): gen 4 x32
[  115.223080] mqnic 0000:01:00.0: 0.000 Gb/s available PCIe bandwidth (Unknown x255 link)
[  115.248667] mqnic 0000:01:00.0: Control BAR size: 16777216

[  121.236814] hrtimer: interrupt took 419799100 ns
[  128.649689] mqnic 0000:01:00.0: Configured 64 IRQs
[  129.042886] mqnic 0000:01:00.0: Device-level register blocks:
[  129.710577] mqnic 0000:01:00.0:  type 0xffffffff (v 0.0.1.0)
[  129.711387] mqnic 0000:01:00.0:  type 0x0000c007 (v 0.0.1.0)
[  129.713212] mqnic 0000:01:00.0:  type 0x0000c000 (v 0.0.1.0)
[  129.714728] mqnic 0000:01:00.0:  type 0x00000000 (v 0.0.0.0)
[  129.715904] mqnic 0000:01:00.0:  type 0x0000c006 (v 0.0.1.0)
[  129.718259] mqnic 0000:01:00.0:  type 0x0000c080 (v 0.0.2.0)
[  129.719341] mqnic 0000:01:00.0:  type 0x0000c008 (v 0.0.1.0)
[  130.112330] mqnic 0000:01:00.0: FPGA ID: 0xdeadbeef
[  130.113798] mqnic 0000:01:00.0: FW ID: 0x00000000
[  130.114432] mqnic 0000:01:00.0: FW version: 0.0.1.0
[  130.115044] mqnic 0000:01:00.0: Board ID: 0x12340000
[  130.115588] mqnic 0000:01:00.0: Board version: 1.0.0.0
[  130.117225] mqnic 0000:01:00.0: Build date: 1989-02-08 21:20:00 UTC (raw: 0x23f0af00)
[  130.118333] mqnic 0000:01:00.0: Git hash: dce357bf
[  130.118857] mqnic 0000:01:00.0: Release info: 00000000
[  130.561000] mqnic 0000:01:00.0: IF offset: 0x00000000
[  130.602693] mqnic 0000:01:00.0: IF count: 1
[  130.603400] mqnic 0000:01:00.0: IF stride: 0x01000000
[  130.603896] mqnic 0000:01:00.0: IF CSR offset: 0x00080000
[  130.605209] mqnic 0000:01:00.0: Unknown board ID, not performing any board-specific init
[  130.609070] mqnic 0000:01:00.0: registered PHC (index 1)
[  130.610035] mqnic 0000:01:00.0: Creating interface 0
[  131.574399] mqnic 0000:01:00.0: Interface-level register blocks:
[  131.575739] mqnic 0000:01:00.0:  type 0x0000c001 (v 0.0.4.0)
[  131.577612] mqnic 0000:01:00.0:  type 0x0000c010 (v 0.0.4.0)
[  131.578277] mqnic 0000:01:00.0:  type 0x0000c020 (v 0.0.4.0)
[  131.579058] mqnic 0000:01:00.0:  type 0x0000c030 (v 0.0.4.0)
[  131.580027] mqnic 0000:01:00.0:  type 0x0000c031 (v 0.0.4.0)
[  131.581944] mqnic 0000:01:00.0:  type 0x0000c090 (v 0.0.2.0)
[  131.583057] mqnic 0000:01:00.0:  type 0x0000c002 (v 0.0.2.0)
[  131.584394] mqnic 0000:01:00.0:  type 0x0000c004 (v 0.0.3.0)
[  131.588300] mqnic 0000:01:00.0: IF features: 0x00001f11
[  131.845714] mqnic 0000:01:00.0: Port count: 1
[  131.846476] mqnic 0000:01:00.0: Scheduler block count: 1
[  131.847083] mqnic 0000:01:00.0: Max TX MTU: 9214
[  131.847731] mqnic 0000:01:00.0: Max RX MTU: 9214
[  131.940379] mqnic 0000:01:00.0: EQ offset: 0x00180000
[  131.978516] mqnic 0000:01:00.0: EQ count: 64
[  131.979200] mqnic 0000:01:00.0: EQ stride: 0x00000010
[  132.091063] mqnic 0000:01:00.0: CQ offset: 0x00200000
[  132.092970] mqnic 0000:01:00.0: CQ count: 16384
[  132.093611] mqnic 0000:01:00.0: CQ stride: 0x00000010
[  132.096760] mqnic 0000:01:00.0: TXQ offset: 0x00400000
[  132.207943] mqnic 0000:01:00.0: TXQ count: 8192
[  132.209172] mqnic 0000:01:00.0: TXQ stride: 0x00000020
[  132.213105] mqnic 0000:01:00.0: RXQ offset: 0x00600000
[  132.321509] mqnic 0000:01:00.0: RXQ count: 256
[  132.322208] mqnic 0000:01:00.0: RXQ stride: 0x00000020
[  132.325427] mqnic 0000:01:00.0: RX queue map indirection table size: 256
[  132.449099] mqnic 0000:01:00.0: Max desc block size: 1
[  132.504297] mqnic 0000:01:00.0: Port-level register blocks:
[  134.355261] mqnic 0000:01:00.0:  type 0x0000c003 (v 0.0.3.0)
[  136.890595] mqnic 0000:01:00.0: Port features: 0x00000007
[  143.165510] mqnic 0000:01:00.0: Port RX ctrl: 0x03010000
[  143.168947] mqnic 0000:01:00.0: Port TX ctrl: 0x03010000
[  158.608503] mqnic 0000:01:00.0: Scheduler block-level register blocks:
[  162.645910] mqnic 0000:01:00.0:  type 0x0000c040 (v 0.0.1.0)
[  162.766715] mqnic 0000:01:00.0: Scheduler type: 0x0000c040
[  162.768456] mqnic 0000:01:00.0: Scheduler offset: 0x00800000
[  162.769464] mqnic 0000:01:00.0: Scheduler channel count: 8192
[  162.770156] mqnic 0000:01:00.0: Scheduler channel stride: 0x00000004
[  162.771836] mqnic 0000:01:00.0: Scheduler count: 1

[  162.784069] mqnic 0000:01:00.0: Exhausted permanent MAC addresses; using random MAC
[  162.849675] mqnic 0000:01:00.0: Registered device mqnic0
ubuntu@ubuntu:~/modules/mqnic$ [  162.974994] mqnic 0000:01:00.0 enp1s0np0: renamed from eth0

ubuntu@ubuntu:~/modules/mqnic$
ubuntu@ubuntu:~/modules/mqnic$ sudo ip addr add 192.168.100.1/24 dev enp1s0np0
sudo ip link set enp1s0np0 up
[  200.641481] mqnic 0000:01:00.0 enp1s0np0: mqnic_start_port on interface 0 netdev 0
ubuntu@ubuntu:~/modules/mqnic$ [  206.284547] IPv6: ADDRCONF(NETDEV_CHANGE): enp1s0np0: link becomes ready

ubuntu@ubuntu:~/modules/mqnic$
ubuntu@ubuntu:~/modules/mqnic$
ubuntu@ubuntu:~/modules/mqnic$
ubuntu@ubuntu:~/modules/mqnic$
ubuntu@ubuntu:~/modules/mqnic$
ubuntu@ubuntu:~/modules/mqnic$
ubuntu@ubuntu:~/modules/mqnic$ ip -s link show enp1s0np0
3: enp1s0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether a6:35:6a:08:2a:06 brd ff:ff:ff:ff:ff:ff
    RX:  bytes packets errors dropped  missed   mcast
             0       0      0       0       0       0
    TX:  bytes packets errors dropped carrier collsns
           266       3      0       0       0       0
ubuntu@ubuntu:~/modules/mqnic$ ping -I enp1s0np0 -c 4 192.168.100.99
PING 192.168.100.99 (192.168.100.99) from 192.168.100.1 enp1s0np0: 56(84) bytes of data.
^C

--- 192.168.100.99 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1024ms

ubuntu@ubuntu:~/modules/mqnic$
ubuntu@ubuntu:~/modules/mqnic$ ping -I enp1s0np0 -c 4 192.168.100.99[  208.352423] ICMPv6: NA: a6:35:6a:08:2a:06 advertised our address fe80::a435:6aff:fe08:2a06 on enp1s0np0!

ubuntu@ubuntu:~/modules/mqnic$ ip -s link show enp1s0np0
3: enp1s0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether a6:35:6a:08:2a:06 brd ff:ff:ff:ff:ff:ff
    RX:  bytes packets errors dropped  missed   mcast
           596       8      0       0       0       0
    TX:  bytes packets errors dropped carrier collsns
           596       8      0       0       0       0


```
## Troubleshooting

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
- cocotb log level if set to DEBUG might trigger sw lock up on linux side.
  Test with INFO or lower level log.
  [  167.426048] watchdog: BUG: soft lockup - CPU#2 stuck for 23s! [systemd:1]
- It takes 5-10 minutes for insmod mqnic.ko to load.

## License

- QEMU device code (`qemu-device/`): **GPL-2.0** (links QEMU)
- Testbench and documentation: **BSD-2-Clause-Views** (matches Corundum and
  cocotbext-pcie)

## Acknowledgments

Built on [Corundum](https://github.com/corundum/corundum) and
[cocotbext-pcie](https://github.com/alexforencich/cocotbext-pcie) by
Alex Forencich. This project adds the QEMU co-simulation bridge enabling
unmodified drivers to run against the Corundum RTL.



