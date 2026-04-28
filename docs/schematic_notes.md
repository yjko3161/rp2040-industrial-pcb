# RP2350B Industrial Board - 회로도 설계 노트

> MCU 변경: RP2040 → **RP2350B** (QFN-80, GPIO 48개, PIO 3블록 12SM, Cortex-M33/RISC-V)

## 회로도 시트 구성

```
Sheet 1: Power Supply (PoE PD + DC-DC + LDO)
Sheet 2: MCU RP2350B (Flash, USB, SWD, Crystal, Reset, Boot)
Sheet 3: RS485 4CH (SP3485 x4 + 보호회로)
Sheet 4: RS232 2CH (MAX3232 x1)
Sheet 5: W5500 Ethernet (W5500 + RJ45 Magjack)
Sheet 6: Expansion IO (LCD, I2C x2, SPI, ADC, GPIO 헤더)
```

---

## Sheet 1: Power Supply

### PoE PD 수신부
```
RJ45 (1,2,3,6: Data / 4,5,7,8: PoE Power)
  │
  ├─ 다이오드 브릿지 (내장 or 외부)
  │
  └─ SI3404-B (PoE PD Controller)
       ├─ RCLASS: 24.9kΩ (Class 0, 12.95W)
       ├─ DET: 25kΩ (Detection)
       ├─ VDD: 48V input
       └─ OUT → DC-DC 입력
```

### DC-DC (48V → 5V)
```
R-78E5.0-1.0 (RECOM 절연형) 또는 STS3A48-5V (비절연)
  ├─ VIN: 36~72V (PoE 범위)
  ├─ VOUT: 5V / 1A
  ├─ 입력 C: 10uF/100V ceramic + 47uF/63V electrolytic
  └─ 출력 C: 22uF/10V ceramic x2
```

### LDO (5V → 3.3V)
```
AP2112K-3.3 (600mA, Low Dropout)
  ├─ VIN: 5V
  ├─ VOUT: 3.3V
  ├─ EN: VIN (항상 ON)
  ├─ 입력 C: 1uF ceramic
  └─ 출력 C: 1uF ceramic
```

### 전류 예산
```
3.3V 소모:
  RP2350B         : 85mA (max, M33 듀얼코어)
  W25Q128 Flash   : 25mA (max)
  W5500           : 132mA (max)
  SP3485 x4       : 1.2mA x4 = 5mA
  MAX3232         : 4mA
  I2C 센서         : ~10mA
  ─────────────────────────
  합계            : ~261mA → AP2112K (600mA) 충분

5V 소모:
  3.3V LDO 입력   : ~261mA
  LCD (5V)        : ~100mA
  ─────────────────────────
  합계            : ~361mA → DC-DC 1A 충분
```

---

## Sheet 2: MCU RP2350B

### RP2350B vs RP2040 주요 차이 (설계 영향)
```
패키지: QFN-56 (7x7) → QFN-80 (10x10)  ← 풋프린트 변경 필수
GPIO:   30개 → 48개
PIO:    2블록(8SM) → 3블록(12SM)
SRAM:   264KB → 520KB
코어:   M0+ → M33(FPU) / RISC-V 선택 가능
보안:   없음 → Secure Boot, TrustZone, SHA-256, OTP
전원:   VREG 구조 유사하나 핀 배치 다름 ← 디커플링 재배치 필수
```

