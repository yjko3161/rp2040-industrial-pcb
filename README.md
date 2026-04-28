# RP2350B Industrial PoE Board

RP2350B 기반 산업용 PoE 통신 보드. RS485 4채널 + RS232 2채널 + Ethernet을 하나의 보드에서 처리.

## Block Diagram

```
                        ┌─────────────────────────────────┐
                        │          RP2350B (QFN-80)        │
                        │     Dual Cortex-M33 @ 150MHz     │
   ┌──────────┐         │                                  │         ┌──────────────┐
   │  RJ45    │PoE 48V  │  ┌──────┐  ┌──────┐  ┌──────┐  │  UART0  │  SP3485 #1   │── RS485 Ch1
   │  Magjack ├────────►│  │ PIO0 │  │ PIO1 │  │ PIO2 │  ├────────►│  SP3485 #2   │── RS485 Ch2
   │          │ Data    │  │4 SM  │  │4 SM  │  │4 SM  │  │  PIO0   │  SP3485 #3   │── RS485 Ch3
   │          ├────┐    │  └──┬───┘  └──┬───┘  └──┬───┘  ├────────►│  SP3485 #4   │── RS485 Ch4
   └──────────┘    │    │     │         │         │       │         └──────────────┘
                   │    │  RS485x2   RS232x2    LCD      │
   ┌──────────┐    │    │                                  │         ┌──────────────┐
   │ SI3404-B │◄───┘    │  ┌──────┐  ┌──────┐  ┌──────┐  │  PIO1   │  MAX3232     │── RS232 Ch1
   │ PD Ctrl  │         │  │ SPI0 │  │ SPI1 │  │I2Cx2 │  ├────────►│              │── RS232 Ch2
   └────┬─────┘         │  └──┬───┘  └──┬───┘  └──┬───┘  │         └──────────────┘
        │               │     │         │         │       │
   ┌────▼─────┐         │  W5500     확장SPI    센서+확장  │         ┌──────────────┐
   │ DC-DC    │         │                                  │  PIO2   │  UART LCD    │
   │ 48V→5V   │         │  USB-C ── Debug/UF2              ├────────►│  (Nextion등)  │
   └────┬─────┘         │  SWD   ── Debug Probe            │         └──────────────┘
        │               │                                  │
   ┌────▼─────┐         └─────────────────────────────────┘
   │ LDO      │              │
   │ 5V→3.3V  ├──────────────┘
   └──────────┘
```

## Specs

| 항목 | 사양 |
|------|------|
| MCU | RP2350B (QFN-80, Dual Cortex-M33/RISC-V, 150MHz) |
| Flash | W25Q128 16MB (QSPI) |
| RAM | 520KB SRAM |
| Ethernet | W5500 (SPI, 10/100 Mbps) |
| 전원 | PoE IEEE 802.3af → 48V→5V DC-DC → 3.3V LDO |
| RS485 | 4채널 (SP3485, 120Ω 종단점퍼, Bias, TVS 보호) |
| RS232 | 2채널 (MAX3232) |
| LCD | UART LCD 포트 (PIO2 하드웨어급 UART) |
| I2C | 2포트 (센서용 + 확장용, 4.7kΩ 풀업) |
| SPI | 2포트 (W5500 전용 SPI0 + 확장 SPI1) |
| ADC | 4채널 (GPIO 26-29) |
| GPIO | 12개 여유 (브레이크아웃 헤더) |
| 디버그 | USB-C (UF2 부트로더) + SWD |
| 보안 | Secure Boot, TrustZone, SHA-256, OTP |

## Pin Map

