# RP2350B Industrial Board - Netlist (연결표)

> EasyEDA에서 부품 배치 후 이 표 보면서 와이어 연결하면 됨

---

## 1. 전원부

### PoE → DC-DC → LDO
```
RJ45(J1) 4,5번 핀 ──→ SI3404(U6) VDD
RJ45(J1) 7,8번 핀 ──→ SI3404(U6) GND (다이오드 브릿지 경유)
SI3404(U6) OUT     ──→ R-78E(U8) VIN
R-78E(U8) VOUT     ──→ +5V 네트
R-78E(U8) GND      ──→ GND
+5V                ──→ AP2112K(U7) VIN
AP2112K(U7) VOUT   ──→ +3V3 네트
AP2112K(U7) EN     ──→ +5V (항상 ON)
```

### RP2350B 전원
```
+3V3 ──→ RP2350B(U1) 모든 IOVDD 핀 (각각 100nF)
+3V3 ──→ RP2350B(U1) VREG_VIN (1uF + 100nF)
+3V3 ──→ RP2350B(U1) USB_VDD (100nF + 1uF)
+3V3 ──→ 100nH ──→ RP2350B(U1) ADC_AVDD (100nF + 1uF)

RP2350B(U1) VREG_VOUT ──→ L1(2.2uH) ──→ DVDD 네트
DVDD ──→ RP2350B(U1) 모든 DVDD 핀 (각각 100nF + 벌크 4.7uF)
```

---

## 2. MCU RP2350B (U1)

### Crystal
```
Y1(12MHz) ──→ RP2350B(U1) XIN / XOUT
Y1 XIN  ──→ 15pF ──→ GND
Y1 XOUT ──→ 15pF ──→ GND
```

### QSPI Flash (U2: W25Q128)
```
RP2350B QSPI_SS  ──→ W25Q128(U2) CS
RP2350B QSPI_SD0 ──→ W25Q128(U2) DI (IO0)
RP2350B QSPI_SD1 ──→ W25Q128(U2) DO (IO1)
RP2350B QSPI_SD2 ──→ W25Q128(U2) WP (IO2)  + 10kΩ → +3V3
RP2350B QSPI_SD3 ──→ W25Q128(U2) HOLD(IO3) + 10kΩ → +3V3
RP2350B QSPI_SCK ──→ W25Q128(U2) CLK
W25Q128(U2) VCC  ──→ +3V3 + 100nF
```

### USB-C (J2)
```
RP2350B USB_DP ──→ 27Ω ──→ J2 D+
RP2350B USB_DM ──→ 27Ω ──→ J2 D-
J2 CC1 ──→ 5.1kΩ ──→ GND
J2 CC2 ──→ 5.1kΩ ──→ GND
J2 VBUS ──→ +5V (감지용, 전원은 PoE)
J2 GND  ──→ GND
```

### SWD Debug (HDR6)
```
RP2350B SWDIO  ──→ HDR6 핀1
RP2350B SWCLK  ──→ HDR6 핀2
GND            ──→ HDR6 핀3
+3V3           ──→ HDR6 핀4
```

### 버튼
```
RP2350B QSPI_SS ──→ SW1(BOOTSEL) ──→ GND    (부트 시 Low)
RP2350B RUN     ──→ SW2(RESET) ──→ GND      + 10kΩ → +3V3 + 100nF
```

---

## 3. RS485 4채널

### Ch1 (HW UART0)
```
RP2350B GPIO0  ──→ SP3485(U4-1) DI
RP2350B GPIO1  ──→ SP3485(U4-1) RO
RP2350B GPIO2  ──→ SP3485(U4-1) DE + /RE (묶음)
SP3485(U4-1) A ──→ 560Ω → +3V3 (bias)
SP3485(U4-1) A ──→ PESD5V0S2BT(D1)
SP3485(U4-1) A ──→ 120Ω ──→ JP1 ──→ SP3485(U4-1) B  (종단)
SP3485(U4-1) B ──→ 560Ω → GND (bias)
SP3485(U4-1) B ──→ PESD5V0S2BT(D1)
SP3485(U4-1) A ──→ TB1 핀1
SP3485(U4-1) B ──→ TB1 핀2
GND            ──→ TB1 핀3
SP3485(U4-1) VCC ──→ +3V3 + 100nF
```

### Ch2 (HW UART1) — 동일 패턴
```
GPIO4 → U4-2 DI, GPIO5 → U4-2 RO, GPIO6 → U4-2 DE+/RE
U4-2 A/B → bias + TVS(D2) + 120Ω+JP2 + TB2
```

### Ch3 (PIO0) — 동일 패턴
```
GPIO8 → U4-3 DI, GPIO9 → U4-3 RO, GPIO10 → U4-3 DE+/RE
U4-3 A/B → bias + TVS(D3) + 120Ω+JP3 + TB3
```

### Ch4 (PIO0) — 동일 패턴
```
GPIO11 → U4-4 DI, GPIO12 → U4-4 RO, GPIO13 → U4-4 DE+/RE
U4-4 A/B → bias + TVS(D4) + 120Ω+JP4 + TB4
```

---

## 4. RS232 2채널

