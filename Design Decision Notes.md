# Field Recorder — Design Notes

Portable, high-quality audio field recorder (TP-7 style).
Status: component selection — audio inputs, MCU, clocks and power system chosen. Output path next.

---

## Decisions so far

| Area | Decision | Why |
|---|---|---|
| ADC | **TI TLV320ADC6140** (4 ch, WQFN-24) | 113 dB dynamic range, ~9 mW/ch, mic bias in the plug-in-power range |
| Number of ADCs | **2** (8 channels) | 5 channels used: 2 stereo jacks (4) + built-in mic (1). 3 spare — left unused for rev 1 |
| Built-in mic | **Infineon IM73A135V01** (analog MEMS, differential) | 73 dB(A) SNR, 135 dB SPL max, IP57 |
| Mic power | ADC **MICBIAS = 2.75 V** (VREF) | Mic VDD range 2.3–3.0 V, **3.0 V absolute max**. Do NOT use the ~3.01 V or 3.3 V (AVDD bypass) settings |
| Mic coupling | **2 × 1 µF** caps, one on OUT+ and one on OUT− | Blocks the mic's ~1.3 V DC; matched caps keep differential noise rejection |
| Starting gain | **~3 dB** (firmware-adjustable, per channel) | Lines up mic max (+3 dBV) with ADC max (+6 dBV) |
| Input switching | Separate ADC channels, selected in firmware | No analog switch in the signal path |
| Jacks | **3 × 3.5 mm: 2 stereo inputs + 1 output** | Personal recorder — 2 inputs is plenty. Output needs a DAC + driver (not designed yet) |
| MCU | **STM32H743ZIT6** (LQFP144, non-crypto) | SAI w/ TDM, SDMMC, 1 MB RAM, USB OTG HS + ULPI |
| Dev board | NUCLEO-H743ZI2 | Same chip, start firmware before custom PCB |
| Storage | Removable SD card, **SDMMC 4-bit** | ~1 MB/s needed; SPI mode is too marginal |
| USB | High-Speed via **USB3300** ULPI PHY | Full-Speed would take ~1 hr to copy 1 hr of audio |
| MCU supply | **3.3 V** | PHY needs ≥3.0 V; analog switch (PC2_C/PC3_C) works best here |
| Crystal (HSE) | **24.576 MHz** | Whole-number dividers give exactly 48 kHz |
| Audio link | MCU = **master**, SAI1 A **Master Receive**, ADCs = slaves | One clock source for both ADCs |

---

## Power system

```
USB-C (5 V) ──► BQ25616 charger ──► SYS (≥3.5 V w/ USB) ──┬─► buck ──► 3.3 V digital (MCU, SD, USB PHY, ADC IOVDD)
                    │                                    └─► TPS7A2033 LDO ──► 3.3 V analog (ADC AVDD → MICBIAS 2.75 V)
                    ▼
            LiPo 2000 mAh (PKCELL LP-803860, with protection board)
```

| Part | Choice | Why |
|---|---|---|
| Battery | **PKCELL LP-803860**, 3.7 V 2000 mAh LiPo, built-in PCM (Adafruit) | Built-in (not swappable) — simpler. ~8 × 38 × 60 mm |
| Analog rail | **TPS7A2033PDBVR** (3.3 V fixed, SOT-23-5) | 7 µV RMS noise, 95 dB PSRR @ 1 kHz, 0.14 V dropout, stable w/ ≥1 µF ceramic, 6.0 V max in |
| Digital rail | **TPS62046** buck (fixed 3.3 V; or TPS62040 adjustable), MSOP-10 PowerPad | 2.5–6.0 V in, 1.2 A, 1.25 MHz, 100% duty cycle, MODE pin for forced PWM. Older part but Active |
| Charger | **BQ25616** (standalone, 1.5 MHz switching, power path) | 92% efficient (linear would dump ~1.3 W of heat), runs device from USB even with dead battery |

**Why these voltages**
- AVDD = **3.3 V** (not 1.8 V): MICBIAS is made from AVDD, and 1.8 V can't make 2.75 V. ADC AVDD range is 3.0–3.6 V
- IOVDD = **3.3 V**: must match the STM32's I/O voltage (mismatch → current through protection diodes, or unreadable logic levels)
- Separate analog LDO: digital switching noise on AVDD gets recorded (whine, SD-write "ticks")
- Battery cutoff ~**3.4 V**: LDO needs ~3.35 V in at ~24 mA. Li-ion holds most energy above 3.5 V, so little is lost

