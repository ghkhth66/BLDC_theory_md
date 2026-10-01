# RX26T Fan Drive EEPROM 변수 주소 및 스케일 변환표

## 1. 문서 목적

이 문서는 RX26T Fan Drive 프로젝트의 다음 자료를 기준으로 EEPROM 옵션 데이터와 실행 중 RAM 변수의 관계를 정리한다.

- `DD_EEPROM.c`의 `roption[]` 매핑
- MAP 파일의 RAM 심벌 주소
- EEPROM 원시값 조립 및 스케일 변환식

현재 확인된 구조는 다음과 같다.

```text
EEPROM_START_ADDRESS + Offset
            ↓
       roption[Offset]
            ↓
   초기화 함수의 변환/조립
            ↓
      실행 중 RAM 변수
```

> 주의: 표의 EEPROM 주소는 `roption[]` 기준 오프셋이다. 실제 절대주소는 `EEPROM_START_ADDRESS + Offset`으로 계산한다.

---

## 2. 기본 데이터 해석 규칙

### 2.1 1바이트 값

```c
Value = roption[offset];
```

### 2.2 2바이트 값

대부분 상위 바이트가 먼저 저장되는 방식이다.

```c
Value = ((unsigned short)roption[offset] << 8)
      + roption[offset + 1];
```

즉 EEPROM 배열 관점에서는 Big-endian 순서이다.

### 2.3 전류 스케일

전류 제한 항목은 다음 방식으로 내부 count로 변환된다.

```c
CurrentCount = EEPROM_Raw * OP_Iscale / 10;
```

현재 프로젝트 기준 `OP_Iscale = 2048`이고, 내부 전류 count 환산식이 다음과 같다면:

```text
I[A] = CurrentCount / 2048
```

최종 물리 전류는 다음과 같이 단순화된다.

```text
I[A] = EEPROM_Raw / 10
```

### 2.4 EEPROM 절대주소

```text
절대주소 = EEPROM_START_ADDRESS + EEPROM Offset
```

`EEPROM_START_ADDRESS`가 아직 확정되지 않은 경우 표의 오프셋만으로 절대주소를 단정하면 안 된다.

---

## 3. 기본 및 스케일 파라미터

| 변수 | RAM 주소 | EEPROM Offset | 크기 | RAM 계산식 | 스케일/의미 |
|---|---:|---:|---:|---|---|
| `OP_Inverter_USP.byte` | `0x0707` | `0x00` | 1B | `EEP[00]` | 비트 옵션 |
| `OP_fan_option` | `0x0708` | `0x01` | 1B | `EEP[01]` | Fan 옵션 |
| `OP_DC_LINK_LOW_VOLTAGE` | `0x09B4` | `0x03` | 1B | `EEP[03] * 10` | 저전압 기준, 코드상 ×10 |
| `PWM_KHZ` | MAP 일부에서 미확인 | `0x0E` | 1B | `EEP[0E]` | PWM 주파수 기준값 |
| `PWM_KHZ_Main` | `0x0A3A` | `0x0E` | 1B | `EEP[0E]` | PWM 주파수, 코드 명칭상 kHz |
| `OP_Vscale` | `0x09D6` | `0x0F` | 1B | `EEP[0F]` | 내부 전압 스케일 |
| `OP_Iscale` | `0x09D4` | `0x10~0x11` | 2B | `(EEP[10] << 8) + EEP[11]` | 내부 전류 스케일, 프로젝트 기준값 2048 |
| `OP_Iscale_Transfer` | `0x0A2A` | `0x12~0x13` | 2B | `(EEP[12] << 8) + EEP[13]` | 전류 변환 스케일 |
| `OP_VdcScale` | `0x09D8` | `0x14` | 1B | `EEP[14]` | DC Link 전압 스케일 |
| `OP_VdcScale_Transfer` | `0x09DA` | `0x15` | 1B | `EEP[15]` | DC Link 변환 스케일 |

---

## 4. 모터 파라미터

