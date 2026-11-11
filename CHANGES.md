# Changes vs. `b465582` ("pierwsza koncepcja schematu")

Scope: `Hardware/User_area.kicad_sch` only. No other project files, template sheets (Bus switch, ID EEPROM, Interboard connector, Power connector) or the PCB were modified.

All changes were verified by exporting the netlist before and after (`kicad-cli sch export netlist`) and diffing it — only the nets listed below changed.

ERC (`kicad-cli sch erc --severity-all`): 82 → 79 violations (16 errors, 63 warnings). No new violation types; most remaining ones are in template sheets.

---

## 1. Pin-assignment fixes (label changes on U701 / U702)

### 1.1 ChainBus SPI → hardware SPI0 slave
The ChainBus standard forbids a hat from being an SPI/I2C master, so the RP2350 is the **SPI slave**. The old mapping did not match any SPI0 function (MOSI was on SPI0 CSn, CS on SPI0 TX).

| GPIO | SPI0 function | Before | After |
|---|---|---|---|
| GPIO0 | RX | Chainbus.MISO | **Chainbus.MOSI** |
| GPIO1 | CSn | Chainbus.MOSI | **Chainbus.CS** |
| GPIO2 | SCK | Chainbus.SCK | Chainbus.SCK (unchanged) |
| GPIO3 | TX | Chainbus.CS | **Chainbus.MISO** |

### 1.2 PSRAM chip select, HDMI DDC, CEC
RP2350 QMI CS1n (memory-mapped PSRAM) is only available on GPIO0/8/19/47 — GPIO25 could not work. DDC was split across I2C0 (SDA GPIO8) and I2C1 (SCL GPIO7).

| GPIO | Before | After | Reason |
|---|---|---|---|
| GPIO6 | MCU_CEC | **MCU_SDA** | I2C1 SDA, pairs with GPIO7 = I2C1 SCL |
| GPIO8 | MCU_SDA | **QPI_CS_SRAM** | QMI CS1n |
| GPIO25 | QPI_CS_SRAM | **MCU_CEC** | CEC is slow single-wire, any GPIO |

### 1.3 HDMI TMDS series-resistor topology
Before: GPIO connected **directly** to the HDMI connector, and the 270 Ω resistors isolated the TPD12S016 ESD clamps from the line (ESD ineffective, no current limiting).

After: `GPIO → R70x → (J701 pin + TPD12S016 pin)`.
- GPIO-side nets renamed `HSTX_D0±`, `HSTX_D1±`, `HSTX_D2±`, `HSTX_CLK±`.
- New labels `D0±/D1±/D2±/CLK±` added at the U702/resistor node so the connector, ESD clamp and resistor share one net.
- *Cosmetic:* new labels next to U702 overlap its pin numbers — tidy manually.

### 1.4 TPD12S016 `CT_HPD` was floating
`CT` label (connected to nothing) renamed to **`HDMI_CT_HPD`** and connected to **GPIO20**, so firmware can enable the HDMI 5 V load switch and HPD buffer. (Leave GPIO20 low → HDMI 5 V off until firmware enables it.)

### 1.5 `VREG_AVDD` net named
Label `VREG_AVDD` added on the R717 – U701 pin 46 net (previously unnamed) to attach its filter capacitor.

---

## 2. Component values set (previously placeholders `R` / `C` / `L` / `Crystal`)

| Ref | Before | After | Function / source |
|---|---|---|---|
| R701–R708 | R | 270R | HSTX TMDS series resistors (Raspberry Pi pico-examples suggestion; Adafruit uses 220 Ω) |
| R709, R710 | R | 27R | Native USB series resistors |
| R711 | R | 10k | RUN pull-up (TPL5010 RSTn is open-drain) |
| R717 | R | 33R | VREG_AVDD RC filter |
| R719 | R | 1k | Crystal series resistor |
| L702 | L | 3.3uH | RP2350 internal regulator inductor |
| Y701 | Crystal | 12MHz | RP2350 crystal |
| C707 | C | 4.7uF | VREG_VIN bulk |
| C708–C711 | C | 100nF | 3V3 decoupling |

### Buck converter U706 (LMR38020), per TI datasheet SNVSC40E §9.2.2 / Table 9-1 (5 V, 400 kHz, 2 A)

