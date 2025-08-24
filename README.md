# HyperionSAT Flight‑Computer – 하드웨어 (2025)

> **Rev**: 1.0 · **ECAD**: KiCad 9.0.4 · **MCU**: STM32H723VGT6 (550 MHz, DP‑FPU)  
> **역할**: 2차 추진, TVC 제어, 센싱, 통신을 담당하는 하이브리드 비행 컴퓨터

---

## 리포지토리 구성 (스냅샷)
```
docs/
└─ images/
   ├─ sch_processor.png
   ├─ sch_sensors.png
   ├─ sch_pmic.png
   ├─ sch_comm.png
   ├─ sch_actuators.png
   └─ sch_additional.png
README.md
```
> PCB 사진(Top/Bottom)은 준비되면 `docs/images/`에 추가하세요.

---

## 1) 시스템 개요

**블록**
- **Processor**: STM32H723VGT6, 12 MHz TCXO, SWD, EEPROM(AT24C08C), RTC(DS3231M), Buzzer/LED, microSD(SDMMC 4‑bit).
- **Flight Sensors**: ICM‑45686(SPI), ICM‑20948(SPI, 1.8 V IO), MS5611(SPI), MMC5983MA(SPI), MAX‑M10S(UART+PPS), VL53L1X(I²C), **CDS‑Top/Bottom**(Bottom=TLV9001 버퍼).
- **Actuators**: TVC 서보 X/Y(12 V), **리니어 솔레노이드**(IRF1404, 어댑터 분리), **낙하산 전개 전자석**(DRV8871DDA).
- **Communications**: LoRa E32‑433T37S(UART + M0/M1/AUX), **Globalstar STX‑3**(UART + EN/RESET), RF 매칭 네트워크.
- **PMIC/Power**: 메인 Li‑Po 3S(셀1/2 버퍼드 ADC), **점화 배터리**(팩 전압 ADC), 3.3 V 스위칭, **INA219** 방전/점화 전류 측정, **AS6221** 온도.

**점화(IGN)**: **1채널** 구성으로 단순화됨.

---

## 2) 전기적 인터페이스 (네트 라벨 기준)

### SPI
| Bus | Device | Signals |
|---|---|---|
| SPI‑A | ICM‑45686 | `ICM-45686_SCK/MISO/MOSI/CS`, `ICM-45686_INT` |
| SPI‑B | ICM‑20948, MS5611 | `ICM-20948_SCK/MISO/MOSI/CS`, `IMU2_INT`; `Barometer_SCK/MISO/MOSI/CS` |
| SPI‑C | MMC5983MA | `MMC5983MA_SCK/MISO/MOSI/CS`, `MMC5983MA_INT` |

### I²C (7‑bit)
| Bus | Device | Addr |
|---|---|---|
| RTC | DS3231M | 0x68 |
| EEPROM | AT24C08C | 0x50 *(데이터시트 8‑bit 코드 0xA0)* |
| TEMP | AS6221(Board/PMIC/FET/LoRa) | 0x48 / 0x49 / 0x4B / 0x4A |
| IGN | INA219 | 0x41 |

### UART / 타이밍 / GPIO
| Link | Signals |
|---|---|
| Debug | `Debug_Tx`, `Debug_Rx` |
| LoRa(E32) | `RF_Tx`, `RF_Rx`, `RF_M0`, `RF_M1`, `RF_AUX` |
| STX‑3 | `STX-3_Tx`, `STX-3_Rx`, `STX-3_CTS`, `STX-3_RTS`, `STX-3_EN`, `STX-3_Reset` |
| GPS | `GPS_Tx`, `GPS_Rx`, `GPS_PPS` |
| SDMMC | `SDMMC_D0..D3`, `SDMMC_CLK`, `SDMMC_CMD`, `SDMMC_DET`, `SDMMC_WP` |
| Actuators | `Servo_X`, `Servo_Y`, `Solenoid_Gate`, `DRV8871_A`, `DRV8871_B` |
| Analog | `Main_Cell1_ADC`, `Main_Cell2_ADC`, `IGN_BAT_ADC`, `Back-up_BAT_ADC`, `CDS_Top`, `CDS_Bottom_ADC` |

---

## 3) 회로도 스냅샷

- **Processor**  
  <img src="docs/images/sch_processor.png" width="900"/>

- **Flight Sensors**  
  <img src="docs/images/sch_sensors.png" width="900"/>

- **PMIC / Power**  
  <img src="docs/images/sch_pmic.png" width="900"/>

- **Communications**  
  <img src="docs/images/sch_comm.png" width="900"/>

- **Actuators**  
  <img src="docs/images/sch_actuators.png" width="900"/>

- **Additional parts (Magnetometer SPI, CDS Bottom 버퍼, LoRa 온도)**  
  <img src="docs/images/sch_additional.png" width="900"/>

---

## 4) 브링업 체크리스트

1. 육안 검사: 극성/방향/브리지 확인, **폴리머 탄탈 100 µF** 위치 점검.  
2. 전원: 12 V CC 0.2→0.8 A 램프, 3.3 V 리플 < 50 mVp‑p.  
3. I²C 버스별 스캔(RTC/EEPROM/TEMP/IGN).  
4. RTC `SQW/INT` 인터럽트, EEPROM 64 B R/W.  
5. SPI WHOAMI: ICM‑45686 → ICM‑20948 → MS5611 → MMC5983MA.  
6. GPS NMEA 및 1 PPS 확인.  
7. SDMMC 4‑bit 연속 쓰기(≥64 KB), MCU 근처 22 Ω 시리즈 검증.  
8. 액추에이터: 서보 PWM 범위/전류, 솔레노이드 트랜지언트, DRV8871 `ILIM=32 kΩ`.  
9. **점화 1채널**: TC427 → 더미로드, **INA219** 전류 로깅.  
10. LoRa/STX‑3: EN/RESET 순서, UART 링크; STX‑3 π‑매칭은 후속 VNA 튜닝.

---

## 5) 비고

- TXB0106 자동 방향 레벨시프터는 SPI에 한계가 있을 수 있음. 필요 시 고정 방향 트랜슬레이터로 변경 검토.  
- RF: STX‑3 π‑네트워크 값은 초기값. 최종 안테나/케이블로 VNA 재튜닝 권장.  
- ADC: 셀 버퍼 출력은 게인/오프셋 보정 필요. PID 이득과 함께 EEPROM에 저장 권장.

---