| 변수 | RAM 주소 | EEPROM Offset | 크기 | RAM 계산식 | 형식/의미 |
|---|---:|---:|---:|---|---|
| `OP_Rs` | `0x09B6` | `0x16~0x17` | 2B | `(EEP[16] << 8) + EEP[17]` | 코드 주석상 Q11 |
| `OP_Ld` | `0x09B8` | `0x18~0x19` | 2B | `(EEP[18] << 8) + EEP[19]` | 코드 주석상 Q13 |
| `OP_Lq` | `0x09BA` | `0x1A~0x1B` | 2B | `(EEP[1A] << 8) + EEP[1B]` | 코드 주석상 Q13 |
| `OP_KE` | `0x09BC` | `0x1C~0x1D` | 2B | `(EEP[1C] << 8) + EEP[1D]` | 코드 주석상 Q13 |
| `FreqScale_fm` | MAP 일부에서 미확인 | `0x1E` | 1B | `FreqScale * EEP[1E]` | 주파수 스케일 |
| `OP_L[j].Ipk` | `OP_L` 시작 `0x0B5A` | `0x20` | 1B | `j * EEP[20] * OP_Iscale / 10` | `j=1~9`; 물리 전류는 대략 `j * EEP[20]/10 A` |
| `OP_L[0~1].Lq` | `OP_L` 구조체 내 | `0x21` | 1B | `EEP[21] * 10` | Lq 테이블 |
| `OP_L[2].Lq` | `OP_L` 구조체 내 | `0x22` | 1B | `EEP[22] * 10` | Lq 테이블 |
| `OP_L[3].Lq` | `OP_L` 구조체 내 | `0x23` | 1B | `EEP[23] * 10` | Lq 테이블 |
| `OP_L[4].Lq` | `OP_L` 구조체 내 | `0x24` | 1B | `EEP[24] * 10` | Lq 테이블 |
| `OP_L[5].Lq` | `OP_L` 구조체 내 | `0x25` | 1B | `EEP[25] * 10` | Lq 테이블 |
| `OP_L[6].Lq` | `OP_L` 구조체 내 | `0x26` | 1B | `EEP[26] * 10` | Lq 테이블 |
| `OP_L[7].Lq` | `OP_L` 구조체 내 | `0x27` | 1B | `EEP[27] * 10` | Lq 테이블 |
| `OP_L[8].Lq` | `OP_L` 구조체 내 | `0x28` | 1B | `EEP[28] * 10` | Lq 테이블 |
| `OP_L[9].Lq` | `OP_L` 구조체 내 | `0x29` | 1B | `EEP[29] * 10` | Lq 테이블 |
| `OP_Pre_FW_Time` | `0x0A30` | `0x2A~0x2B` | 2B | `(EEP[2A] << 8) + EEP[2B]` | FW 진입 전 시간 관련 값 |
| `OP_Limit_del_Theta` | `0x0A22` | `0x30` | 1B | `EEP[30]` | Δθ 제한값 |

`Origin` 변수는 별도 EEPROM 영역을 읽지 않고 초기 로딩값을 복사한다.

| 변수 | RAM 주소 | 값 |
|---|---:|---|
| `OP_Rs_Origin` | `0x09BE` | `OP_Rs` 복사 |
| `OP_Ld_Origin` | `0x09C0` | `OP_Ld` 복사 |
| `OP_Lq_Origin` | `0x09C2` | `OP_Lq` 복사 |
| `OP_KE_Origin` | `0x09C4` | `OP_KE` 복사 |

---

## 5. 전압 및 전류 제한 파라미터

| 변수 | RAM 주소 | EEPROM Offset | 크기 | 내부 계산식 | 물리값/설명 |
|---|---:|---:|---:|---|---|
| `OP_INV_Limit.Vde` | `0x09D0` | `0x32` | 1B | `EEP[32] * 10 * OP_Vscale` | d축 전압 제한 내부 count |
| `OP_INV_Limit.Vqe` | `0x09D2` | `0x33` | 1B | `EEP[33] * 10 * OP_Vscale` | q축 전압 제한 내부 count |
| `OP_INV_Limit.Ide` | `0x09CA` | `0x34` | 1B | `EEP[34] * OP_Iscale / 10` | `EEP[34]/10 A` 기준 |
| `OP_INV_Limit.Iqe` | `0x09CC` | `0x35` | 1B | `EEP[35] * OP_Iscale / 10` | `EEP[35]/10 A` 기준 |
| `OP_INV_Limit.Imag` | `0x09CE` | `0x36` | 1B | `EEP[36] * OP_Iscale / 10` | `EEP[36]/10 A` 기준 |
| `OP_INV_Limit.PreImag` | `0x09C8` | `0x37` | 1B | `EEP[37] * OP_Iscale / 10` | `EEP[37]/10 A` 기준 |
| `OP_INV_Limit.OP_Iabcs` | `0x09C6` | `0x38` | 1B | `EEP[38] * OP_Iscale / 10` | 상전류 제한, `EEP[38]/10 A` 기준 |
| `OP_Start_Fail_Imag_Curr` | `0x0A24` | `0x39` | 1B | `EEP[39] * OP_Iscale / 10` | 기동 실패 전류, `EEP[39]/10 A` 기준 |
| `OP_Start_Fail_Freq` | `0x0A26` | `0x3A` | 1B | `EEP[3A]` | 코드 주석에 `50rpm`; 정확한 환산식은 사용처 확인 필요 |

### 전류 변환 예

`OP_Iscale = 2048`, `EEP[39] = 15`일 때:

```text
내부 count = 15 * 2048 / 10
           = 3072

전류[A] = 3072 / 2048
        = 1.5 A
```

---

## 6. Align 파라미터

| 변수 | RAM 주소 | EEPROM Offset | 크기 | 계산식 | 스케일/의미 |
|---|---:|---:|---:|---|---|
| `OP_Align_Time` | `0x070A` | `0x3C` | 1B | `EEP[3C]` | Align 시간, 시간 단위는 사용처 확인 필요 |
| `Align_Current_Step[0]` | `0x0A18`부터 | `0x3D` | 1B 원본 | `EEP[3D]` | 기준상 `EEP[3D]/10 A` |
| `Align_Current_Step[1]` | `0x0A1A` 추정 | `0x3D, 0x3E` | 계산값 | `EEP[3D] + EEP[3E]` | 기준상 결과/10 A |
| `Align_Current_Step[2]` | `0x0A1C` 추정 | `0x3D, 0x3E` | 계산값 | `EEP[3D] + 2*EEP[3E]` | 기준상 결과/10 A |
| `OP_Align_Current` | `0x0A16` | `0x3D` | 1B 원본 | `Align_Current_Step[0]` | `EEP[3D]/10 A` |
| `OP_Kp_Align` | `0x0A14` | `0x9C~0x9D` | 2B | `(EEP[9C] << 8) + EEP[9D]` | Align 전류제어 P 이득 |
| `OP_Ki_Align` | `0x0709` | `0x9E` | 1B | `EEP[9E]` | Align 전류제어 I 이득 |

### Align 전류 스텝 예

`EEP[3D] = 12`, `EEP[3E] = 3`일 때:

```text
Step 0 = 12       → 1.2 A
Step 1 = 12 + 3   → 1.5 A
Step 2 = 12 + 6   → 1.8 A
```

---

## 7. Lead Angle 파라미터

전류 기준점은 `OP_Iscale`을 적용하며, 각도값은 EEPROM 1바이트를 그대로 대입한다.

| 항목 | EEPROM Offset | 계산식 | 해석 |
|---|---:|---|---|
| `OP_Iopt[1]` | `0x40` | `EEP[40] * OP_Iscale / 10` | 기준상 `EEP[40]/10 A` |
| `OP_Aopt[1]` | `0x41` | `EEP[41]` | 각도 테이블 값 |
| `OP_Iopt[2]` | `0x42` | `EEP[42] * OP_Iscale / 10` | 기준상 `EEP[42]/10 A` |
| `OP_Aopt[2]` | `0x43` | `EEP[43]` | 각도 테이블 값 |
| `OP_Iopt[3]` | `0x44` | `EEP[44] * OP_Iscale / 10` | 기준상 `EEP[44]/10 A` |
| `OP_Aopt[3]` | `0x45` | `EEP[45]` | 각도 테이블 값 |
| `OP_Iopt[4]` | `0x46` | `EEP[46] * OP_Iscale / 10` | 기준상 `EEP[46]/10 A` |
| `OP_Aopt[4]` | `0x47` | `EEP[47]` | 각도 테이블 값 |
| `OP_Iopt[5]` | `0x48` | `EEP[48] * OP_Iscale / 10` | 기준상 `EEP[48]/10 A` |
| `OP_Aopt[5]` | `0x49` | `EEP[49]` | 각도 테이블 값 |
| `OP_Iopt[6]` | `0x4A` | `EEP[4A] * OP_Iscale / 10` | 기준상 `EEP[4A]/10 A` |
| `OP_Aopt[6]` | `0x4B` | `EEP[4B]` | 각도 테이블 값 |
| `OP_Iopt[7]` | `0x4C` | `EEP[4C] * OP_Iscale / 10` | 기준상 `EEP[4C]/10 A` |
| `OP_Aopt[7]` | `0x4D` | `EEP[4D]` | 각도 테이블 값 |
| `OP_Iopt[8]` | `0x4E` | `EEP[4E] * OP_Iscale / 10` | 기준상 `EEP[4E]/10 A` |
| `OP_Aopt[8]` | `0x4F` | `EEP[4F]` | 각도 테이블 값 |
| `OP_Iopt[9]` | `0x50` | `EEP[50] * OP_Iscale / 10` | 기준상 `EEP[50]/10 A` |
| `OP_Aopt[9]` | `0x51` | `EEP[51]` | 각도 테이블 값 |
| `OP_Iopt[10]` | `0x52` | `EEP[52] * OP_Iscale / 10` | 기준상 `EEP[52]/10 A` |
| `OP_Aopt[10]` | `0x53` | `EEP[53]` | 각도 테이블 값 |
| `OP_Iopt[11]` | `0x54` | `EEP[54] * OP_Iscale / 10` | 기준상 `EEP[54]/10 A` |
| `OP_Aopt[11]` | `0x55` | `EEP[55]` | 각도 테이블 값 |
| `OP_Iopt[12]` | `0x56` | `EEP[56] * OP_Iscale / 10` | 기준상 `EEP[56]/10 A` |
| `OP_Aopt[12]` | `0x57` | `EEP[57]` | 각도 테이블 값 |
| `OP_Iopt[13]` | `0x58` | `EEP[58] * OP_Iscale / 10` | 기준상 `EEP[58]/10 A` |
| `OP_Aopt[13]` | `0x59` | `EEP[59]` | 각도 테이블 값 |
| `OP_Limit_LeadAngle_FW` | `0x5A` | `EEP[5A]` | FW Lead Angle 제한 |
| `OP_Limit_LeadAngle_Cont` | `0x5B` | `EEP[5B]` | 연속 Lead Angle 제한 |