| GPIO | Function | Interface | Target |
|------|----------|-----------|--------|
| 0-2 | UART0 TX/RX + DE | RS485 Ch1 | SP3485 #1 |
| 4-6 | UART1 TX/RX + DE | RS485 Ch2 | SP3485 #2 |
| 8-10 | PIO0 TX/RX + DE | RS485 Ch3 | SP3485 #3 |
| 11-13 | PIO0 TX/RX + DE | RS485 Ch4 | SP3485 #4 |
| 14-15 | PIO1 TX/RX | RS232 Ch1 | MAX3232 |
| 16-17 | PIO1 TX/RX | RS232 Ch2 | MAX3232 |
| 18-19 | PIO2 TX/RX | UART LCD | LCD Header |
| 20-23 | SPI0 MISO/CS/SCK/MOSI | Ethernet | W5500 |
| 24-25 | INT/RST | Ethernet | W5500 |
| 26-29 | ADC0-3 | ADC | ADC Header |
| 32-33 | I2C0 SDA/SCL | I2C Sensor | Sensor Header |
| 34-35 | I2C1 SDA/SCL | I2C Exp | Expansion Header |
| 40-43 | SPI1 MISO/CS/SCK/MOSI | SPI Exp | Expansion Header |
| 3,7,30-31,36-39,44-47 | GPIO | Expansion | GPIO Header (x12) |

### PIO Allocation

```
PIO0: RS485 Ch3 (SM0+SM1) + RS485 Ch4 (SM2+SM3)
PIO1: RS232 Ch1 (SM0+SM1) + RS232 Ch2 (SM2+SM3)
PIO2: LCD UART (SM0+SM1) + 여유 (SM2+SM3)
```

## Power Tree

```
RJ45 PoE (48V)
 └─ SI3404-B (PD Controller)
      └─ DC-DC (R-78E5.0-1.0) → 5V / 1A
           ├─ LCD (5V)
           └─ AP2112K-3.3 (LDO) → 3.3V / 600mA
                ├─ RP2350B
                ├─ W25Q128 Flash
                ├─ W5500
                ├─ SP3485 x4
                ├─ MAX3232
                └─ I2C/SPI Sensors
```

## Project Structure

```
├── docs/
│   ├── pinmap.csv              # GPIO 핀맵 (CSV)
│   └── schematic_notes.md      # 시트별 회로도 설계 노트 + BOM
├── hardware/
│   └── rp2350b_industrial.kicad_pro  # KiCad 프로젝트
└── firmware/                   # 펌웨어 (TBD)
```

## Reference Designs

- [RP2350 Hardware Design Guide (공식 PDF)](https://datasheets.raspberrypi.com/rp2350/hardware-design-with-rp2350.pdf)
- [RP2350 Datasheet](https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf)
- [RP2350B Dev Board (KiCad, JLCPCB)](https://github.com/jvanderberg/RP2350B-Dev-Board)
- [Adafruit Metro RP2350 PCB](https://github.com/adafruit/Adafruit-Metro-RP2350-PCB)
- [Adafruit Feather RP2350 PCB](https://github.com/adafruit/Adafruit-Feather-RP2350-PCB)
- [RP2350 KiCad PCB Design Tutorial](https://deepbluembedded.com/rp2350-hardware-pcb-design-in-kicad-rp2350-schematic/)

## BOM (Key Components)

| Part | Package | Qty | Note |
|------|---------|-----|------|
| RP2350B | QFN-80 | 1 | MCU |
| W25Q128JVSIQ | SOIC-8 | 1 | 16MB QSPI Flash |
| W5500 | QFP-48 | 1 | Ethernet Controller |
| SI3404-B | SOIC-8 | 1 | PoE PD Controller |
| R-78E5.0-1.0 | SIP-3 | 1 | 48V→5V DC-DC |
| AP2112K-3.3 | SOT-23-5 | 1 | 5V→3.3V LDO |
| SP3485EN | SOIC-8 | 4 | RS485 Transceiver |
| MAX3232ECSE | SOIC-16 | 1 | RS232 Dual Transceiver |
| HR911105A | RJ45 Magjack | 1 | Ethernet + PoE |
| 12MHz Crystal | 3225 | 1 | MCU Clock |
| 25MHz Crystal | 3225 | 1 | W5500 Clock |
| 2.2uH Inductor | 0805 | 1 | Internal SMPS |

## License

MIT
