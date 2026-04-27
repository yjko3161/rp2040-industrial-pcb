# RP2040 Industrial Board - 회로도 설계 노트

## 회로도 시트 구성

```
Sheet 1: Power Supply (PoE PD + DC-DC + LDO)
Sheet 2: MCU RP2040 (Flash, USB, SWD, Crystal, Reset, Boot)
Sheet 3: RS485 4CH (SP3485 x4 + 보호회로)
Sheet 4: RS232 2CH (MAX3232 x1)
Sheet 5: W5500 Ethernet (W5500 + RJ45 Magjack)
Sheet 6: Expansion IO (LCD, I2C, SPI, GPIO 헤더)
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
  RP2040          : 50mA (max)
  W25Q128 Flash   : 25mA (max)
  W5500           : 132mA (max)
  SP3485 x4       : 1.2mA x4 = 5mA
  MAX3232         : 4mA
  I2C 센서         : ~10mA
  ─────────────────────────
  합계            : ~226mA → AP2112K (600mA) 충분

5V 소모:
  3.3V LDO 입력   : ~226mA
  LCD (5V)        : ~100mA
  ─────────────────────────
  합계            : ~326mA → DC-DC 1A 충분
```

---

## Sheet 2: MCU RP2040

### 필수 외부 부품
```
RP2040 (QFN-56, 7x7mm)
  │
  ├─ Crystal: 12MHz (ABM8-272-T3, 12pF 로드)
  │   ├─ XIN (GPIO20 아님, 전용핀)
  │   ├─ XOUT
  │   └─ 로드 캡: 15pF x2 (계산: (2*CL - Cstray) = ~15pF)
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
  ├─ SWD 디버그 헤더 (1.27mm 4핀)
  │   ├─ SWDIO
  │   ├─ SWCLK
  │   ├─ GND
  │   └─ 3V3 (타겟 전원 표시)
  │
  ├─ BOOTSEL 버튼: GPIO → GND (내부 풀업)
  ├─ RESET 버튼: RUN → GND + 10kΩ 풀업 + 100nF 디바운스
  │
  └─ 디커플링 캐패시터:
      ├─ DVDD (1.1V core): 100nF x6 (각 DVDD 핀)
      ├─ IOVDD (3.3V IO): 100nF x3 (각 IOVDD 핀)
      ├─ USB_VDD: 100nF
      ├─ ADC_AVDD: 100nF + 1uF
      └─ VREG_VIN: 1uF + 100nF
```

### RP2040 전원 핀 연결
```
IOVDD (핀 1,10,22,33,42,49): 3.3V + 100nF 각각
DVDD (핀 23,50): 내부 1.1V 레귤레이터 출력 → 100nF 각각
VREG_VIN (핀 44): 3.3V → 1uF + 100nF
VREG_VOUT (핀 45): 1.1V → 100nF + 1uF → DVDD 연결
USB_VDD (핀 48): 3.3V → 100nF
ADC_AVDD (핀 43): 3.3V → 인덕터(100nH) + 100nF + 1uF
```

---

## Sheet 3: RS485 4CH

### 단일 채널 회로 (x4 반복)
```
RP2040                    SP3485EN
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

### 120Ω 종단저항 점퍼
```
  A ──[120Ω]──[0Ω 점퍼]── B
           또는
  A ──[120Ω]──[2핀 헤더]── B  (점퍼캡으로 ON/OFF)
```

### 커넥터 배치
```
  CH1: 3P 터미널 (A, B, GND) - 3.5mm 또는 5.08mm
  CH2: 3P 터미널 (A, B, GND)
  CH3: 3P 터미널 (A, B, GND)
  CH4: 3P 터미널 (A, B, GND)
```

---

## Sheet 4: RS232 2CH

### MAX3232 회로 (1개 IC로 2채널)
```
3.3V ── VCC
         │
        100nF

MAX3232ECSE (SOIC-16)
  T1IN  ← GPIO 14 (RS232 Ch1 TX)
  T1OUT → DB9 핀3 (또는 헤더)
  R1IN  ← DB9 핀2
  R1OUT → GPIO 15 (RS232 Ch1 RX)

  T2IN  ← GPIO 22 (RS232 Ch2 TX)
  T2OUT → DB9 핀3 (또는 헤더)
  R2IN  ← DB9 핀2
  R2OUT → GPIO 23 (RS232 Ch2 RX)

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
RP2040 SPI0              W5500
GPIO 19 (MOSI) ───────── MOSI
GPIO 16 (MISO) ───────── MISO
GPIO 18 (SCK)  ───────── SCLK
GPIO 17 (CS)   ───────── SCSn
GPIO 20 (INT)  ───────── INTn  (10kΩ 풀업 to 3.3V)
GPIO 21 (RST)  ───────── RSTn  (10kΩ 풀업 + 100nF)

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
  핀3: TX (GPIO 24)
  핀4: RX (GPIO 25)
```

### I2C 센서 헤더 (4핀, 2.54mm)
```
  핀1: VCC (3.3V)
  핀2: GND
  핀3: SDA (GPIO 26) ── 4.7kΩ → 3.3V
  핀4: SCL (GPIO 27) ── 4.7kΩ → 3.3V
```

### SPI 확장 헤더 (6핀, 2.54mm)
```
  별도 SPI1 사용 불가 (핀 부족)
  → W5500 SPI0 버스 공유 + 별도 CS
  핀1: VCC (3.3V)
  핀2: GND
  핀3: MOSI (GPIO 19, W5500 공유)
  핀4: MISO (GPIO 16, W5500 공유)
  핀5: SCK  (GPIO 18, W5500 공유)
  핀6: CS   (GPIO 3 또는 7, 별도 CS)
```

### GPIO 확장 헤더
```
  GPIO 3:  범용 (SPI 확장 CS 겸용 가능)
  GPIO 7:  범용
  GPIO 28: ADC 입력 가능 (0~3.3V)
  GPIO 29: ADC 입력 가능 (0~3.3V)
```

---

## BOM (주요 부품)

| # | Part | Package | Qty | Note |
|---|------|---------|-----|------|
| 1 | RP2040 | QFN-56 | 1 | MCU |
| 2 | W25Q128JVSIQ | SOIC-8 | 1 | 16MB QSPI Flash |
| 3 | 12MHz Crystal | 3225 | 1 | ABM8-272-T3 |
| 4 | W5500 | QFP-48 | 1 | Ethernet Controller |
| 5 | 25MHz Crystal | 3225 | 1 | W5500용 |
| 6 | HR911105A | RJ45 Magjack | 1 | 내장 트랜스 + LED |
| 7 | SI3404-B | SOIC-8 | 1 | PoE PD Controller |
| 8 | R-78E5.0-1.0 | SIP-3 | 1 | 48V→5V DC-DC |
| 9 | AP2112K-3.3 | SOT-23-5 | 1 | 5V→3.3V LDO |
| 10 | SP3485EN | SOIC-8 | 4 | RS485 트랜시버 |
| 11 | MAX3232ECSE | SOIC-16 | 1 | RS232 듀얼 트랜시버 |
| 12 | USB-C 커넥터 | SMD | 1 | 디버그/UF2 |
| 13 | PESD5V0S2BT | SOT-23 | 4 | RS485 TVS 보호 |
| 14 | 100nF MLCC | 0402 | ~30 | 디커플링 |
| 15 | 터미널 블록 3P | 3.5mm | 4 | RS485 커넥터 |
| 16 | 2P 헤더+점퍼 | 2.54mm | 4 | 120Ω 종단 선택 |