`OP_Aopt[]`의 실제 각도 단위는 해당 변수를 사용하는 각도 변환 코드와 함께 확인해야 한다.

---

## 8. 속도 기울기, Cutoff 및 오차 제한

| 변수 | RAM 주소 | EEPROM Offset | 크기 | 계산식 | 의미 |
|---|---:|---:|---:|---|---|
| `OP_Slope_Start` | `0x0A20` | `0x65~0x66` | 2B | `(EEP[65] << 8) + EEP[66]` | 속도 상승 기울기 |
| `OP_INV_Gain[1~9].gain_zone_Hz` | `OP_INV_Gain` 내 | `0x73~0x7B` | 각 1B | `EEP[offset]` | Gain zone 기준 주파수 |
| `OP_INV_Gain[0~9].Cutoff_Hz_We` | `OP_INV_Gain` 내 | `0x7C~0x85` | 각 1B | `EEP[offset]` | 속도 Cutoff Hz |
| `OP_INV_Gain[0~9].Cutoff_Hz_Curr` | `OP_INV_Gain` 내 | `0x86~0x8F` | 각 1B | `EEP[offset]` | 전류 Cutoff Hz |
| `OP_SpeedErrorLimit` | `0x0A32` | `0x90` | 1B | `EEP[90] * 100` | 속도 오차 제한 |
| `OP_CurrentErrorLimit` | `0x0A34` | `0x91` | 1B | `EEP[91] * 100` | 전류 오차 제한 |

---

## 9. 전류 Iq 제어 게인

### 9.1 `KpIqe`

각 값은 2바이트 Big-endian이다.

| 배열 항목 | EEPROM Offset | 계산식 |
|---|---:|---|
| `OP_INV_Gain[0].KpIqe` | `0x9F~0xA0` | `(EEP[9F] << 8) + EEP[A0]` |
| `OP_INV_Gain[1].KpIqe` | `0xA1~0xA2` | `(EEP[A1] << 8) + EEP[A2]` |
| `OP_INV_Gain[2].KpIqe` | `0xA3~0xA4` | `(EEP[A3] << 8) + EEP[A4]` |
| `OP_INV_Gain[3].KpIqe` | `0xA5~0xA6` | `(EEP[A5] << 8) + EEP[A6]` |
| `OP_INV_Gain[4].KpIqe` | `0xA7~0xA8` | `(EEP[A7] << 8) + EEP[A8]` |
| `OP_INV_Gain[5].KpIqe` | `0xA9~0xAA` | `(EEP[A9] << 8) + EEP[AA]` |
| `OP_INV_Gain[6].KpIqe` | `0xAB~0xAC` | `(EEP[AB] << 8) + EEP[AC]` |
| `OP_INV_Gain[7].KpIqe` | `0xAD~0xAE` | `(EEP[AD] << 8) + EEP[AE]` |
| `OP_INV_Gain[8].KpIqe` | `0xAF~0xB0` | `(EEP[AF] << 8) + EEP[B0]` |
| `OP_INV_Gain[9].KpIqe` | `0xB1~0xB2` | `(EEP[B1] << 8) + EEP[B2]` |

### 9.2 `KiIqe`

| 배열 범위 | EEPROM Offset | 계산식 |
|---|---:|---|
| `OP_INV_Gain[0~9].KiIqe` | `0xB3~0xBC` | 각 항목 `EEP[offset]` |

---

## 10. 전류 Id 제어 게인

### 10.1 `KpIde`