**Battery sizing**
- Power budget (rough): ADC ×2 ~160 mW, mic <1 mW, H743 ~150–300 mW, SD ~50–150 mW, UI ~50 mW → **~0.6 W** incl. regulator losses
- 0.6 W × 8 h = 4.8 Wh → 4.8 Wh ÷ 3.7 V = **~1300 mAh** minimum
- +30–50% margin (cutoff, aging, cold, estimate error) → **2000 mAh**
- Remember: **Ah = Wh ÷ V**

**LDO waste:** (V_in − V_out) × I. With the Li-ion at full charge: (4.2 − 3.3) × 24 mA ≈ **22 mW**, dropping as the battery drains. Small vs. ~0.6 W total

**BQ25616 settings**
| Pin | Setting | Value |
|---|---|---|
| ILIM | 1 kΩ (1%) | Input limit = 478 A·Ω ÷ R → **500 mA max** (459–500 mA over K tolerance). Safe on any USB port |
| ICHG | 680 Ω (1%) | Charge current = 677 A·Ω ÷ R → **~1 A** (max ~1.05 A; battery allows 2 A) |
| VSET | set for 4.2 V | Matches battery |
| D+/D– | **not connected** to USB data lines | Avoids stub/capacitance on HS USB pair and detection collisions → charger uses ILIM |
| TS | RT1 = 5.3 kΩ (REGN→TS), RT2 = 31.14 kΩ (TS→GND), **10 kΩ fixed resistor** in the NTC position | Fakes 25 °C. Swap for a 103AT NTC later for real 0–45 °C protection |
| CE | low (tie or GPIO) | Low = charging enabled |
| STAT, PG | → MCU GPIOs | Charge status / USB power good |
| OTG boost | off | Not needed |

**Charging reality:** USB capped at 500 mA × 5 V = 2.5 W. Recording while charging: (2.5 − 0.6) W × 0.92 ÷ 3.8 V ≈ **0.46 A** into battery; recorder off ≈ 0.6 A. Full charge ≈ **4–5 h** (constant-current to ~80%, then slow constant-voltage taper)

**USB-C:** 5.1 kΩ pull-downs on CC1 and CC2 (marks the device as a power sink)

**TPS62046 buck settings**
| Pin | Setting | Why |
|---|---|---|
| MODE | → MCU GPIO + **~100 kΩ pull-down** | Low = power-save (efficient when idle, default during boot). High = forced PWM (fixed 1.25 MHz, no frequency dropping into the audio band) |
| L / C | ~6.2 µH, 22 µF in and out (per datasheet typical application) | Follow the datasheet layout guidance closely |

- Firmware: set MODE **high before** a recording starts, low when idle. Never switch mid-recording (rail blip)
- Power-save (PFM) mode lowers and varies the switching frequency at light load — can land in the audio band as a whine

---

## Input jacks (×2, stereo)

TRS plug: **Tip = left, Ring = right, Sleeve = ground**. Each jack uses 2 ADC channels.

**Signal chain (per jack, per channel)**
```
Tip/Ring ─┬─ 5 V bidirectional TVS ─ GND
          ├─ ~2.2 kΩ ─ [plug-in power switch] ─ MICBIAS (2.75 V)
          └─ 0 Ω series ─┬─ [DNP pad resistor to GND] ─ 1 µF coupling cap ─ ADC INxP
                                                        ADC INxM ─ 1 µF ─ GND  (grounded after its cap — confirm against ADC datasheet Figure 37)
Detect switch (isolated, normally open) ─ GND / other side ─ MCU GPIO (internal pull-up)
```