### MAX3232 (U5) 차지펌프 캐패시터
```
U5 C1+ ──→ 100nF ──→ U5 C1-
U5 C2+ ──→ 100nF ──→ U5 C2-
U5 V+  ──→ 100nF ──→ GND
U5 V-  ──→ 100nF ──→ GND
U5 VCC ──→ +3V3 + 100nF
```

### Ch1 (PIO1)
```
RP2350B GPIO14 ──→ MAX3232(U5) T1IN
MAX3232(U5) T1OUT ──→ RS232 커넥터/헤더 TX
RS232 커넥터/헤더 RX ──→ MAX3232(U5) R1IN
MAX3232(U5) R1OUT ──→ RP2350B GPIO15
```

### Ch2 (PIO1)
```
RP2350B GPIO16 ──→ MAX3232(U5) T2IN
MAX3232(U5) T2OUT ──→ RS232 커넥터/헤더 TX
RS232 커넥터/헤더 RX ──→ MAX3232(U5) R2IN
MAX3232(U5) R2OUT ──→ RP2350B GPIO17
```

---

## 5. W5500 Ethernet (U3)

### SPI0 연결
```
RP2350B GPIO23 ──→ W5500(U3) MOSI
RP2350B GPIO20 ──→ W5500(U3) MISO
RP2350B GPIO22 ──→ W5500(U3) SCLK
RP2350B GPIO21 ──→ W5500(U3) SCSn
RP2350B GPIO24 ──→ W5500(U3) INTn  + 10kΩ → +3V3
RP2350B GPIO25 ──→ W5500(U3) RSTn  + 10kΩ → +3V3 + 100nF → GND
```

### W5500 전원
```
+3V3 ──→ W5500(U3) VCC33A + 100nF + 10uF
+3V3 ──→ W5500(U3) VCC33D + 100nF + 10uF
W5500(U3) VCC25A ──→ 10uF → GND (내부 2.5V 출력)
```

### W5500 Crystal
```
Y2(25MHz) ──→ W5500(U3) XI / XO
Y2 XI  ──→ 18pF ──→ GND
Y2 XO  ──→ 18pF ──→ GND
```

### RJ45 Magjack (J1)
```
W5500(U3) TXP ──→ J1(HR911105A) TX+
W5500(U3) TXN ──→ J1(HR911105A) TX-
W5500(U3) RXP ──→ J1(HR911105A) RX+
W5500(U3) RXN ──→ J1(HR911105A) RX-
W5500(U3) LINKLED ──→ J1 LED1
W5500(U3) ACTLED  ──→ J1 LED2
```

---

## 6. 확장부

### LCD UART (PIO2)
```
RP2350B GPIO18 ──→ HDR3 핀3 (TX)
RP2350B GPIO19 ──→ HDR3 핀4 (RX)
+5V            ──→ HDR3 핀1 (VCC)
GND            ──→ HDR3 핀2 (GND)
```

### I2C0 센서
```
RP2350B GPIO32 ──→ HDR2-1 핀3 (SDA) + 4.7kΩ → +3V3
RP2350B GPIO33 ──→ HDR2-1 핀4 (SCL) + 4.7kΩ → +3V3
+3V3           ──→ HDR2-1 핀1 (VCC)
GND            ──→ HDR2-1 핀2 (GND)
```

### I2C1 확장
```
RP2350B GPIO34 ──→ HDR2-2 핀3 (SDA) + 4.7kΩ → +3V3
RP2350B GPIO35 ──→ HDR2-2 핀4 (SCL) + 4.7kΩ → +3V3
+3V3           ──→ HDR2-2 핀1 (VCC)
GND            ──→ HDR2-2 핀2 (GND)
```

### SPI1 확장
```
RP2350B GPIO43 ──→ HDR4 핀3 (MOSI)
RP2350B GPIO40 ──→ HDR4 핀4 (MISO)
RP2350B GPIO42 ──→ HDR4 핀5 (SCK)
RP2350B GPIO41 ──→ HDR4 핀6 (CS)
+3V3           ──→ HDR4 핀1 (VCC)
GND            ──→ HDR4 핀2 (GND)
```

### ADC
```
RP2350B GPIO26 ──→ HDR5 핀3 (ADC0)
RP2350B GPIO27 ──→ HDR5 핀4 (ADC1)
RP2350B GPIO28 ──→ HDR5 핀5 (ADC2)
RP2350B GPIO29 ──→ HDR5 핀6 (ADC3)
+3V3           ──→ HDR5 핀1 (VCC)
GND            ──→ HDR5 핀2 (GND)
```

### GPIO 확장 (HDR1: 2x10)
```
RP2350B GPIO3  ──→ HDR1
RP2350B GPIO7  ──→ HDR1
RP2350B GPIO30 ──→ HDR1
RP2350B GPIO31 ──→ HDR1
RP2350B GPIO36 ──→ HDR1
RP2350B GPIO37 ──→ HDR1
RP2350B GPIO38 ──→ HDR1
RP2350B GPIO39 ──→ HDR1
RP2350B GPIO44 ──→ HDR1
RP2350B GPIO45 ──→ HDR1
RP2350B GPIO46 ──→ HDR1
RP2350B GPIO47 ──→ HDR1
+3V3           ──→ HDR1 (VCC x2)
GND            ──→ HDR1 (GND x2)
```