| 배열 항목 | EEPROM Offset | 계산식 |
|---|---:|---|
| `OP_INV_Gain[0].KpIde` | `0xBD~0xBE` | `(EEP[BD] << 8) + EEP[BE]` |
| `OP_INV_Gain[1].KpIde` | `0xBF~0xC0` | `(EEP[BF] << 8) + EEP[C0]` |
| `OP_INV_Gain[2].KpIde` | `0xC1~0xC2` | `(EEP[C1] << 8) + EEP[C2]` |
| `OP_INV_Gain[3].KpIde` | `0xC3~0xC4` | `(EEP[C3] << 8) + EEP[C4]` |
| `OP_INV_Gain[4].KpIde` | `0xC5~0xC6` | `(EEP[C5] << 8) + EEP[C6]` |
| `OP_INV_Gain[5].KpIde` | `0xC7~0xC8` | `(EEP[C7] << 8) + EEP[C8]` |
| `OP_INV_Gain[6].KpIde` | `0xC9~0xCA` | `(EEP[C9] << 8) + EEP[CA]` |
| `OP_INV_Gain[7].KpIde` | `0xCB~0xCC` | `(EEP[CB] << 8) + EEP[CC]` |
| `OP_INV_Gain[8].KpIde` | `0xCD~0xCE` | `(EEP[CD] << 8) + EEP[CE]` |
| `OP_INV_Gain[9].KpIde` | `0xCF~0xD0` | `(EEP[CF] << 8) + EEP[D0]` |

### 10.2 `KiIde`

| 배열 범위 | EEPROM Offset | 계산식 |
|---|---:|---|
| `OP_INV_Gain[0~9].KiIde` | `0xD1~0xDA` | 각 항목 `EEP[offset]` |

---

## 11. 속도 제어 게인

### 11.1 `KpWe`

| 배열 항목 | EEPROM Offset | 계산식 |
|---|---:|---|
| `OP_INV_Gain[0].KpWe` | `0xDB~0xDC` | `(EEP[DB] << 8) + EEP[DC]` |
| `OP_INV_Gain[1].KpWe` | `0xDD~0xDE` | `(EEP[DD] << 8) + EEP[DE]` |
| `OP_INV_Gain[2].KpWe` | `0xDF~0xE0` | `(EEP[DF] << 8) + EEP[E0]` |
| `OP_INV_Gain[3].KpWe` | `0xE1~0xE2` | `(EEP[E1] << 8) + EEP[E2]` |
| `OP_INV_Gain[4].KpWe` | `0xE3~0xE4` | `(EEP[E3] << 8) + EEP[E4]` |
| `OP_INV_Gain[5].KpWe` | `0xE5~0xE6` | `(EEP[E5] << 8) + EEP[E6]` |
| `OP_INV_Gain[6].KpWe` | `0xE7~0xE8` | `(EEP[E7] << 8) + EEP[E8]` |
| `OP_INV_Gain[7].KpWe` | `0xE9~0xEA` | `(EEP[E9] << 8) + EEP[EA]` |
| `OP_INV_Gain[8].KpWe` | `0xEB~0xEC` | `(EEP[EB] << 8) + EEP[EC]` |
| `OP_INV_Gain[9].KpWe` | `0xED~0xEE` | `(EEP[ED] << 8) + EEP[EE]` |

### 11.2 `KiWe`

| 배열 범위 | EEPROM Offset | 계산식 |
|---|---:|---|
| `OP_INV_Gain[0~9].KiWe` | `0xEF~0xF8` | 각 항목 `EEP[offset]` |

---

## 12. Sensorless Observer 게인

### 12.1 `Ke`

| 배열 범위 | EEPROM Offset | 크기/항목 | 계산식 |
|---|---:|---:|---|
| `OP_INV_Gain[0~9].Ke` | `0xF9~0x10C` | 각 2B | 각 항목 `(High << 8) + Low` |

세부 주소:

```text
Ke[0] : 0x0F9~0x0FA
Ke[1] : 0x0FB~0x0FC
Ke[2] : 0x0FD~0x0FE
Ke[3] : 0x0FF~0x100
Ke[4] : 0x101~0x102
Ke[5] : 0x103~0x104
Ke[6] : 0x105~0x106
Ke[7] : 0x107~0x108
Ke[8] : 0x109~0x10A
Ke[9] : 0x10B~0x10C
```

### 12.2 `Ko`

| 배열 범위 | EEPROM Offset | 크기/항목 | 계산식 |
|---|---:|---:|---|
| `OP_INV_Gain[0~9].Ko` | `0x10D~0x120` | 각 2B | 각 항목 `(High << 8) + Low` |

세부 주소:

```text
Ko[0] : 0x10D~0x10E
Ko[1] : 0x10F~0x110
Ko[2] : 0x111~0x112
Ko[3] : 0x113~0x114
Ko[4] : 0x115~0x116
Ko[5] : 0x117~0x118
Ko[6] : 0x119~0x11A
Ko[7] : 0x11B~0x11C
Ko[8] : 0x11D~0x11E
Ko[9] : 0x11F~0x120
```

### 12.3 EEMF/PLL 필터 및 게인

