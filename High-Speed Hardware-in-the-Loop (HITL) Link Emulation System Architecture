# High-Speed Hardware-in-the-Loop (HITL) Link Emulation System Architecture

## 1. Abstract
Heterogeneous multi-vendor field-programmable gate array (FPGA) environments present significant physical and logical alignment challenges for high-throughput, low-latency inter-chip communication[cite: 6, 7]. This paper presents the design and implementation of a multi-vendor Hardware-in-the-Loop (HITL) link emulation platform bridging an Intel Agilex 5 FPGA and an AMD/Xilinx Kintex UltraScale+ FPGA[cite: 6, 7]. The physical link utilizes an external Texas Instruments LMK62E2 high-performance oscillator delivering a 156.25 MHz low-jitter reference clock to establish lock and phase synchronization across heterogeneous serializer/deserializer (SerDes) architectures[cite: 6, 7]. The data plane features an ultra-wide 1024-bit parallel packet processing pipeline operating on a dual-clock domain (156.25 MHz / 312.5 MHz), yielding an internal pipeline processing bandwidth up to 320 Gbps[cite: 6, 7]. Real-time frame ingress is driven from a host source over Gigabit Ethernet, while egress traffic is streamed via a four-lane PCI Express (PCIe 3.0 x4) Direct Memory Access (DMA) engine into a host sink[cite: 6, 7]. The resulting platform enables deterministic real-time header parsing, frame filtering, and network coding hardware offload across physical and protocol-level boundaries[cite: 6, 7].

---

## 2. Hardware Component Inventory

The experimental HITL system consists of eight key hardware entities spanning control, processing, conversion, and physical transmission domains, as detailed in Table 1[cite: 6, 7]:

### Table 1: Hardware Component Inventory and Functional Roles

| Component Category | Device / Part Model | Vendor / Model ID | Primary Functional Role & Interconnect |
| :--- | :--- | :--- | :--- |
| **Data Source Host** | Workstation PC1 | Standard x86 Host | Generates and transmits raw data packets via Gigabit Ethernet to the transmitter FPGA[cite: 6, 7]. |
| **Transmitter Platform** | Terasic DE25 Board | Intel Agilex 5 FPGA | Executes ingress packet parsing, 1024-bit header processing, and network coding offload[cite: 6, 7]. |
| **High-Speed Breakout** | Terasic HSMC-XTS | Terasic | Adapts high-density HSMC transceiver lines to coaxial SMA ports (supports 6.25 Gbps per channel)[cite: 6, 7]. |
| **Clock Reference Source** | LMK62E2-156M25EVM | Texas Instruments | Generates a differential 156.25 MHz low-jitter reference clock for SerDes PLL locking[cite: 6, 7]. |
| **Coaxial to SFP+ Adapter** | TE0422-03 Module | Trenz Electronic | Plug-in module adapting 4x SMA differential coax cables (2x TX, 2x RX) to an SFP+ footprint[cite: 6, 7]. |
| **Form-Factor Adapter** | WADQS-28-MEL | 10GTek | Passive port adapter converting SFP+ plug-in modules to a QSFP28 port interface[cite: 6, 7]. |
| **Receiver Platform** | RK-XCKU5P-F V1.2 | OpenSourceSDRLab (Kintex UltraScale+) | Receives high-speed serial streams via QSFP28 port and bridges payload to host memory via PCIe DMA[cite: 6, 7]. |
| **Data Sink Host** | Workstation PC2 | Standard x86 Host | Captures, logs, and evaluates output frame integrity over a PCIe 3.0 x4 bus link[cite: 6, 7]. |

---

## 3. System Model

### 3.1 End-to-End Pipeline Architecture

The complete system topology is modeled as a formal end-to-end data processing chain, illustrated conceptually in Figure 1 and implemented in physical laboratory hardware as shown in Figure 2[cite: 6, 7]:

$$S = ( \text{PC1}_{\text{src}} \rightarrow \text{Ethernet} \rightarrow \mathcal{F}_{\text{Agilex}} \rightarrow \text{HSMC} \rightarrow \mathcal{L}_{\text{coax}} \rightarrow \text{SFP+/QSFP28} \rightarrow \mathcal{F}_{\text{Kintex}} \rightarrow \text{PCIe} \rightarrow \text{PC2}_{\text{snk}} )$$

![Figure 1: Conceptual system block diagram](docs/images/figure1.png)  
*Figure 1: Conceptual system block diagram illustrating end-to-end data pipelines, physical interconnects, and external SerDes clock distribution[cite: 6, 7].*

* **Ingress Stage ($\text{PC1} \rightarrow \text{DE25}$):** Test payloads are framed and transmitted across a Gigabit Ethernet network interface[cite: 6, 7]. The onboard Ethernet MAC on the Intel Agilex 5 converts byte-oriented network streams into internal wide-bus words[cite: 6, 7].
* **Transmitter Processing ($\mathcal{F}_{\text{Agilex}}$):** Incoming packets are parallelized into an ultra-wide $W = 1024\text{-bit}$ internal datapath[cite: 6, 7]. The pipeline performs single-cycle header parsing, field insertion/modification, and network coding (NC) linear combination encoding[cite: 6, 7].
* **SerDes Physical Layer ($\mathcal{L}_{\text{coax}}$):** Wide parallel words are serialized by the Agilex 5 transceivers[cite: 6, 7]. SerDes phase locking across vendors is maintained via the common TI LMK62E2 reference clock ($f_{\text{ref}} = 156.25\text{ MHz}$) over RG405 SMA coaxial lines[cite: 6, 7]. High-speed differential data lanes (SS405, 2x TX, 2x RX) transport multi-gigabit serial data into the Trenz TE0422-03 module[cite: 6, 7].
* **Form-Factor Conversion:** The Trenz TE0422-03 module outputs an SFP+ optical/electrical format, which is passively adapted to a QSFP28 form factor via the 10GTek WADQS-28-MEL converter before entering the AMD/Xilinx Kintex UltraScale+ transceiver RX pins[cite: 6, 7].
* **Egress Stage ($\mathcal{F}_{\text{Kintex}} \rightarrow \text{PC2}$):** The Kintex UltraScale+ FPGA performs deserialization, packet validation, and Network Coding decoding[cite: 6, 7]. High-throughput scatter-gather DMA engines transfer completed frames across a PCIe 3.0 x4 interface into PC2 host memory[cite: 6, 7].