### 필수 외부 부품
```
RP2350B (QFN-80, 10x10mm)
  │
  ├─ Crystal: 12MHz (ABM8-272-T3 또는 Abracon 권장)
  │   ├─ XIN (전용핀)
  │   ├─ XOUT
  │   └─ 로드 캡: 15pF x2
  │
  ├─ Flash: W25Q128JVSIQ (16MB, QSPI)
  │   ├─ QSPI_SS  → CS
  │   ├─ QSPI_SD0 → DI (IO0)
  │   ├─ QSPI_SD1 → DO (IO1)
  │   ├─ QSPI_SD2 → WP (IO2) - 10kΩ 풀업
  │   ├─ QSPI_SD3 → HOLD (IO3) - 10kΩ 풀업
  │   ├─ QSPI_SCK → CLK
  │   └─ 디커플링: 100nF
  │
  ├─ USB-C 커넥터
  │   ├─ USB_DP → 27Ω → D+
  │   ├─ USB_DM → 27Ω → D-
  │   ├─ CC1, CC2: 5.1kΩ → GND (Device 식별)
  │   └─ VBUS → 5V (전원은 PoE에서 공급, VBUS는 감지용)
  │
  ├─ SWD 디버그 헤더 (1.27mm 4핀 또는 Tag-Connect)
  │   ├─ SWDIO
  │   ├─ SWCLK
  │   ├─ GND
  │   └─ 3V3 (타겟 전원 표시)
  │
  ├─ BOOTSEL 버튼: QSPI_SS → GND (부트 시 Low 감지)
  ├─ RESET 버튼: RUN → GND + 10kΩ 풀업 + 100nF 디바운스
  │
  └─ 디커플링 캐패시터 (RP2350B 데이터시트 기준):
      ├─ DVDD (1.1V core): 100nF x 각 DVDD 핀 + 벌크 4.7uF x1
      ├─ IOVDD (3.3V IO): 100nF x 각 IOVDD 핀
      ├─ USB_VDD: 100nF + 1uF
      ├─ ADC_AVDD: 100nF + 1uF (인덕터 100nH 분리)
      ├─ VREG_VIN: 3.3V → 1uF + 100nF
      ├─ VREG_VOUT: 1.1V → 1uF + 100nF → DVDD
      └─ 총 디커플링: 100nF x ~15개 + 벌크 캡 다수
         ※ QFN-80은 전원핀이 RP2040보다 많음 → 캡 수 증가
```

### RP2350B 전원 핀 연결 (QFN-80)
```
IOVDD: 3.3V → 각 핀에 100nF (약 6~8핀)
DVDD:  내부 1.1V VREG 출력 → 각 핀에 100nF + 벌크 4.7uF
VREG_VIN: 3.3V 입력
VREG_VOUT: 1.1V 출력 → DVDD 연결
USB_VDD: 3.3V → 100nF + 1uF
ADC_AVDD: 3.3V → 100nH 인덕터 → 100nF + 1uF

※ 정확한 핀 번호는 RP2350B 데이터시트 QFN-80 핀아웃 참조
※ Raspberry Pi 공식 KiCad 레퍼런스 디자인 사용 권장:
   https://datasheets.raspberrypi.com/rp2350/hardware-design-with-rp2350.pdf
```

### RP2350B 내부 SMPS 인덕터
```
VREG_VOUT ─── L (2.2uH, Murata LQH32MN2R2K) ─── DVDD
  ※ RP2350 HW 설계 가이드 권장 인덕터 사용 필수
  ※ 인덕터 극성 방향 주의 (데이터시트 확인)
  ※ JLCPCB 조립 시 방향 재확인
```

---

## Sheet 3: RS485 4CH

### 단일 채널 회로 (x4 반복)
```
RP2350B                   SP3485EN
GPIO_TX ──────────────── DI
GPIO_RX ──────────────── RO
GPIO_DE ──┬───────────── DE
          └───────────── /RE

                          A ──┬── Rbias 560Ω → 3.3V
                              ├── TVS (PESD5V0S2BT)
                              ├── R_term 120Ω (점퍼) ──┐
                              └── 터미널 블록 A          │
                          B ──┬── Rbias 560Ω → GND    │
                              ├── TVS                   │
                              ├── R_term ───────────────┘
                              └── 터미널 블록 B
                        GND ──── 터미널 블록 GND

SP3485EN 전원:
  VCC: 3.3V + 100nF 디커플링
```

### GPIO 할당
```
Ch1: GPIO 0 (TX), GPIO 1 (RX), GPIO 2 (DE)   ← HW UART0
Ch2: GPIO 4 (TX), GPIO 5 (RX), GPIO 6 (DE)   ← HW UART1
Ch3: GPIO 8 (TX), GPIO 9 (RX), GPIO 10 (DE)  ← PIO0 SM0+SM1
Ch4: GPIO 11 (TX), GPIO 12 (RX), GPIO 13 (DE) ← PIO0 SM2+SM3
```

### 120Ω 종단저항 점퍼
```
  A ──[120Ω]──[0Ω 점퍼]── B
           또는
  A ──[120Ω]──[2핀 헤더]── B  (점퍼캡으로 ON/OFF)
```

### 커넥터 배치
```
  CH1~CH4: 3P 터미널 (A, B, GND) - 3.5mm 또는 5.08mm
```

---

## Sheet 4: RS232 2CH