| Item | Decision | Why |
|---|---|---|
| Plug-in power | **Switchable, one switch per jack** (P-MOSFET or load switch between MICBIAS and that jack's bias resistors, GPIO-controlled) | Lav mic on one jack + line source on the other. Always-on bias causes pops, DC through dynamic mics/pickups, issues with DC-coupled outputs. Can't switch MICBIAS itself — the built-in mic needs it |
| Input mode | **Single-ended** (INxM grounded after its coupling cap) | 3.5 mm signals are single-ended. Full scale is **1 V RMS** single-ended (vs 2 V RMS differential) |
| Line pad | **None for rev 1** — 0 Ω series resistor + DNP shunt footprint | Small mics only; pad can be added later by fitting two resistors. Pro line (~1.2 V RMS + peaks) would clip |
| ESD | **5 V standoff, bidirectional TVS** on tip and ring | Max normal voltage = 2.75 V bias + 1.41 V peak (1 V RMS × √2) = 4.16 V. Bidirectional because with plug-in power off the signal swings negative — a unidirectional TVS would clip below −0.7 V |
| Jack detect | Jack with an **isolated, normally-open detect switch**; switch → GND, other side → GPIO with pull-up | Active low: HIGH = no plug, LOW = plug. Avoid jacks whose "detect" is part of the tip contact (puts bias/audio on the detect pin) |

**Firmware notes**
- Detect: interrupt on both edges, debounce ~20–50 ms
- After detect settles: turn on plug-in power (if enabled for that jack), then unmute — avoids plug-in pops
- **V_peak = V_RMS × √2**

**Spare ADC channels (3):** route to small test pads for later experiments (second built-in mic for stereo, "safety track" at lower gain, extra input)

---

## Key numbers

**Mic → ADC levels**
- Mic sensitivity: −38 dBV at 94 dB SPL (= 1 Pa)
- Mic max output: −38 + (135 − 94) = **+3 dBV ≈ 1.41 V RMS** (10% THD point; 1% THD at 132 dB SPL)
- ADC full scale: **2 V RMS differential ≈ +6 dBV**
- Mic noise floor: −111 dBV(A)
- ADC noise floor at 0 dB gain: +6 − 113 ≈ **−107 dBV**
- Gain slides the mic's "window" up 1 dB per dB. Too little → ADC hiss; too much → clipping

**Coupling cap**
- Cutoff f = 1 / (2πRC). With R = 10 kΩ, 20 Hz needs ≥ ~800 nF → use **1 µF** (~16 Hz)
- ADC's default quick-charge assumes caps ≤ 1 µF (else change INCAP_QCHG)
- Use **low voltage-coefficient caps** (film or C0G), not X7R, for low distortion

**Data rate / buffering**
- bytes/s = channels × sample rate × bytes per sample
- 7 ch × 48 kHz × 3 B = **~1.0 MB/s**
- SD cards can stall 100–250 ms → 250 ms × 1 MB/s ≈ 252 KB → target **~500 KB RAM buffer**

**Clock chain (exact 48 kHz)**
```
24.576 MHz HSE → DIVM2 ÷2 → 12.288 MHz
             → DIVN2 ×16 → 196.608 MHz (VCO)
             → DIVP2 ÷4  → 49.152 MHz  → SAI1 kernel clock
             → SAI ÷4    → 12.288 MHz  BCLK
             → ÷256      → 48 kHz      FSYNC
```
- BCLK = 8 slots × 32 bits × 48 kHz = 12.288 MHz
- If CubeMX flags the VCO as too low: DIVP2 ÷8, DIVN2 ×32 (VCO 393.216 MHz), same output

---

## CubeMX configuration (STM32H743ZIT6)

**Pins**
| Function | Pins |
|---|---|
| SAI1 A | FS = PE4, SCK = PE5, SD = PE6 |
| SDMMC1 | D0 = PC8, D1 = PC9, D2 = PC10, D3 = PC11, CK = PC12, CMD = PD2 |
| I2C1 | SCL = PB6, SDA = PB7 |
| USB OTG HS (ULPI) | STP = PC0, DIR = PC2_C, NXT = PC3_C, CK = PA5, D0 = PA3, D1 = PB0, D2 = PB1, D3 = PB10, D4 = PB11, D5 = PB12, D6 = PB13, D7 = PB5 |
| Debug | Serial Wire (Trace and Debug → DEBUG) — PA13/PA14 |
| HSE | Crystal on PH0/PH1 |

**SAI1 A parameters**
| Setting | Value |
|---|---|
| Audio Mode | Master Receive |
| Protocol | Free |
| Frame Length | 256 bits |
| Data Size | 24 bits |
| Slot Size | 32 bits |
| Number of Slots | 8 |
| Slot Active | All |
| Audio Frequency | 48 kHz → Real 48.0 kHz, 0.0% error ✓ |
| Frame Synchro Active Level Length | 1 |
| Frame Synchro Definition | Start Frame |
| Frame Synchro Offset | First Bit (matches ADC TX_OFFSET = 0) |
| Frame Synchro Polarity | **Active High** (ADC frame starts on FSYNC rising edge) |
| Clock Strobing | Falling Edge — **verify** (see To verify) |

**Clocks**
- PLL2 → SAI1 (49.152 MHz)
- USB clock mux → **RC48** (USB needs exactly 48 MHz; PLL1Q was giving 49.152)
- System clock currently HSI 64 MHz — revisit (PLL1 for more speed later)
- PLL3 unused — set to valid values (e.g. DIVM3 ÷2) to clear the error

---

## SAI ↔ ADC wiring

| Signal | MCU pin | ADC pin | Direction |
|---|---|---|---|
| Bit clock (12.288 MHz) | PE5 (SAI1_SCK_A) | BCLK | MCU → ADC |
| Frame sync (48 kHz) | PE4 (SAI1_FS_A) | FSYNC | MCU → ADC |
| Audio data | PE6 (SAI1_SD_A) | SDOUT | ADC → MCU |

- With 2 ADCs: **shared** BCLK, FSYNC and data line. Each ADC is set (over I2C) to use its own slots and tri-state the rest
- Give each ADC its own I2C address with ADDR0/ADDR1
- No MCLK needed — ADC's PLL runs from BCLK
- Start BCLK/FSYNC **before** powering up the ADCs so their PLL can lock

---

## Open questions

- [ ] Output jack: headphones, line out, or both? → which DAC + driver
- [ ] Should the CPU run from PLL1 (faster) or stay on HSI (lower power)?
- [ ] DRE on or off? (Design without it; test on the built-in mic and listen for "breathing")
- [ ] Rev 2: real NTC thermistor on the battery? Faster charging via I2C charger or USB switch (options B/C)?

## To verify

- [ ] SAI **Clock Strobing** setting: ADC sends data on BCLK rising edge, so MCU should sample on falling edge. Check CKSTR in the reference manual and confirm with a logic analyzer
- [ ] USB3300 output delay ≤ 6 ns (assumed from ULPI spec). ULPI slack with PC2_C/PC3_C ≈ 6.2 ns (16.7 − 6 − 4.5)
- [ ] H743 silicon revision — the 4.5 ns ULPI setup figure is from the **rev V** section (Table 202)
- [ ] Ordering code is the **LQFP** package (check ordering table, datasheet p. 346)
- [ ] Mic bottom-port hole size (mic datasheet p. 12 footprint)
- [ ] BQ25616: what to do with **unused D+/D– pins** (floating vs. tied) — check D+/D– detection section / TI E2E
- [ ] BQ25616: VSET pin connection for 4.2 V
- [ ] **JST battery connector polarity** — check against the board before plugging in
- [ ] Battery fits the case (8 × 38 × 60 mm), with space for swelling
- [ ] TPS62046 price/stock on DigiKey (or a newer TPS62xxx equivalent)
- [ ] Pick the actual 3.5 mm jack part (isolated NO detect switch) and TVS part (5 V, bidirectional, low leakage)
- [ ] Pick the plug-in power switch part (small P-MOSFET or load switch, works at 2.75 V)

---

## Pitfalls to remember

- Never design right at an absolute max (mic VDD 3.0 V)
- Datasheet speeds are raw maximums — real throughput is lower (USB FS ≈ 1 MB/s, not 1.5)
- H743 RAM is split into blocks — **not all are DMA-reachable**; put audio buffers in the right one
- H743 data cache vs DMA can disagree about memory contents — handle cache coherency
- SD cards stall — always buffer
- Debounce the jack-detect switch; fade or switch only when not recording to avoid pops
- ULPI traces: short and length-matched
- Don't charge Li-ion below 0 °C (permanent damage) — rev 1 has no temperature sensing
- LiPo pouches are fragile: no screws/sharp edges/pressure; leave room for swelling
- Keep the charger and its inductor **far from the ADC and mic inputs**
- Design for the worst case of a tolerance (e.g. max K_ILIM), not the typical
- Use 0 Ω + DNP footprints on rev 1 to keep options open without a new board
- Unidirectional TVS on an AC signal = one-sided clipping (confusing to debug)

## Next up

1. **Output path** (DAC + headphone/line driver — ADC6140 is input only)
2. Pick real part numbers for the input jacks, TVS and plug-in power switches
3. SAI DMA, SDMMC and USB settings (firmware stage)

## Reference documents

- TLV320ADC6140 datasheet (SBAS992A) — TDM timing §8.3.1.2.1, DRE §8.3.7
- Infineon IM73A135V01 datasheet (v1.20)
- STM32H742/743 datasheet (DS12110 Rev 11) — ULPI timing Table 202, pin table, ordering p. 346
- TPS7A20 datasheet (SBVS338H)
- BQ25616 datasheet (SLUSDF7A) — TS divider §9.3.7.4
- TPS6204x datasheet (SLVS463B) — MODE pin, typical application circuit
- PKCELL LP-803860 spec (QA.S.0228)