| Ref | Before | After | Function |
|---|---|---|---|
| R712 | R | 100k | RFBT (VCC → FB) |
| R715 | R | 24.9k | RFBB (FB → GND) → VOUT = 1.0 V × (1 + 100k/24.9k) = 5.02 V |
| R716 | R | 64.9k | RT → fSW = 400 kHz (Table 8-1) |
| L701 | L | 15uH | Isat must be **≥ 3.8 A** (max high-side current limit) |
| C701 | C | 4.7uF/100V | CIN, X7R |
| C702 | C | 100nF/100V | CHF, X7R, closest to VIN/GND pins |
| C703 | C | 100nF/16V | CBOOT |
| C704 | C | 22uF/16V | COUT 1/3 |

---

## 3. Components added

| Ref | Value | Net (top / bottom) | Purpose |
|---|---|---|---|
| C712 | 4.7uF | +1V1 / GND | Internal regulator output bulk (previously **no** capacitor on +1V1) |
| C713–C715 | 100nF | +1V1 / GND | DVDD pins 6, 23, 39 |
| C716 | 4.7uF | VREG_AVDD / GND | VREG_AVDD RC filter (with R717 33 Ω) |
| C717–C721 | 100nF | +3V3 / GND | Remaining IOVDD, ADC_AVDD, USB_OTP_VDD, QSPI_IOVDD |
| C722 | 100nF | +3V3 / GND | W25Q128 flash VCC |
| C723 | 100nF | +3V3 / GND | APS6404L PSRAM VDD |
| C724 | 100nF | +3V3 / GND | TPD12S016 VCCA |
| C725 | 100nF | +3V3 / GND | TPL5010 VDD |
| C726 | 100nF | VCC / GND | TPD12S016 VCC5V |
| C727 | 100nF | HDMI_SUPPLY / GND | TPD12S016 5V_OUT |
| R724 | 10k | +3V3 / QPI_CS_SRAM | PSRAM CS1 pull-up (required by RP2350 datasheet for a GPIO-based second chip select) |
| C728, C729 | 22uF/16V | VCC / GND | LMR38020 output capacitors 2/3 and 3/3 (datasheet: 3 × 22 µF) |

C712–C727 and R724 are in a new block "Decoupling / pull-ups added in review" at the bottom of the sheet; C728/C729 are next to C704 in the buck converter block. No footprints assigned yet.

---

## 4. Final RP2350 GPIO map

| GPIO | Function | GPIO | Function |
|---|---|---|---|
| 0 | SPI0 RX ← Chainbus.MOSI | 12/13 | HSTX D2+/− |
| 1 | SPI0 CSn ← Chainbus.CS | 14/15 | HSTX D1+/− |
| 2 | SPI0 SCK ← Chainbus.SCK | 16/17 | HSTX D0+/− |
| 3 | SPI0 TX → Chainbus.MISO | 18/19 | HSTX CLK+/− |
| 4/5 | I2C0 SDA/SCL (ChainBus, EEPROM 0x50) | 20 | HDMI_CT_HPD |
| 6/7 | I2C1 SDA/SCL (HDMI DDC) | 23/24 | Watchdog DONE / WAKE |
| 8 | PSRAM CS (QMI CS1n) | 25 | HDMI CEC |
| 9 | HDMI HPD in (+ TPD12S016 LS_OE) | 21, 22, 26–29 | free |
| 10/11 | PIO-USB D+/D− (HMI USB) | | |

HSTX lane-to-pin mapping differs from pico-examples defaults — configure it in firmware.

---

## 5. Not changed — still open

- **HMI USB (P702) cannot act as a host:** VBUS not connected, 5.1 kΩ Rd on CC (advertises a sink), no current-limited VBUS switch. Needs a decision: USB-C receptacle (56 kΩ Rp + load switch) or USB-A.
- Both USB parts use the USB-C **plug** symbol instead of a receptacle.
- Values still unset: C705/C706 (crystal load caps, depend on crystal; 15 pF for ABM8-272-T3), R718 (TPL5010 period), R720/R721 (PIO-USB series), R713/R714/R722/R723.
- Footprints missing for ~30 User Area parts; PCB layout of the User Area not started (DRC: 0 errors, 81 schematic-parity warnings).
- PWR_FLAGs needed on `VREG_AVDD`, `+1V1`, `HDMI_SUPPLY` (ERC false positives).
- No interrupt/ready line on ChainBus (pins 25/27 unused — template-level decision).
- Power budget exceeds the 185 mA/hat BRD_VIN limit at 12 V if the touchscreen draws ~500 mA.
- Minor: USB ESD diodes on MCU side of 27 Ω resistors; no 1 kΩ in series with BOOTSEL (SW702).