| 변수 | RAM 주소 | EEPROM Offset | 크기 | 계산식 | 의미 |
|---|---:|---:|---:|---|---|
| `OP_Filter_emf_ob` | `0x078C` | `0x121` | 1B | `EEP[121]` | EEMF Observer 필터 |
| `OP_Filter_wr` | `0x078D` | `0x122` | 1B | `EEP[122]` | 속도 필터, 코드 주석에 `120 = Q7` |
| `OP_Kp_mori` | MAP 일부에서 미확인 | `0x123~0x124` | 2B | `(EEP[123] << 8) + EEP[124]` | Morimoto PLL P 게인 |

---

## 13. I/F 기동 및 전환 파라미터

| 변수 | RAM 주소 | EEPROM Offset | 크기 | 계산식 | 스케일/의미 |
|---|---:|---:|---:|---|---|
| `OP_IF_START` | `0x079C` | `0x136` | 1B | `EEP[136]` | I/F 기동 옵션 |
| `IF_transition_Rpm_Start` | MAP 일부에서 미확인 | `0x137` | 1B | `EEP[137]` | I/F→FOC 전환 시작 기준, rpm 환산은 사용처 확인 |
| `IF_transition_Rpm_End` | MAP 일부에서 미확인 | `0x138` | 1B | `EEP[138]` | I/F→FOC 전환 종료 기준, rpm 환산은 사용처 확인 |
| `IF_B_WeRef_Kf_A` | `0x0791` | `0x139` | 1B | `EEP[139]` | I/F 속도기울기 계수 A |
| `IF_B_WeRef_Kf_B` | `0x0792` | `0x13A` | 1B | `EEP[13A]` | I/F 속도기울기 계수 B |
| `IF_ImagRef_Step[0]` | MAP 일부에서 미확인 | `0x13B` | 계산값 | `EEP[13B]` | 기준상 `/10 A` 검토 |
| `IF_ImagRef_Step[1]` | MAP 일부에서 미확인 | `0x13B,0x13C` | 계산값 | `EEP[13B] + EEP[13C]` | 기준상 결과 `/10 A` 검토 |
| `IF_ImagRef_Step[2]` | MAP 일부에서 미확인 | `0x13B,0x13C` | 계산값 | `EEP[13B] + 2*EEP[13C]` | 기준상 결과 `/10 A` 검토 |
| `IF_B_WeRef_start` | `0x0794` | `0x13E` | 1B | `EEP[13E]` | I/F 속도 명령 시작값 |
| `IF_B_WeRef_end` | `0x0793` | `0x13F` | 1B | `EEP[13F]` | I/F 속도 명령 종료값 |
| `IF_Fan_locking_limit` | MAP 일부에서 미확인 | `0x140` | 1B | `EEP[140] * 2` | Fan locking 판정 제한 |
| `IF_LeadAngle_limit` | MAP 일부에서 미확인 | `0x141` | 1B | `EEP[141] * 1024 / 180` | degree 입력을 내부 angle count로 변환 |
| `IF_Trans_pass` | `0x0795` | `0x142` | 1B | `EEP[142]` | 전환 PASS 기준/카운트 |
| `IF_Trans_fail` | `0x0796` | `0x143` | 1B | `EEP[143]` | 전환 FAIL 기준/카운트 |
| `IF_Trans_Delay_Count` | `0x0798` | `0x144` | 1B | `EEP[144]` | 전환 후 지연 카운트 |
| `IF_Trans_Delay_Is` | `0x0799` | `0x145` | 1B | `EEP[145]` | 전환 후 전류감소 기준 |
| `IF_Trans_Delay_Shift` | `0x079A` | `0x146` | 1B | `EEP[146]` | 전환 후 감소 Shift |
| `OP_Reverse_Wind_Start_Current` | MAP 일부에서 미확인 | `0x152~0x153` | 2B | `(EEP[152] << 8) + EEP[153]` | 역풍 기동 전류 내부값 |
| `OP_Reverse_Wind_Restart_Current` | MAP 일부에서 미확인 | `0x154~0x155` | 2B | `(EEP[154] << 8) + EEP[155]` | 역풍 재기동 전류 내부값 |

> 기존 프로젝트 기준에서 `Input_Target`는 `1 count = 10 rpm`이지만, 위 I/F EEPROM 항목에도 동일 스케일이 적용되는지는 각 변수의 사용처 확인 후 확정해야 한다.

---

## 14. 1-Shunt 파라미터

`OP_1SHUNT_ITGAP`, `OP_1SHUNT_B_TMIN`은 매크로로 정의된 EEPROM 인덱스다. 현재 제공된 코드에는 해당 매크로의 숫자 정의가 없다.

