# Wireless System Emulation: Fully Scalable AWGN Hardware Emulator

**Project Title:** Fully Scalable AWGN System with Variable Bandwidth and Channel Support   
**Target Hardware:** Intel Agilex 7 / AMD Xilinx FPGAs  
**Core Specification:** 375 MHz – 400 MHz ($F_{\max}$), 256-bit / 512-bit AXI4-Stream Datapath, APB Control Plane  

---

## 1. Executive Summary

This project presents a fully scalable, resource-optimized Additive White Gaussian Noise (AWGN) hardware emulation system designed for satellite (DVB-S2X) and wideband wireless communication testbeds. Operating at a 375 MHz core clock on an Intel Agilex 7 FPGA, the design supports variable sample rates ranging from **375 MSPS to 6 GSPS** across arbitrary $N$-channel configurations ($N \le 64$). 

### Key Project Highlights & CV Credentials
* **High-Throughput Datapath:** Architected 256-bit and 512-bit AXI4-Stream datapaths reaching a maximum operating frequency ($F_{\max}$) of **400 MHz** on Intel Agilex 7, with a fixed pipeline latency of 57 clock cycles.
* **Taus258 + Box-Muller Core:** Combined a 5-component 64-bit LFSR PRNG ($\\approx 2^{258}-1$ period, or $2.45 \\times 10^{61}$ years) with a pipeline-aligned Box-Muller transformation engine generating complex 32-bit Gaussian noise.
* **Signal-Agnostic Power Scaling:** Designed an online Exponential Moving Average (EMA) power estimation block that dynamically measures input signal power ($P_{\text{signal}}$) and scales injected noise to enforce a software-defined target $E_s/N_0$ with **$\\pm 0.1 \\text{ dB}$ precision**.
* **Resource-Shared Multi-Channel Architecture:** Decoupled active channel count from physical AWGN hardware instances using AXI4-Stream `tkeep` sample masking and shared-resource adder trees.
* **HW/SW Co-Design:** Integrated an APB slave interface with C/C++ memory-mapped drivers for dynamic seed configuration, adaptive/static power mode selection, and live 64-bin PDF histogram extraction.

---

## 2. System Architecture

![Figure 1: System Architecture](docs/images/awgn/DesignBlock_AWGN_Adder-001-001.jpg)

### 2.1 Taus258 Combined LFSR Generator
To eliminate statistical correlations common in single LFSRs, the PRNG implements the Taus258 algorithm running five independent 64-bit LFSR components in parallel:

$$u = \text{LFSR}_1 \oplus \text{LFSR}_2 \oplus \text{LFSR}_3 \oplus \text{LFSR}_4 \oplus \text{LFSR}_5$$

```c
// Taus258 Reference Generator Snippet
unsigned long long z1, z2, z3, z4, z5;

double lfsr258() {
    unsigned long long b;
    b = (((z1 << 1) ^ z1) >> 53);   z1 = (((z1 & 18446744073709551614ULL) << 10) ^ b);
    b = (((z2 << 24) ^ z2) >> 50);  z2 = (((z2 & 18446744073709551104ULL) << 5) ^ b);
    b = (((z3 << 3) ^ z3) >> 23);   z3 = (((z3 & 18446744073709547520ULL) << 29) ^ b);
    b = (((z4 << 5) ^ z4) >> 24);   z4 = (((z4 & 18446744073709420544ULL) << 23) ^ b);
    b = (((z5 << 3) ^ z5) >> 33);   z5 = (((z5 & 18446744073701163008ULL) << 8) ^ b);
    return (z1 ^ z2 ^ z3 ^ z4 ^ z5);
}
```

### 2.2 Box-Muller Transformation
Uniform random variables $u_0$ (48-bit) and $u_1$ (16-bit) are transformed into normalized Gaussian variables ($awgn_0, awgn_1$):

$$awgn_0 = f \cdot \cos(\theta), \quad awgn_1 = f \cdot \sin(\theta)$$

Where amplitude $f = \sqrt{-2 \ln(u_0)}$ and phase $\theta = 2\pi u_1$.

* **Amplitude ($f$):** Normalizes $u_0 = 2^{-k}(1+m)$, split into an Exponent LUT ($2k\ln 2$) and Mantissa LUT ($-2\ln(1+m)$), followed by a Digit-Recurrence Square Root module.
* **Phase ($\\theta$):** Uses a Quarter-Wave Sine/Cosine LUT mapping $[0, 65535] \rightarrow [0, 2\pi]$.
* **Pipeline Synchronization:** Aligns the 20-cycle amplitude calculation delay with the 3-cycle phase path via shift registers before final multiplication to Q6.12 (18-bit fixed-point).

![Figure 2: Box-Muller Transformation](docs/images/awgn/DesignBlock_BoxMuller-001-001.jpg)

### 2.3 Real-Time Power Estimation & Scaling
To maintain a target $E_s/N_0$ dynamically, input power $P_{\text{inst}}[n] = I[n]^2 + Q[n]^2$ is computed using dedicated 19x18 DSP blocks and filtered via an Exponential Moving Average (EMA) pipeline:

$$P_{\text{avg}}[n] = \alpha \cdot P_{\text{inst}}[n] + (1 - \alpha) \cdot P_{\text{avg}}[n-1], \quad \alpha = 2^{-16}$$

The gain scaling factor $K_{\text{noise}}$ is derived as:

$$K_{\text{noise}} = \sqrt{P_{\text{signal}}} \cdot \frac{1}{\sqrt{2}} \cdot \frac{1}{\sqrt{10^{\text{SNR}_{\text{dB}}/10}}}$$

$$\text{SNR}_{\text{dB}} = \left(\frac{E_s}{N_0}\right)_{\text{dB}} - 10 \log_{10}\left(\frac{F_s}{B}\right)$$

![Figure 3: Real-Time Power Estimation](docs/images/awgn/DesignBlock_PowerEst-001-001.jpg)


---

## 3. Register Map & Control Plane (APB Slave)

| Address | Bit Range | Access | Register Name | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x000` | `[0]` | R/W | **System Enable** | `0`: Disable, `1`: Enable noise injection |
| `0x000` | `[1]` | R/W | **System Reset** | `0`: Active Reset, `1`: Normal Operation |
| `0x000` | `[31]` | R/W | **Power Est. Enable** | `0`: Static (Manual Power), `1`: Adaptive (Auto EMA) |
| `0x040` | `[31:0]` | R/W | **Initial Power Ch 0** | EMA seed parameter (Adaptive) or static value (Manual) |
| `0x140` | `[15:0]` | R/W | **Norm. StdDev Ch 0** | Software scale factor: $\\frac{1}{\\sqrt{2 \\cdot 10^{\\text{SNR}/10}}}$ |
| `0x240` | `[31:0]` | R/W | **PRNG Seed Low 0** | 32 LSBs of Taus258 seed for Channel 0 |
| `0x244` | `[31:0]` | R/W | **PRNG Seed High 0** | 32 MSBs of Taus258 seed for Channel 0 |
| `0xE40` | `[5:0]` | R/W | **Hist. Indexing** | Bin index (0–63) for hardware PDF histogram readback |
| `0xE40` | `[11:8]` | R/W | **Hist. Scale Mode** | Scale selector ($/1$ to $/512$) for Gaussian bell-curve viewing |

![Figure 3: APB registers](docs/images/awgn/DesignBlock_AWGN_Top-001-001.jpg)

---

## 4. Hardware Resource Utilization (Intel Agilex 7)

All configurations achieve a maximum frequency ($F_{\max}$) of **400 MHz**:

| Hierarchy Configuration | ALUTs | Dedicated Registers | Memory Bits | DSP Blocks |
| :--- | :---: | :---: | :---: | :---: |
| **256-bit (8 AWGN Lanes, Core)** | 13,544 | 27,973 | 423,424 | 42 |
| **512-bit (16 AWGN Lanes, Core)** | 23,431 | 51,411 | 846,848 | 82 |
| **256-bit (8 AWGN Lanes, With Mon.)** | 21,463 | 38,309 | 423,424 | 76 |
| **512-bit (16 AWGN Lanes, With Mon.)** | 32,481 | 65,471 | 846,848 | 148 |

---

## 5. Verification & Experimental Results

### 5.1 Fixed-Point RTL vs. Floating-Point Software Model
Validated SystemVerilog RTL (16-bit Q2.14 fixed-point) against a double-precision floating-point Python model across $N = 10,000,000$ samples:
* **Time-Domain Waveform:** Perfect cycle-by-cycle tracking without cumulative phase drift.
* **Variance Match:** Hardware variance measured **1.00072** vs. floating-point reference **1.00026**.
* **Quantization Error:** Mean Absolute Error (MAE) of **$0.00051$**; maximum absolute error bounded at **$0.02946$**.

### 5.2 Multi-Channel SNR Sweep Accuracy
Evaluated over 8 independent channels with AXI4-Stream `tkeep`/`tlast` signaling across an $E_s/N_0$ range of $-9 \text{ dB}$ to $+38 \text{ dB}$:

![Figure 5: Waveform Overlay of first 1000 samples](docs/images/awgn/Result_Sim_WaveOverlay-001-001.jpg)

![Figure 6: Gaussian Distribution](docs/images/awgn/Result_Sim_Dist-001-001.jpg)

![Figure 7: Absolute Error Distribution](docs/images/awgn/Result_Sim_Err-001-001.jpg)

* **Optimal Range ($0 \text{ dB}$ to $25 \text{ dB}$):** Injection error is strictly bounded within **$\\pm 0.02 \\text{ dB}$**.
* **Extremes ($> 30 \text{ dB}$):** Slight deviation ($< 0.18 \text{ dB}$) due to Q2.14 LSB truncation limits at very small noise power levels.

---

## 6. Conclusion

The fully scalable AWGN hardware emulator provides a robust, high-throughput, and resource-efficient solution for real-time wireless link emulation. By combining Taus258 PRNG, Box-Muller transformation, dynamic EMA power estimation, and APB host control, the system achieves $\\pm 0.1 \\text{ dB}$ noise injection accuracy at $400 \\text{ MHz}$ on Intel Agilex 7 FPGAs.