![Figure 2: Physical laboratory hardware setup](docs/images/figure2.png)  
*Figure 2: Physical laboratory hardware-in-the-loop (HITL) setup demonstrating inter-board coaxial differential cabling, clock distribution, and form-factor plug-in adapters[cite: 6, 7].*

### 3.2 Throughput & Datapath Capacity Model

The internal datapath throughput capacity $C_{\text{pipeline}}$ is defined as a function of parallel bus width $W_{\text{bus}}$ and core clock frequency $f_{\text{core}}$[cite: 6, 7]:

$$C_{\text{pipeline}} = W_{\text{bus}} \times f_{\text{core}}$$

Given $W_{\text{bus}} = 1024\text{ bits}$[cite: 6, 7]:

* At $f_{\text{core}} = 156.25\text{ MHz}$:
  $$C_{\text{pipeline}} = 1024 \times 156.25 \times 10^6\text{ bits/s} = 160\text{ Gbps}$$

* At $f_{\text{core}} = 312.50\text{ MHz}$:
  $$C_{\text{pipeline}} = 1024 \times 312.50 \times 10^6\text{ bits/s} = 320\text{ Gbps}$$

This 320 Gbps internal capacity guarantees zero-drop line-rate processing for multi-gigabit physical link speeds, providing adequate headroom for hardware-assisted Network Coding overhead and real-time packet filtering operations[cite: 6, 7].

### 3.3 Detailed Stage-by-Stage Processing Flow

The detailed functional processing pipeline across both FPGA nodes is depicted in Figure 3 and structured into seven discrete sequential stages[cite: 6, 7]:

![Figure 3: Detailed functional flowchart](docs/images/figure3.png)  
*Figure 3: Detailed functional flowchart of the end-to-end hardware-in-the-loop (HITL) datapath across TX (Agilex 5) and RX (Kintex UltraScale+) processing pipelines[cite: 6, 7].*

#### Transmitter Pipeline: Terasic DE25 (Intel Agilex 5)
* **Stage 1 - Host Ingress & DMA:** Workstation PC1 transmits raw Ethernet frames to the Hard Processor System (HPS)[cite: 6, 7]. The HPS transfers packet buffers into the HPS DMA engine, streaming word-aligned payload directly to the FPGA core logic fabric[cite: 6, 7].
* **Stage 2 - Ethernet Ingress Deparser:** Extracts Ethernet MAC layer fields, strips Destination/Source MAC addresses and EtherType headers, and evaluates the Frame Check Sequence (CRC check) in parallel[cite: 6, 7].
* **Stage 3 - IP Ingress Deparser:** Identifies IP protocol version, header length, and 5-tuple addressing parameters[cite: 6, 7]. Strips the IP header and verifies IP header checksum integrity[cite: 6, 7].
* **Stage 4 - Filtering & Pipeline Sync:** Applies real-time rule match logic (e.g., 5-tuple packet filtering)[cite: 6, 7]. Holds packet payload within a delay pipeline buffer while CRC and checksum verification flags settle[cite: 6, 7].
* **Stage 5 - Network Coding Offload:** Encodes valid payload blocks using linear combination coefficients and appends a structured Network Coding (NC) metadata header[cite: 6, 7].
* **Stage 6 - Packet Re-Encapsulation:** Re-encapsulates the encoded payload by prepending an updated IP header with recomputed checksum and re-attaching standard Ethernet MAC framing[cite: 6, 7].
* **Stage 7 - High-Speed SerDes Egress:** Parallelizes data across the 1024-bit wide datapath into the Intel Agilex 5 transceiver block for multi-gigabit serialization through the HSMC-XTS breakout board[cite: 6, 7].

#### Receiver Pipeline: OpenSourceSDRLab KU5P (AMD/Xilinx Kintex UltraScale+)
* **Stage 1 - High-Speed SerDes Ingress:** Receives multi-gigabit serial stream through the QSFP28 interface into Kintex UltraScale+ transceivers, deserializing into a 1024-bit parallel bus[cite: 6, 7].
* **Stage 2 - Ethernet Deparser:** Extracts and strips Ethernet encapsulation while performing parallel CRC validation[cite: 6, 7].
* **Stage 3 - IP Deparser:** Extracts and strips IP headers while evaluating IP header checksum integrity[cite: 6, 7].
* **Stage 4 - NC Header Deparsing:** Parses NC generation vectors and coding coefficients from the header and strips the NC header[cite: 6, 7].
* **Stage 5 - Network Coding Offload:** Executes Gaussian elimination decoding logic to reconstruct original raw packet payload blocks[cite: 6, 7].
* **Stage 6 - Packet Reconstruction:** Re-attaches valid IP headers with recalculated checksums and reconstructs standard Ethernet framing[cite: 6, 7].
* **Stage 7 - Host Egress & PCIe DMA:** Streams reconstructed packets directly to PC2 host memory via a high-performance PCIe 3.0 x4 scatter-gather DMA engine[cite: 6, 7].