| 변수 | EEPROM Offset | 계산식 | 의미 |
|---|---:|---|---|
| `iTGap` | `OP_1SHUNT_ITGAP` | `-1 * EEP[index]` | 음의 TGap 값 |
| `B_Tmin` | `OP_1SHUNT_B_TMIN` | `EEP[index] * 10` | 최소 시간/카운트 스케일 |

---

## 15. Auto Tuning 파라미터

| 변수 | RAM 주소 | EEPROM Offset | 크기 | 계산식 | 의미 |
|---|---:|---:|---:|---|---|
| `gucOpCurrentZeta` | `0x0713` | `0x15B` | 1B | `EEP[15B]` | 전류제어 감쇠비 관련 설정 |
| `gushOpCurrentNaturalFreq` | `0x0A40` | `0x15C~0x15D` | 2B | `(EEP[15C] << 8) + EEP[15D]` | 전류제어 자연주파수 |
| `gucOpSpeedZeta` | `0x0715` | `0x15E` | 1B | `EEP[15E]` | 속도제어 감쇠비 관련 설정 |
| `gushOpSpeedNaturalFreq` | `0x0A42` | `0x15F~0x160` | 2B | `(EEP[15F] << 8) + EEP[160]` | 속도제어 자연주파수 |
| `gushOpJ_DIVIDE_KT` | `0x0A44` | `0x161~0x162` | 2B | `(EEP[161] << 8) + EEP[162]` | `J / Kt` 관련 설정값 |
| `gucOpMotorPolePair` | `0x0714` | `0x163` | 1B | `EEP[163]` | 모터 극쌍 수 |

---

## 16. EEPROM SW Part Number

| 변수 | RAM 주소 | EEPROM Offset | 크기 | 계산식 |
|---|---:|---:|---:|---|
| `OP_EEPROM_SW_P_NO_FIRST` | `0x070F` | `0x1F8` | 1B | `EEP[1F8]` |
| `OP_EEPROM_SW_P_NO_SECOND` | `0x0710` | `0x1F9` | 1B | `EEP[1F9]` |
| `OP_EEPROM_SW_P_NO_THIRD` | `0x0711` | `0x1FA` | 1B | `EEP[1FA]` |
| `OP_EEPROM_SW_P_NO_REVISION` | `0x0712` | `0x1FB` | 1B | `EEP[1FB]` |

---

## 17. EEPROM Checksum 구조

EEPROM 마지막 두 바이트는 내부 체크섬으로 사용된다.

```c
for (i = 0; i < EEPROM_SIZE - 2; i++)
{
    eeprom_check_sum += roption[i];
}
```

비교 규칙:

```text
roption[EEPROM_SIZE - 2] = Sum High Byte XOR 0x55
roption[EEPROM_SIZE - 1] = Sum Low Byte  XOR 0x55
```

주의사항:

- EEPROM 데이터 바이트를 변경하면 내부 체크섬 2바이트를 다시 계산해야 한다.
- Intel HEX 각 레코드 끝의 체크섬과 EEPROM 내부 체크섬은 서로 다른 값이다.
- HEX 텍스트를 직접 수정한다면 두 체크섬 체계를 모두 올바르게 갱신해야 한다.

---

## 18. HEX 파일과 MAP 파일의 관계

### MAP 파일

MAP 파일은 변수와 함수가 실행 시 어느 메모리 주소에 배치되는지 보여준다.

```text
OP_Rs → RAM 0x000009B6
OP_Ld → RAM 0x000009B8
OP_Iscale → RAM 0x000009D4
```

### HEX 파일

HEX 파일은 MCU Flash 또는 지정된 메모리 영역에 기록되는 실제 바이트 데이터다.

### DD_EEPROM.c

`DD_EEPROM.c`는 EEPROM의 어느 오프셋을 읽어 어떤 RAM 변수에 넣고, 어떤 스케일 계산을 수행하는지 정의한다.

```text
MAP          : RAM에서 어디에 있는가
HEX          : 저장 메모리에 어떤 바이트가 있는가
DD_EEPROM.c  : 저장 바이트가 변수로 어떻게 변환되는가
```

---

## 19. 핵심 파라미터 요약