### MAX3232 회로 (1개 IC로 2채널)
```
3.3V ── VCC
         │
        100nF

MAX3232ECSE (SOIC-16)
  T1IN  ← GPIO 14 (RS232 Ch1 TX, PIO1 SM0)
  T1OUT → DB9 핀3 (또는 헤더)
  R1IN  ← DB9 핀2
  R1OUT → GPIO 15 (RS232 Ch1 RX, PIO1 SM1)

  T2IN  ← GPIO 16 (RS232 Ch2 TX, PIO1 SM2)
  T2OUT → DB9 핀3 (또는 헤더)
  R2IN  ← DB9 핀2
  R2OUT → GPIO 17 (RS232 Ch2 RX, PIO1 SM3)

차지펌프 캐패시터 (100nF x5):
  C1+/C1- : 100nF
  C2+/C2- : 100nF
  C3+/C3- : 100nF  (V+)
  C4+/C4- : 100nF  (V-)
  VCC     : 100nF
```

---

## Sheet 5: W5500 Ethernet

```
RP2350B SPI0              W5500
GPIO 23 (MOSI) ───────── MOSI
GPIO 20 (MISO) ───────── MISO
GPIO 22 (SCK)  ───────── SCLK
GPIO 21 (CS)   ───────── SCSn
GPIO 24 (INT)  ───────── INTn  (10kΩ 풀업 to 3.3V)
GPIO 25 (RST)  ───────── RSTn  (10kΩ 풀업 + 100nF)

W5500 전원:
  VCC33A (Analog):  3.3V + 100nF + 10uF
  VCC33D (Digital): 3.3V + 100nF + 10uF
  VCC25A:           내부 2.5V 레귤레이터 출력 → 10uF

W5500 Crystal: 25MHz + 18pF x2 로드캡

RJ45 Magjack (HR911105A 또는 동급):
  ├─ TX+/TX- → W5500 TXP/TXN (직접 연결, 내장 트랜스)
  ├─ RX+/RX- → W5500 RXP/RXN
  ├─ LED: LINKLED, ACTLED → Magjack LED
  └─ PoE: 4,5,7,8 핀 → PD 컨트롤러
```

### RJ45+Magjack 겸용 PoE
```
RJ45 핀 배치:
  1,2 (TX+/-) ──── Magjack 트랜스 ──── W5500 TX
  3,6 (RX+/-) ──── Magjack 트랜스 ──── W5500 RX
  4,5 (PoE V+) ─── 다이오드 브릿지 ──── PD Controller +
  7,8 (PoE V-) ─── 다이오드 브릿지 ──── PD Controller -
```

---

## Sheet 6: Expansion

### UART LCD 헤더 (4핀, 2.54mm)
```
  핀1: VCC (5V)
  핀2: GND
  핀3: TX (GPIO 18, PIO2 SM0)  ← bit-bang 아닌 PIO UART!
  핀4: RX (GPIO 19, PIO2 SM1)
```

### I2C 센서 헤더 (4핀, 2.54mm) - I2C0
```
  핀1: VCC (3.3V)
  핀2: GND
  핀3: SDA (GPIO 32) ── 4.7kΩ → 3.3V
  핀4: SCL (GPIO 33) ── 4.7kΩ → 3.3V
```

### I2C 확장 헤더 (4핀, 2.54mm) - I2C1
```
  핀1: VCC (3.3V)
  핀2: GND
  핀3: SDA (GPIO 34) ── 4.7kΩ → 3.3V
  핀4: SCL (GPIO 35) ── 4.7kΩ → 3.3V
```

### SPI 확장 헤더 (6핀, 2.54mm) - SPI1 독립 포트!
```
  ※ RP2350B는 GPIO 충분 → SPI1 별도 할당 (W5500과 버스 공유 불필요!)
  핀1: VCC (3.3V)
  핀2: GND
  핀3: MOSI (GPIO 43)
  핀4: MISO (GPIO 40)
  핀5: SCK  (GPIO 42)
  핀6: CS   (GPIO 41)
```

### ADC 헤더 (6핀, 2.54mm)
```
  핀1: VCC (3.3V)
  핀2: GND
  핀3: ADC0 (GPIO 26) ── 0~3.3V 아날로그 입력
  핀4: ADC1 (GPIO 27)
  핀5: ADC2 (GPIO 28)
  핀6: ADC3 (GPIO 29)
```