| 변수 | EEPROM Offset | RAM 주소 | 최종 계산/해석 |
|---|---:|---:|---|
| `PWM_KHZ_Main` | `0x0E` | `0x0A3A` | `EEP[0E]`, 코드 명칭상 kHz |
| `OP_Iscale` | `0x10~0x11` | `0x09D4` | `(H << 8) + L`, 프로젝트 기준 2048 |
| `OP_Rs` | `0x16~0x17` | `0x09B6` | `(H << 8) + L`, Q11 |
| `OP_Ld` | `0x18~0x19` | `0x09B8` | `(H << 8) + L`, Q13 |
| `OP_Lq` | `0x1A~0x1B` | `0x09BA` | `(H << 8) + L`, Q13 |
| `OP_KE` | `0x1C~0x1D` | `0x09BC` | `(H << 8) + L`, Q13 |
| `OP_Start_Fail_Imag_Curr` | `0x39` | `0x0A24` | `EEP[39] * Iscale / 10`; 기준상 `EEP[39]/10 A` |
| `OP_Start_Fail_Freq` | `0x3A` | `0x0A26` | `EEP[3A]`; rpm 환산은 사용처 확인 |
| `OP_Align_Current` | `0x3D` | `0x0A16` | `EEP[3D]`; 기준상 `EEP[3D]/10 A` |
| `OP_Slope_Start` | `0x65~0x66` | `0x0A20` | `(H << 8) + L` |
| `OP_Kp_Align` | `0x9C~0x9D` | `0x0A14` | `(H << 8) + L` |
| `OP_Ki_Align` | `0x9E` | `0x0709` | `EEP[9E]` |
| `OP_Kp_mori` | `0x123~0x124` | MAP 일부에서 미확인 | `(H << 8) + L` |
| `IF_transition_Rpm_Start` | `0x137` | MAP 일부에서 미확인 | `EEP[137]`; rpm 스케일 확인 필요 |
| `IF_transition_Rpm_End` | `0x138` | MAP 일부에서 미확인 | `EEP[138]`; rpm 스케일 확인 필요 |
| `gushOpCurrentNaturalFreq` | `0x15C~0x15D` | `0x0A40` | `(H << 8) + L` |
| `gushOpSpeedNaturalFreq` | `0x15F~0x160` | `0x0A42` | `(H << 8) + L` |
| `gucOpMotorPolePair` | `0x163` | `0x0714` | `EEP[163]`, 극쌍 수 |

---

## 20. 아직 확정이 필요한 항목

다음 항목은 추가 정의 또는 실제 사용 코드를 확인해야 물리 단위를 확정할 수 있다.

1. `EEPROM_START_ADDRESS`
2. `EEPROM_SIZE`
3. `OP_1SHUNT_ITGAP`
4. `OP_1SHUNT_B_TMIN`
5. `OP_Start_Fail_Freq`의 정확한 rpm/count
6. `IF_transition_Rpm_Start`, `IF_transition_Rpm_End`의 정확한 rpm/count
7. `OP_Aopt[]`의 실제 각도 단위
8. `OP_Rs`, `OP_Ld`, `OP_Lq`, `OP_KE`의 최종 물리 단위 환산식
9. 별도 EEPROM HEX가 존재하는지 여부

이 값들이 확보되면 다음 형태의 최종 실측 테이블을 작성할 수 있다.

| 변수 | EEPROM 절대주소 | 원시 HEX | Raw 10진수 | RAM 계산값 | 물리값 | 단위 |
|---|---:|---|---:|---:|---:|---|
| `OP_Rs` | `BASE+0x16` | 미확정 | 미확정 | 미확정 | 미확정 | Ω 또는 내부 Q값 |
| `OP_Ld` | `BASE+0x18` | 미확정 | 미확정 | 미확정 | 미확정 | H 또는 내부 Q값 |
| `OP_Lq` | `BASE+0x1A` | 미확정 | 미확정 | 미확정 | 미확정 | H 또는 내부 Q값 |
| `OP_KE` | `BASE+0x1C` | 미확정 | 미확정 | 미확정 | 미확정 | 역기전력 상수 |
| `OP_Align_Current` | `BASE+0x3D` | 미확정 | 미확정 | 미확정 | Raw/10 | A |

---

## 21. 최종 정리

- EEPROM 데이터는 `roption[]` 포인터를 통해 접근한다.
- 변수별 실제 EEPROM 저장 위치는 `EEPROM_START_ADDRESS + Offset`이다.
- 16비트 파라미터는 대부분 EEPROM에서 상위 바이트, 하위 바이트 순으로 저장된다.
- 전류 제한 항목은 `EEPROM Raw * OP_Iscale / 10`으로 내부 count가 생성된다.
- `OP_Iscale = 2048`이고 `I[A] = count/2048` 기준이면 EEPROM 1 count는 0.1 A에 해당한다.
- Align 전류와 I/F 전류 스텝은 기준값과 증분값으로 3단계를 생성한다.
- MAP 파일의 주소는 실행 중 RAM 주소이며 EEPROM 저장주소가 아니다.
- 실제 EEPROM HEX 값과 연결하려면 `EEPROM_START_ADDRESS`와 EEPROM 데이터가 포함된 HEX가 필요하다.
- EEPROM 값을 수정할 때에는 EEPROM 내부 체크섬과 Intel HEX 레코드 체크섬을 각각 다시 계산해야 한다.