### GPIO 확장 헤더 (2x10, 2.54mm)
```
  GPIO 3:  범용
  GPIO 7:  범용
  GPIO 30: 범용
  GPIO 31: 범용
  GPIO 36: 범용
  GPIO 37: 범용
  GPIO 38: 범용
  GPIO 39: 범용
  GPIO 44: 범용
  GPIO 45: 범용
  GPIO 46: 범용
  GPIO 47: 범용
  총 12개 여유 GPIO → 확장 헤더로 브레이크아웃
```

---

## PIO State Machine 할당

```
PIO0 (4 SM) - RS485 Ch3 + Ch4:
  SM0 → RS485 Ch3 TX (GPIO 8)
  SM1 → RS485 Ch3 RX (GPIO 9)
  SM2 → RS485 Ch4 TX (GPIO 11)
  SM3 → RS485 Ch4 RX (GPIO 12)

PIO1 (4 SM) - RS232 Ch1 + Ch2:
  SM0 → RS232 Ch1 TX (GPIO 14)
  SM1 → RS232 Ch1 RX (GPIO 15)
  SM2 → RS232 Ch2 TX (GPIO 16)
  SM3 → RS232 Ch2 RX (GPIO 17)

PIO2 (4 SM) - LCD UART + 여유 2SM:
  SM0 → LCD TX (GPIO 18)
  SM1 → LCD RX (GPIO 19)
  SM2 → 여유 (향후 확장용)
  SM3 → 여유 (향후 확장용)
```

---

## BOM (주요 부품)

| # | Part | Package | Qty | Note |
|---|------|---------|-----|------|
| 1 | **RP2350B** | **QFN-80 (10x10)** | 1 | MCU (M33/RISC-V, 150MHz) |
| 2 | W25Q128JVSIQ | SOIC-8 | 1 | 16MB QSPI Flash |
| 3 | 12MHz Crystal | 3225 | 1 | Abracon 권장 |
| 4 | **2.2uH 인덕터** | **0805/1008** | **1** | **내부 SMPS용 (RP2350 신규)** |
| 5 | W5500 | QFP-48 | 1 | Ethernet Controller |
| 6 | 25MHz Crystal | 3225 | 1 | W5500용 |
| 7 | HR911105A | RJ45 Magjack | 1 | 내장 트랜스 + LED + PoE |
| 8 | SI3404-B | SOIC-8 | 1 | PoE PD Controller |
| 9 | R-78E5.0-1.0 | SIP-3 | 1 | 48V→5V DC-DC |
| 10 | AP2112K-3.3 | SOT-23-5 | 1 | 5V→3.3V LDO |
| 11 | SP3485EN | SOIC-8 | 4 | RS485 트랜시버 |
| 12 | MAX3232ECSE | SOIC-16 | 1 | RS232 듀얼 트랜시버 |
| 13 | USB-C 커넥터 | SMD | 1 | 디버그/UF2 |
| 14 | PESD5V0S2BT | SOT-23 | 4 | RS485 TVS 보호 |
| 15 | 100nF MLCC | 0402 | **~35** | 디커플링 (QFN-80 핀 증가) |
| 16 | 4.7uF MLCC | 0402 | 2 | DVDD 벌크 캡 |
| 17 | 터미널 블록 3P | 3.5mm | 4 | RS485 커넥터 |
| 18 | 2P 헤더+점퍼 | 2.54mm | 4 | 120Ω 종단 선택 |
| 19 | 2x10 핀 헤더 | 2.54mm | 1 | GPIO 확장 브레이크아웃 |

---

## 레퍼런스 디자인

| 자료 | 용도 |
|------|------|
| [RP2350 HW Design Guide (공식)](https://datasheets.raspberrypi.com/rp2350/hardware-design-with-rp2350.pdf) | 전원, 디커플링, 레이아웃 필수 참고 |
| [RP2350B Dev Board (KiCad)](https://github.com/jvanderberg/RP2350B-Dev-Board) | KiCad 구조, JLCPCB BOM/CPL |
| [Adafruit Metro RP2350](https://github.com/adafruit/Adafruit-Metro-RP2350-PCB) | 완성도 높은 전원부 참고 |
| [RP2350 Datasheet](https://datasheets.raspberrypi.com/rp2350/rp2350-datasheet.pdf) | 핀아웃, 전기적 특성 |
