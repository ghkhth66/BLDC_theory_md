# Part XIV. RX26T BLDC Sensorless Startup Tracer

## Porting, 이론, 실측 검증 및 결과 판정 통합 교재

- **대상 MCU:** Renesas RX26T
- **기준 시스템:** Panasonic Micom Tracer Framework
- **문서 목적:** Tracer 포팅 변경 내역, 센서리스 제어 이론, 단계별 검증 절차 및 결과 판정 기준 정리
- **작성 기준일:** 2026-10-08
- **현재 상태:** CC-RX 컴파일 및 링크 성공, `Error: 0`


## 교재 적용 범위와 학습 목표

- **적용 범위:** Stage 3A ~ Stage 6B
- **기준 펌웨어:** RX26T Fan Drive
- **기준 Tracer 구조:** 32-bit × 2CH × 4064 samples
- **기준 PWM 인터럽트:** 62.5 us
- **표준 저장 주기:** 1 ms/sample, `tracer_sample_decimation = 16`
- **현재 범위 제외:** Trigger 5 `SPEED_CHANGE`의 실제 Hook 개발 및 검증

이 장을 완료하면 다음 항목을 수행할 수 있어야 한다.

1. Signal, Trigger, Capture Start Mode를 서로 독립적으로 설정한다.
2. Align, I/F Startup, FOC 전환, Sensorless Stable 구간을 구분한다.
3. Observer, PLL, Current 및 Speed 신호를 물리적으로 해석한다.
4. 정상 기동과 탈조 파형을 구분하고 원인 후보를 좁힌다.
5. 동일 조건 반복시험으로 재현성과 KPI 편차를 평가한다.

### 세 가지 핵심 질문

```text
Signal Mode       = 무엇을 볼 것인가?
Trigger Mode      = 언제 기준을 잡을 것인가?
Capture Start Mode = Trigger 뒤 언제 저장할 것인가?
```

---

## Stage 기반 검증 Roadmap

| Stage | 검증 대상 | 대표 설정 | 완료 조건 |
|---|---|---|---|
| 3A | Build 및 RAM 배치 | Error 0, Link 완료 | ABS/HEX 생성, Buffer 충돌 없음 |
| 3B | Framework 수동 기록 | Manual, Mode 0 | Index 4064, Done 1 |
| 4A | EEMF Observer 시작 | Trigger 1, Mode 2/3 | 최초 1회 Trigger, Raw/Filtered 유효 |
| 4B | I/F 성공 판정 | Trigger 2, Mode 4 | 실패 시 미발생, 성공 시 1회 발생 |
| 5A | FOC 전환 시작 | Trigger 3, Mode 4/5/6 | Theta Error 및 Speed Gap 수렴 |
| 5B | FOC 전환 완료 | Trigger 4, Mode 6 | 완료 후 Gap 감소 및 제어 지속 |
| 6A | Align 실제 전류 | Trigger 9, Mode 9 | IdeRef 대비 Ide 추종 확인 |
| 6B | Stable 및 반복성 | Trigger 6, Mode 6 | Hold 만족, 3회 반복 KPI 비교 |

> Stage 통과는 단순히 Trigger가 발생했다는 의미가 아니다. Trigger 시점, 저장 시간축, 데이터 유효성, 물리 단위, 파형 KPI와 반복성까지 모두 확인해야 한다.

---

## 1. 문서 목적과 현재 결론

본 문서는 Panasonic Micom에서 실측 검증한 Tracer Framework를 Renesas RX26T로 이식한 결과를 정리하고, 센서리스 BLDC/PMSM 제어 관점에서 각 Signal Mode와 Trigger Mode의 의미를 설명하며, 이후 실기 검증 절차와 결과 분석 기준을 정의한다.

현재 이식 방향은 다음과 같다.

```text
Panasonic Tracer Framework
        Golden Reference
               ↓
Renesas RX26T Tracer
        Porting Target
```

핵심 결론은 다음과 같다.

1. RX26T Tracer는 Panasonic과 동일하게 **32-bit, 2-channel** 구조를 사용한다.
2. Signal Mode, Trigger Mode, Capture Mode의 번호와 의미를 유지한다.
3. RX26T PWM 인터럽트 주기는 62.5 us이며, Decimation을 조절하여 250 us, 1 ms, 4 ms 등의 저장 주기를 선택한다.
4. 32-bit 2CH 버퍼는 기존 16-bit 4CH 버퍼와 동일한 총 RAM 사용량을 유지한다.
5. 현재 빌드는 성공했으며, 이후 단계는 Watch 설정, Trigger 발생, Buffer 기록, Scale, CSV Export 및 Python Viewer 검증이다.

---

## 2. 변경 전후 구조

### 2.1 기존 임시 Tracer 구조

기존 RX26T 시험 구조는 다음과 같았다.

```c
volatile signed short test1[4064];
volatile signed short test2[4064];
volatile signed short test3[4064];
volatile signed short test4[4064];
```

구조적 특징은 다음과 같다.

```text
16-bit × 4 channels × 4064 samples
= 32,512 bytes
```

이 구조는 동시에 네 신호를 관측하기에는 유리하지만, `theta_err_mori`, `Error_sum_mori`, `wo_hat_mori`, `wr_hat_mori`처럼 32-bit 동적 범위를 사용하는 Observer 및 PLL 내부 신호를 저장할 때 overflow, wrap-around, 부호 반전 또는 정보 손실이 발생할 수 있다.

### 2.2 변경된 RX26T Tracer 구조

변경 후 구조는 다음과 같다.

```c
volatile SLong tracer_ch1[TRACER_LOG_SIZE];
volatile SLong tracer_ch2[TRACER_LOG_SIZE];
```

```text
32-bit × 2 channels × 4064 samples
= 32,512 bytes
```

따라서 총 RAM 사용량은 기존 구조와 동일하고, 각 채널의 데이터 폭만 16-bit에서 32-bit로 확장된다.

### 2.3 RAM 배치

기존 `test1~test4`가 사용하던 메모리 영역을 두 개의 32-bit 채널로 재구성하였다.

```text
tracer_ch1: 0x00003000 ~ 0x00006F7F
tracer_ch2: 0x00006F80 ~ 0x0000AEFF
```

각 채널의 크기는 다음과 같다.

```text
4064 samples × 4 bytes = 16,256 bytes = 0x3F80 bytes
```

절대주소 배치 시 기존 `test1~test4` 정의는 제거되어야 하며, `tracer_ch1`, `tracer_ch2` 정의는 `DD_INV_Tracer.c` 한 곳에만 존재해야 한다.

---

## 3. 파일별 변경 내용

## 3.1 DD_INV_Tracer.h

`DD_INV_Tracer.h`는 Tracer Framework의 공통 사양을 선언한다.

### 3.1.1 Buffer 및 Decimation 정의

```c
#define TRACER_LOG_SIZE      (4064U)
#define TRACER_DECIMATION    (16U)
```

- `TRACER_LOG_SIZE = 4064`: 채널당 저장 샘플 수
- `TRACER_DECIMATION = 16`: 62.5 us PWM 인터럽트 기준 1 ms/sample

### 3.1.2 Signal Mode 정의

```text
0  MANUAL
1  DQ_INPUT
2  EEMF_RAW
3  EEMF_FILTERED
4  THETA_ERROR
5  PLL_OUTPUT
6  SPEED_ESTIMATION
7  ANGLE_ESTIMATION
8  PHASE_CURRENT
9  ALIGN_D_CURRENT
```

Signal Mode 번호는 Panasonic 교재 및 Python Viewer와의 호환 기준이므로 변경하지 않는다.

### 3.1.3 Trigger Mode 정의

```text
0  MANUAL
1  EEMF_FIRST_CALL
2  IF_OK
3  FOC_START
4  FOC_COMPLETE
5  SPEED_CHANGE
6  STABLE
7  ALIGN_START
8  IF_START
9  ALIGN_CURRENT_START
```

현재 RX26T에 우선 연결된 자동 Trigger는 1, 2, 3, 4, 9이다. Trigger 5는 보류 항목이며 Trigger 6은 Stable Service로 처리한다. Trigger 7, 8은 실제 상태 전이 지점 검증 후 Hook 위치를 확정한다.

### 3.1.4 Capture Start Mode 정의

```text
0  IMMEDIATE
1  TIME_DELAY
2  VALUE_STABLE
```

- **IMMEDIATE:** Trigger 수신 즉시 기록 시작
- **TIME_DELAY:** Trigger 후 지정 시간 경과 시 기록 시작
- **VALUE_STABLE:** 선택 신호가 지정 범위에서 지정 시간 유지될 때 기록 시작

### 3.1.5 Value-Stable Condition 정의

다음 신호를 안정 조건으로 선택할 수 있도록 Getter 기반 조건 항목을 선언하였다.

```text
B_WeEst
wo_hat_mori >> 6
wr_hat_mori >> 6
Speed Gap
Theta Error
B_Ide
B_IdeRef
Maximum Phase Current
Input Target
```

### 3.1.6 Align Current Trigger Threshold

```c
#define TRACER_ALIGN_CURRENT_THRESHOLD_COUNT (OP_Iscale / 10)
```

`OP_Iscale = 2048 count/A` 기준으로 약 0.1 A에 해당하는 count를 Trigger 9의 최소 전류 조건으로 사용한다. 이 값은 실제 current offset, noise floor 및 Align 전류 프로파일을 확인한 뒤 최종 확정한다.

---

## 3.2 DD_INV_Tracer.c

`DD_INV_Tracer.c`는 Buffer, Trigger, Capture State, Delay, Value Stable 및 Stable Trigger 기능을 구현한다.

### 3.2.1 32-bit 2CH Buffer

```c
#pragma address tracer_ch1=0x00003000
volatile SLong tracer_ch1[TRACER_LOG_SIZE];

#pragma address tracer_ch2=0x00006F80
volatile SLong tracer_ch2[TRACER_LOG_SIZE];
```

고정주소는 기존에 확보한 Tracer 전용 RAM 영역을 재사용하기 위한 것이다.

### 3.2.2 주요 제어 변수

```text
tracer_signal_mode
tracer_trigger_mode
tracer_captured_trigger
tracer_request
tracer_sequence
tracer_start_speed
tracer_start_target
tracer_index
tracer_divider
tracer_sample_decimation
tracer_enable
tracer_done
```

이 변수들은 Watch에서 Tracer 설정과 상태를 확인하기 위한 핵심 인터페이스다.

### 3.2.3 Tracer_Start

수동 Capture를 시작한다.

```text
현재 Capture 상태 초기화
→ captured trigger를 MANUAL로 설정
→ sequence 증가
→ tracer_enable 활성화
```

### 3.2.4 Tracer_Stop

현재 Capture 또는 Waiting 상태를 정지한다.

```text
tracer_enable = DISABLE
tracer_request = NONE
tracer_waiting = 0
```

### 3.2.5 Tracer_Trigger

자동 Trigger 이벤트를 처리한다.

주요 보호 조건은 다음과 같다.

1. 이미 Delay 또는 Value-Stable 대기 중이면 최초 이벤트 메타데이터를 보호한다.
2. `tracer_monitor_enable = 0`이면 운전 코드에 영향을 주지 않고 즉시 반환한다.
3. Watch에서 선택한 Trigger ID와 실제 발생 ID가 다르면 반환한다.
4. Capture 진행 중이면 새 Trigger를 무시한다.
5. 처리되지 않은 Start Request가 있으면 중복 Request를 무시한다.

### 3.2.6 Tracer_Record

PWM 인터럽트에서 CH1, CH2를 기록한다.

```text
Trigger Request 처리
→ Capture 상태 초기화
→ Decimation Counter 증가
→ 지정 주기마다 32-bit CH1/CH2 저장
→ 4064 samples 완료 시 tracer_done 설정
```

### 3.2.7 Tracer_MonitorService_1ms

기존 1 ms Service에서 호출되어 다음 기능을 처리한다.

```text
TIME_DELAY Capture
VALUE_STABLE Capture
STABLE Trigger
Service Alive 확인
```

이 함수가 실제 1 ms 주기로 호출되지 않으면 Delay, Hold Time 및 Stable Trigger 시간이 실제 시간과 일치하지 않는다.

### 3.2.8 Stable Trigger

Stable Trigger는 속도, 속도 추정 Gap 및 위치 오차가 설정 범위 안에서 지정 시간 유지될 때 Trigger 6을 발생한다.

```text
Speed Low <= B_WeEst <= Speed High
Speed Gap <= Gap Max
abs(theta_err_mori) <= Theta Max
위 조건이 Hold Time 동안 연속 유지
```

Stable Trigger의 판정값은 모두 firmware native count이므로 테스트 전 scale 및 임계값을 확인해야 한다.

---

## 3.3 DD_INV_PWM.c

`DD_INV_PWM.c`에는 실제 센서리스 제어 단계와 Tracer Framework를 연결하는 Hook, Signal 선택 및 Getter 함수가 추가되었다.

### 3.3.1 초기화 Latch Reset

`DD_EEMF_CtrlInit()`과 `DD_IF_CtrlInit()`에서 자동 Trigger one-shot latch를 초기화하였다.

```text
tracer_eemf_first_call_latched
tracer_if_ok_latched
tracer_foc_start_latched
tracer_foc_complete_latched
tracer_align_start_latched
tracer_if_start_latched
tracer_align_current_start_latched
```

모터 정지 후 재기동 시 동일 Trigger가 다시 정상적으로 발생하도록 하기 위한 처리다.

### 3.3.2 Trigger 1: EEMF_FIRST_CALL

Hook 위치:

```text
DD_EEMF_sensorless() 최초 진입
```

의미:

```text
Observer 계산 최초 실행 지점
```

센서리스 제어 관점에서 이 시점은 EEMF Observer와 PLL 내부 상태가 실제 입력 전압 및 전류를 기반으로 업데이트되기 시작하는 지점이다.

### 3.3.3 Trigger 2: IF_OK

Hook 위치:

```text
DD_IF_pwm_int_start_fail()
```

조건:

```text
IF_StartFail_Flag == 0
IF_Fan_locking >= IF_Fan_locking_limit
```

의미:

I/F 강제 기동 구간에서 전압, 전류 및 예상 EEMF 관계가 허용 범위에 들어와 FOC 전환을 준비할 수 있는 상태가 되었음을 의미한다.

### 3.3.4 Trigger 3: FOC_START

Hook 위치:

```text
DD_IF_transition()
```

조건:

```text
I/F 전환 속도 도달
Lead Angle 허용 범위 만족
Transition Delay 완료
f_sensorless = 2 직전
```

의미:

강제각 기반 I/F 제어에서 Observer 추정각 기반 센서리스 FOC로 전환이 시작되는 핵심 시점이다.

### 3.3.5 Trigger 4: FOC_COMPLETE

Hook 위치:

```text
DD_EEMF_Current_Control()
```

조건:

```text
IF_transition: 1 → 0
```

의미:

전환 초기값 및 과도 상태 처리가 종료되고 정상 FOC 전류 제어 루프가 지속되는 시점이다.

### 3.3.6 Trigger 9: ALIGN_CURRENT_START

Hook 위치:

```text
DD_EEMF_pwm_int()
```

조건:

```text
Align/Rs Tune 상태
B_IdeRef가 Threshold 이상
상전류 중 하나 이상이 Threshold 이상
```

의미:

Align 명령만 설정된 시점이 아니라 실제 물리 전류가 흐르기 시작한 시점을 검출한다.

### 3.3.7 Signal Mode 선택

`DD_EEMF_sensorless()` 및 `DD_EEMF_pwm_int()`에서 `tracer_signal_mode`에 따라 CH1, CH2 Source를 선택한다.

```text
Mode 0: theta_err_mori, wr_hat_mori >> 6
Mode 1: IDS_mori, IQS_mori
Mode 2: Error_VDS_sl, Error_VQS_sl
Mode 3: Error_VDS_hat_sl, Error_VQS_hat_sl
Mode 4: theta_err_mori, Error_sum_mori >> 6
Mode 5: theta_err_mori, wo_hat_mori >> 6
Mode 6: wo_hat_mori >> 6, wr_hat_mori >> 6
Mode 7: theta_mori, theta_err_mori
Mode 8: B_Ias, B_Ibs
Mode 9: B_IdeRef, B_Ide
```

### 3.3.8 Getter 함수

1 ms Monitor Service에서 PWM 내부 변수를 안전하게 읽기 위해 Getter를 추가하였다.

```text
Tracer_Get_BWeEst
Tracer_Get_WoHatLogged
Tracer_Get_WrHatLogged
Tracer_Get_SpeedGapLogged
Tracer_Get_ThetaError
Tracer_Get_BIde
Tracer_Get_BIdeRef
Tracer_Get_MaxPhaseCurrent
```

---

## 4. 센서리스 제어 이론적 배경

## 4.1 전체 기동 단계

본 펌웨어의 센서리스 기동은 다음 단계로 해석한다.

```text
Command
→ Align
→ I/F Startup
→ FOC Transition Start
→ FOC Transition Complete
→ Sensorless Stable
```

### Align

정지 상태에서는 역기전력 정보가 부족하므로 회전자 자속 방향을 특정 방향으로 정렬한다. 이때 `B_IdeRef`와 `B_Ide`의 추종 관계, 상전류 상승 및 전류 포화 여부를 확인해야 한다.

### I/F Startup

저속에서는 EEMF 기반 위치 추정의 신뢰도가 낮기 때문에 강제 전기각과 전류 크기로 회전자를 가속한다. I/F 구간에서는 전류가 과도하게 크지 않으면서 실제 회전자가 강제각을 따라가는지를 확인한다.

### FOC Transition

강제각에서 Observer 추정각으로 제어 기준이 전환된다. 이 구간이 Startup 탈조가 가장 빈번하게 발생하는 핵심 관측 구간이다.

### Sensorless FOC

Observer가 전압과 전류로부터 EEMF 또는 위치 오차를 추정하고 PLL이 위치 오차를 줄이도록 속도와 각도를 갱신한다.

---

## 4.2 EEMF Observer 신호의 의미

### Raw EEMF Error

```text
Error_VDS_sl
Error_VQS_sl
```

전압 입력에서 저항 전압강하, 전류 미분항 및 회전 결합항을 보상한 Observer 입력 성분이다.

분석 기준:

- PWM/ADC noise가 포함될 수 있다.
- 두 축 중 한 축만 비정상적으로 치우치면 전류 변환, 전압 추정 또는 파라미터 불일치를 의심한다.
- FOC 전환 직전 신호 크기가 지나치게 작으면 저속 EEMF 관측성이 부족할 수 있다.
- 불연속 jump나 clipping이 있으면 scale, overflow 또는 계산 포화를 점검한다.

### Filtered EEMF Error

```text
Error_VDS_hat_sl
Error_VQS_hat_sl
```

Raw EEMF Error를 Filter한 신호다.

분석 기준:

- Raw 신호 대비 noise가 감소해야 한다.
- Filter가 너무 강하면 위상 지연이 증가해 FOC 전환 시 PLL 응답이 늦어질 수 있다.
- Filter가 너무 약하면 noise가 `theta_err_mori`와 PLL 출력에 직접 전달될 수 있다.
- Raw는 정상이나 Filtered가 느리거나 찌그러지면 `Filter_emf_ob`와 실행주기 의존성을 점검한다.

---

## 4.3 위치 오차 및 PLL

### Theta Error

```text
theta_err_mori
```

Filtered EEMF 벡터로부터 계산한 전기각 위치 오차다.

물리 단위 환산:

```text
electrical degree
= count × 180 / (π × 8192)
```

분석 기준:

- FOC 전환 전후 순간 오차가 발생할 수 있으나 이후 감소해야 한다.
- 오차가 한 방향으로 지속하면 각도 offset, 회전 방향, phase sequence 또는 Observer 부호를 점검한다.
- 오차가 반복 진동하면 PLL Gain, Filter 지연 또는 PWM 주기 변경 영향을 점검한다.
- ±π 부근 wrap이 반복되면 PLL Lock 실패 또는 탈조 가능성이 높다.

### PLL PI Controller

```text
Error_sum_mori
wo_hat_mori
wr_hat_mori
```

구조적으로 다음과 같이 해석한다.

```text
Theta Error
→ P Term + I Term
→ wo_hat_mori
→ Speed Filter
→ wr_hat_mori
→ Estimated Angle Integration
```

`Error_sum_mori`는 누적 적분 상태이므로 32-bit 저장이 반드시 필요하다.

분석 기준:

- `theta_err_mori`가 감소하는 동안 `Error_sum_mori`는 필요한 정상상태 속도 성분으로 수렴해야 한다.
- `Error_sum_mori`가 제한값에 붙으면 PLL 포화 상태를 의심한다.
- `wo_hat_mori`는 빠른 속도 보정 성분을 포함하므로 변동이 크다.
- `wr_hat_mori`는 Filter된 속도 추정값이므로 `wo_hat_mori`보다 매끄럽고 약간 늦을 수 있다.
- Gain이 너무 크면 진동, overshoot, noise amplification이 발생할 수 있다.
- Gain이 너무 작으면 FOC 전환 후 lock 시간이 길어지고 부하 변화 추종이 느려질 수 있다.

---

## 4.4 PWM 주기와 Gain의 관계

RX26T의 PWM 인터럽트 주기는 62.5 us이다. 기존 250 us 기반 제어와 비교하면 동일한 실제 시간 동안 제어 계산 횟수가 네 배가 된다.

특히 적분 Gain은 코드에서 실행주기 또는 PWM 주파수를 포함하므로, PWM 주기 변경 시 기존 Gain을 단순 재사용하면 실제 시간축의 적분 작용이 달라질 수 있다.

검토 대상:

```text
Ki_mori
Filter_emf_ob
Filter_wr
Current PI Ki
Speed PI Ki
Transition Delay Count
Hold Count
```

판정 시 count 값만 비교하지 말고 실제 시간 환산값을 함께 비교해야 한다.

---

## 5. Sampling 및 전체 기록 시간

RX26T PWM 인터럽트 기준:

```text
1 interrupt = 62.5 us
```

| Decimation | Sample Interval | 4064 Samples 전체 시간 | 권장 분석 |
|---:|---:|---:|---|
| 1 | 62.5 us | 0.254 s | PWM 직결 과도현상 |
| 4 | 250 us | 1.016 s | FOC 전환 정밀 분석 |
| 16 | 1 ms | 4.064 s | Panasonic 기준 호환, 표준 기동 분석 |
| 64 | 4 ms | 16.256 s | 기동 후 안정화, 재시동 |
| 256 | 16 ms | 65.024 s | 장시간 운전 및 Fault 추세 |

기본값은 다음과 같다.

```c
tracer_sample_decimation = 16U;
```

이는 Panasonic 교재의 1 ms/sample 시간축과 동일한 분석 기준을 제공한다.

---

## 6. 물리 단위 Scale

## 6.1 확정 Scale

### Target Speed

```text
Input_Target
1 count = 10 rpm
```

### Estimated Speed

```text
B_WeEst rpm = count × 0.15
```

### Observer Speed

```text
(wo_hat_mori >> 6) rpm = logged count × 0.01845703125
(wr_hat_mori >> 6) rpm = logged count × 0.01845703125
```

### Phase 및 dq Current

```text
B_Ias, B_Ibs, B_Ics
B_Ide, B_Iqe, B_IdeRef, B_IqeRef

Current [A] = count / 2048
```

### Align Current Option

```text
OP_Align_Current [A] = value / 10
```

### Theta Error

```text
theta_err_mori [electrical degree]
= count × 180 / (π × 8192)
```

대상 모터는 8극, 극쌍수 4를 기준으로 한다.

## 6.2 Count 유지 신호

다음 내부 Q-format 신호는 물리 단위 환산을 확정하기 전까지 Count로 표시한다.

```text
IDS_mori
IQS_mori
VDS_sl_mori
VQS_sl_mori
Error_VDS_sl
Error_VQS_sl
Error_VDS_hat_sl
Error_VQS_hat_sl
Error_sum_mori
```

Q-format 값만으로 A 또는 V를 단정하지 않고, 입력 Scaling과 Base Value를 추가 확인한 뒤 물리 단위로 승격한다.

---

## 7. 검증 전 공통 준비

### 7.1 Build 상태 확인

```text
Build ended(Error:0)
Renesas Optimizing Linker Completed
HEX 생성 완료
```

Warning은 별도 관리하되 Tracer 검증 시작 조건은 Error 0으로 한다.

### 7.2 초기 Watch 항목

```text
tracer_monitor_enable
tracer_signal_mode
tracer_trigger_mode
tracer_capture_start_mode
tracer_sample_decimation
tracer_enable
tracer_request
tracer_waiting
tracer_done
tracer_index
tracer_sequence
tracer_captured_trigger
tracer_start_speed
tracer_start_target
tracer_ch1
tracer_ch2
```

### 7.3 공통 설정

표준 검증은 다음 설정으로 시작한다.

```text
tracer_monitor_enable = 1
tracer_capture_start_mode = 0
tracer_sample_decimation = 16
```

### 7.4 재시험 전 초기화 확인

모터 정지 또는 `DD_EEMF_CtrlInit()` 수행 후 다음 latch가 0으로 복귀하는지 확인한다.

```text
tracer_eemf_first_call_latched
tracer_if_ok_latched
tracer_foc_start_latched
tracer_foc_complete_latched
tracer_align_current_start_latched
```

---

## 8. 단계별 검증 절차

## 8.1 Phase A: Framework 기본 동작

### 목적

Trigger 이전에 Buffer, Index, Decimation 및 Completion State 자체를 검증한다.

### 설정

```text
tracer_monitor_enable = 1
tracer_trigger_mode = 0
tracer_signal_mode = 0
tracer_sample_decimation = 16
```

### 실행

```text
tracer_enable = 1
```

### 확인 항목

```text
tracer_index가 0에서 증가하는가
tracer_index가 4064에서 정지하는가
tracer_done이 1이 되는가
tracer_enable이 완료 후 0이 되는가
tracer_ch1, tracer_ch2에 32-bit 값이 기록되는가
```

### PASS 기준

```text
Index overflow 없음
두 채널 RAM 손상 없음
4064 sample 완료 후 정상 정지
재시작 시 index 0부터 다시 기록
```

---

## 8.2 Phase B: Sampling Interval 검증

### 목적

Decimation별 실제 Sample Interval을 확인한다.

### 방법

빠르게 변하는 알려진 변수 또는 Toggle 신호를 기록하고 CS+ 시간축 또는 외부 GPIO 기준과 비교한다.

### 시험값

```text
Decimation 4   → 250 us/sample
Decimation 16  → 1 ms/sample
Decimation 64  → 4 ms/sample
```

### PASS 기준

실측 Sample Interval이 설정값과 일치하고, Random PWM 사용 시에도 Tracer 시간축 정의가 의도대로 유지되는지 확인한다.

### 주의

현재 코드는 PWM interrupt 호출 횟수 기반 Decimation이다. Random PWM으로 PWM 주파수가 변하면 sample interval도 함께 변할 수 있으므로, 고정 시간축이 필요하면 향후 timer 기반 sampling을 별도로 검토해야 한다.

---

## 8.3 Phase C: Trigger 검증

### Trigger 1: EEMF_FIRST_CALL

#### 설정

```text
tracer_trigger_mode = 1
tracer_signal_mode = 2 또는 3
```

#### 확인

- `DD_EEMF_sensorless()` 최초 호출에서 한 번만 Trigger되는가
- `tracer_captured_trigger = 1`인가
- 재기동 전 중복 Trigger가 발생하지 않는가
- 초기화 후 다시 Trigger 가능한가

#### 분석 기준

Observer 최초 계산 시 Raw/Filtered EEMF가 0 또는 초기값에서 시작해 실제 운전값으로 변화하는지 확인한다.

---

### Trigger 2: IF_OK

#### 설정

```text
tracer_trigger_mode = 2
tracer_signal_mode = 2, 3 또는 4
```

#### 확인

- `IF_StartFail_Flag = 0` 상태에서만 발생하는가
- `IF_Fan_locking >= IF_Fan_locking_limit` 시점과 Trigger가 일치하는가
- 실패 기동에서는 Trigger되지 않는가

#### 분석 기준

I/F 성공 판정 직전후 EEMF 크기와 Theta Error가 FOC 전환 가능한 수준으로 형성되는지 확인한다.

---

### Trigger 3: FOC_START

#### 설정

```text
tracer_trigger_mode = 3
```

권장 Signal:

```text
Mode 4: Theta Error + PLL Integral
Mode 5: Theta Error + wo_hat
Mode 6: wo_hat + wr_hat
```

#### 확인

- `f_sensorless = 2` 직전에 Trigger되는가
- `IF_transition = 1` 설정 지점과 일치하는가
- Capture가 한 번만 시작되는가

#### 분석 기준

FOC 전환 직후 다음을 확인한다.

```text
Theta Error peak
Theta Error settling
wo_hat overshoot
wr_hat lag
Error_sum saturation
전류 급증 여부
```

FOC 전환 후 Theta Error가 줄어들고 속도 추정이 안정되면 정상이다. Theta Error 발산, PLL 속도 반전, 반복 wrap 또는 전류 급증이 나타나면 탈조 위험으로 판정한다.

---

### Trigger 4: FOC_COMPLETE

#### 설정

```text
tracer_trigger_mode = 4
tracer_signal_mode = 4 또는 6
```

#### 확인

- `IF_transition`이 1에서 0으로 바뀔 때 한 번 발생하는가
- Trigger 3 이후 논리적 순서로 발생하는가
- 완료 이후 정상 전류 PI 제어가 지속되는가

#### 분석 기준

FOC Complete 이후에도 Theta Error 또는 속도 Gap이 계속 증가하면 전환 완료 판정이 너무 빠르거나 Observer Lock이 완료되지 않은 상태일 수 있다.

---

### Trigger 6: STABLE

#### 설정

```text
tracer_trigger_mode = 6
tracer_stable_enable = 1
```

추가 설정:

```text
tracer_stable_speed_low
tracer_stable_speed_high
tracer_stable_speed_gap_max
tracer_stable_theta_abs_max
tracer_stable_hold_ms
```

#### 확인

- 조건 진입 즉시가 아니라 Hold Time 유지 후 Trigger되는가
- 조건이 중간에 깨지면 Hold Counter가 0으로 복귀하는가
- Stable Trigger가 one-shot으로 동작하는가

#### 분석 기준

Stable 판정 후에도 속도 Gap 또는 Theta Error가 크게 변하면 임계값이 느슨하거나 Hold Time이 짧은 것이다.

---

### Trigger 9: ALIGN_CURRENT_START

#### 설정

```text
tracer_trigger_mode = 9
tracer_signal_mode = 9
```

#### 확인

- `B_IdeRef`만 증가하고 실제 전류가 없을 때 Trigger되지 않는가
- 실제 상전류가 Threshold를 넘을 때 Trigger되는가
- Current offset만으로 오동작하지 않는가

#### 분석 기준

```text
CH1 = B_IdeRef
CH2 = B_Ide
```

참조 전류 상승 후 실제 `B_Ide`가 지연을 두고 추종해야 한다. 다음은 비정상 후보이다.

```text
B_IdeRef 상승, B_Ide 무응답
B_Ide 과도 overshoot
지속 진동
Current limit saturation
상전류와 dq 전류 불일치
```

---

## 8.4 Phase D: Signal Mode 검증

각 Mode는 같은 운전 조건에서 최소 2회 반복하여 재현성을 확인한다.

### Mode 0: MANUAL

```text
CH1 = theta_err_mori
CH2 = wr_hat_mori >> 6
```

Observer lock 상태를 간단히 확인하는 기본 조합이다.

### Mode 1: DQ_INPUT

```text
CH1 = IDS_mori
CH2 = IQS_mori
```

Observer 내부 좌표계 전류 입력의 연속성과 포화를 확인한다.

### Mode 2: EEMF_RAW

```text
CH1 = Error_VDS_sl
CH2 = Error_VQS_sl
```

Motor Parameter, 전압 추정, 전류 미분 및 noise 영향을 확인한다.

### Mode 3: EEMF_FILTERED

```text
CH1 = Error_VDS_hat_sl
CH2 = Error_VQS_hat_sl
```

Observer Filter의 noise 저감과 지연을 Mode 2와 비교한다.

### Mode 4: THETA_ERROR

```text
CH1 = theta_err_mori
CH2 = Error_sum_mori >> 6
```

PLL 위치 오차와 Integral 누적 상태를 분석한다.

### Mode 5: PLL_OUTPUT

```text
CH1 = theta_err_mori
CH2 = wo_hat_mori >> 6
```

Theta Error에 대한 PLL의 빠른 속도 응답을 분석한다.

### Mode 6: SPEED_ESTIMATION

```text
CH1 = wo_hat_mori >> 6
CH2 = wr_hat_mori >> 6
```

PLL 출력과 Filtered 추정속도 간의 Gap, overshoot 및 settling을 분석한다.

### Mode 7: ANGLE_ESTIMATION

```text
CH1 = theta_mori
CH2 = theta_err_mori
```

추정각 진행의 연속성을 확인한다. Angle wrap 자체는 정상일 수 있으나 비정상 jump 또는 역방향 진행은 점검 대상이다.

### Mode 8: PHASE_CURRENT

```text
CH1 = B_Ias
CH2 = B_Ibs
```

`B_Ics = -(B_Ias + B_Ibs)` 관계와 current sensing balance를 분석한다.

### Mode 9: ALIGN_D_CURRENT

```text
CH1 = B_IdeRef
CH2 = B_Ide
```

Align 전류 참조 추종, rise time, overshoot 및 정상상태 오차를 분석한다.

---

## 8.5 Phase E: Capture Mode 검증

### Immediate

```text
tracer_capture_start_mode = 0
```

Trigger 직후 `tracer_request`가 처리되고 Capture가 시작되어야 한다.

### Time Delay

```text
tracer_capture_start_mode = 1
tracer_delay_ms = 설정값
```

검증 항목:

```text
Trigger 발생 시 waiting = 1
지정 Delay 후 request = START
실제 대기시간과 tracer_delay_ms 일치
Timeout 동작 확인
```

### Value Stable

```text
tracer_capture_start_mode = 2
```

검증 항목:

```text
Condition Signal 선택
Low <= Value <= High
Hold Time 연속 만족
조건 이탈 시 Hold Counter Reset
Timeout 시 Waiting Cancel
```

---

## 9. 결과 분석 KPI

## 9.1 공통 KPI

- Trigger 발생 여부
- Trigger 중복 발생 여부
- Trigger 순서
- Trigger부터 Capture 시작까지 지연
- 유효 Sample 수
- Sample Interval
- Buffer 완료 여부
- 재기동 반복성

## 9.2 PLL 및 FOC 전환 KPI

- FOC Start 시점 Theta Error
- Maximum absolute Theta Error
- Theta Error settling time
- `wo_hat` peak
- `wr_hat` peak
- `wo_hat - wr_hat` maximum gap
- Speed Gap settling time
- Error Integral saturation 여부
- Angle wrap 또는 역방향 진행 횟수

## 9.3 Current KPI

- `B_IdeRef` rise time
- `B_Ide` rise time
- Current overshoot
- 정상상태 current error
- Phase current peak
- Phase current balance
- current clipping 또는 saturation 여부

## 9.4 반복성 KPI

동일 조건에서 최소 3회 시험하여 다음을 비교한다.

```text
Trigger 시점
FOC Start speed
FOC Complete speed
Maximum Theta Error
Speed Gap peak
Current peak
Settling time
```

반복 시험 간 편차가 큰 경우 초기 회전자 위치, 부하, DC Link, current offset, Random PWM 및 상태 초기화를 확인한다.

---

## 10. 정상 및 이상 패턴 판정 기준

## 10.1 정상 패턴

```text
Align current가 참조값을 안정적으로 추종
I/F 구간에서 속도가 단조 증가
FOC Start 후 Theta Error가 감소
wo_hat 과도 후 wr_hat가 추종
FOC Complete 후 Speed Gap 감소
Current가 한계 내에서 안정
Stable 조건 유지 후 Trigger 6 발생
```

## 10.2 Motor Parameter 불일치 의심

```text
Raw EEMF의 d/q 축 비대칭
FOC 전환 직후 Theta Error bias
부하 증가 시 Speed Gap 급증
Current는 증가하지만 추정속도 불안정
```

점검 대상:

```text
Rs
Ld
Lq
KE
Pole Pair
Scale
```

## 10.3 PLL Gain 부적합 의심

### Gain 과대

```text
Theta Error 고주파 진동
wo_hat overshoot
wr_hat 반복 진동
FOC 전환 직후 전류 급증
```

### Gain 과소

```text
Theta Error 감소가 느림
FOC 전환 후 Lock 지연
부하 변화 속도 추종 지연
```

## 10.4 Filter 부적합 의심

### Filter 과대

```text
Filtered EEMF 지연
FOC 전환 시 PLL 반응 지연
wo_hat와 wr_hat 차이 장기 지속
```

### Filter 과소

```text
Filtered EEMF noise 증가
Theta Error noise 증가
PLL 출력 변동 증가
```

## 10.5 Current Sensing 이상 의심

```text
Phase current offset
Ia + Ib + Ic 관계 불일치
특정 상만 clipping
dq current와 phase current 결과 불일치
Align 참조는 있으나 실제 d축 전류 무응답
```

## 10.6 탈조 의심

```text
Theta Error 발산
±π 부근 반복 wrap
추정속도 급락 또는 부호 반전
전류 급증
FOC Complete 이후에도 Speed Gap 증가
Fault 또는 Restart 연계 발생
```

---

## 11. 권장 시험 Matrix

| Test ID | Trigger | Signal | Decimation | 목적 |
|---|---:|---:|---:|---|
| T00 | Manual | 0 | 16 | Framework 및 Buffer |
| T01 | 9 | 9 | 4 | Align Current Start |
| T02 | 1 | 2 | 4 | Observer 최초 Raw EEMF |
| T03 | 1 | 3 | 4 | Observer 최초 Filtered EEMF |
| T04 | 2 | 4 | 16 | I/F 성공 시 PLL 상태 |
| T05 | 3 | 4 | 4 | FOC Start Theta/Integral |
| T06 | 3 | 5 | 4 | FOC Start PLL Output |
| T07 | 3 | 6 | 4 | FOC Start Speed Gap |
| T08 | 4 | 6 | 16 | FOC Complete 안정화 |
| T09 | 6 | 6 | 16 | Stable 판정 |
| T10 | 3 | 8 | 4 | FOC 전환 상전류 |
| T11 | 3 | 7 | 4 | Angle 연속성 |

시험 결과는 각 Test ID별로 다음 내용을 기록한다.

```text
Firmware Version
PWM Frequency
PWM Interrupt Period
Decimation
Sample Interval
전체 기록시간
Trigger Mode
Signal Mode
운전 Target
부하 조건
전류 제한
Gain 값
Capture 결과
KPI
PASS/FAIL
비고
```

---

## 12. CSV 및 Python Viewer 검증

각 MCU 업체의 Emulator와 Watch 데이터 취득 방법은 다를 수 있다. 그러나 최종 분석 인터페이스는 다음 항목으로 통일한다.

```text
Signal Mode
Trigger Mode
CH1
CH2
Sample Interval
Time Axis
Scale Metadata
```

CS+에서는 Watch에서 `tracer_ch1`, `tracer_ch2`를 확인하고 제공 가능한 Export 기능으로 데이터를 저장한다. CSV 변환 또는 Python 전처리 시 다음 메타데이터를 함께 저장하는 것을 권장한다.

```text
MCU = RX26T
Signal Mode
Trigger Mode
Decimation
PWM INT Period
Sample Interval
Capture Sequence
Start Speed
Start Target
```

Python Viewer 검증 기준:

1. 4064 sample을 누락 없이 읽는가
2. CH1/CH2 부호와 32-bit 값이 유지되는가
3. 시간축이 `sample index × sample interval`로 생성되는가
4. Signal Mode별 Scale이 올바르게 적용되는가
5. Panasonic 데이터와 동일한 Mode 의미로 비교 가능한가

---

## 13. PASS/FAIL 판정 Template

```text
Test ID:
Firmware Version:
Trigger Mode:
Signal Mode:
Capture Mode:
Decimation:
Sample Interval:
Total Capture Time:

Trigger Result:
- Trigger occurred:
- Captured trigger ID:
- Duplicate trigger:
- Sequence:

Buffer Result:
- Final index:
- Done flag:
- CH1 valid:
- CH2 valid:

Sensorless KPI:
- Maximum abs theta error:
- Theta settling time:
- wo_hat peak:
- wr_hat peak:
- Maximum speed gap:
- Speed gap settling time:
- Current peak:
- Integral saturation:

Judgment:
- PASS / FAIL / CONDITIONAL PASS

Root Cause Candidate:
- Motor parameter
- PLL gain
- Filter
- Current sensing
- Voltage estimation
- Scale
- Trigger timing
- State initialization

Follow-up:
```

---

## 14. 최종 완료 기준

RX26T Tracer Porting은 다음 조건을 모두 만족할 때 완료로 판정한다.

```text
Build Error = 0
Link Error = 0
32-bit 2CH Buffer 정상
4064 samples 정상
Decimation별 시간축 검증
Signal Mode 0~9 검증
Trigger 1, 2, 3, 4, 6, 9 검증
Immediate, Time Delay, Value Stable 검증
Scale 검증
CSV Export 검증
Python Viewer 호환 검증
동일 조건 3회 이상 반복성 확인
```

---

## 15. 최종 결론

이번 RX26T 변경은 단순한 데이터 배열 확장이 아니라 Panasonic에서 검증된 Tracer Framework를 RX26T 센서리스 제어 흐름에 맞춰 이식한 것이다.

핵심 성과는 다음과 같다.

```text
기존 16-bit 4CH
→ 32-bit 2CH

RAM 사용량 유지
→ Observer 및 PLL 32-bit 신호 손실 방지

Panasonic Signal/Trigger 정의 유지
→ 교재와 Python Viewer 재사용 가능

Trigger Hook 추가
→ Align, I/F, FOC 전환 및 안정 상태를 동일 기준으로 분석 가능
```

현재 빌드와 링크는 성공하였다. 다음 단계에서는 코드 변경을 확대하기보다, 본 문서의 시험 Matrix에 따라 Trigger, Signal, 시간축, Scale 및 센서리스 제어 KPI를 순차 검증해야 한다.


---

## 16. 현장 시험 최종 체크리스트

### A. 시험 전

- [ ] 최신 Build가 `Error: 0`인지 확인했다.
- [ ] `Renesas Optimizing Linker Completed`를 확인했다.
- [ ] `tracer_ch1`, `tracer_ch2` 주소가 의도한 RAM 영역에 배치되었다.
- [ ] 기존 `test1~test4` 절대주소 정의가 제거되었다.
- [ ] 모터, 부하, DC Link, PWM 주파수 및 Gain 조건을 기록했다.
- [ ] 모터 정지 상태에서 `tracer_monitor_enable = 0`으로 설정값을 입력했다.
- [ ] Signal, Trigger, Capture, Decimation 입력 후 마지막에 Monitor를 활성화했다.

### B. Capture 중

- [ ] `tracer_captured_trigger`가 선택 Trigger와 일치한다.
- [ ] `tracer_sequence`가 Capture마다 한 번 증가한다.
- [ ] `tracer_index`가 예상 속도로 증가한다.
- [ ] Capture 중 중복 Trigger로 Index가 초기화되지 않는다.
- [ ] 운전 제어 주기와 Fault 동작에 이상이 없다.

### C. Capture 완료 후

- [ ] `tracer_done = 1`이다.
- [ ] `tracer_index = 4064`이다.
- [ ] CH1, CH2가 모두 유효한 32-bit 데이터다.
- [ ] Sample Interval과 전체 Capture Window가 계산과 일치한다.
- [ ] Scale 변환 전 원본 Count를 별도 보존했다.
- [ ] CSV에 Mode, Trigger, Decimation, Target, Start Speed를 기록했다.

### D. 센서리스 판정

- [ ] Align에서 `B_Ide`가 `B_IdeRef`를 추종한다.
- [ ] I/F 구간에서 속도와 전류가 불연속 없이 증가한다.
- [ ] FOC Start 후 Theta Error가 감소한다.
- [ ] `wo_hat` 과도응답 후 `wr_hat`가 추종한다.
- [ ] FOC Complete 후 Speed Gap이 감소한다.
- [ ] Integral이 Limit에 지속 고정되지 않는다.
- [ ] 각도 역진행, 반복 Wrap, 속도 부호 반전이 없다.
- [ ] 부하 증가 시 Current, Theta Error, Speed Gap이 허용 범위에서 회복한다.

### E. 반복성

- [ ] 동일 조건 3회 이상 시험했다.
- [ ] Trigger 순서가 동일하다.
- [ ] FOC Start와 Complete 시점 편차를 기록했다.
- [ ] Theta Error Peak, Speed Gap Peak, Current Peak 편차를 기록했다.
- [ ] 편차 발생 시 초기 회전자 위치, 부하, DC Link, Offset, Random PWM을 비교했다.

---

## 17. 시험 결과 요약표

| 항목 | 결과 | 판정 | 비고 |
|---|---|---|---|
| Build 및 Link |  | PASS / FAIL |  |
| 32-bit 2CH Buffer |  | PASS / FAIL |  |
| Sampling Interval |  | PASS / FAIL |  |
| Trigger 1 EEMF_FIRST_CALL |  | PASS / FAIL |  |
| Trigger 2 IF_OK |  | PASS / FAIL |  |
| Trigger 3 FOC_START |  | PASS / FAIL |  |
| Trigger 4 FOC_COMPLETE |  | PASS / FAIL |  |
| Trigger 6 STABLE |  | PASS / FAIL |  |
| Trigger 9 ALIGN_CURRENT_START |  | PASS / FAIL |  |
| Signal Mode 0~9 |  | PASS / FAIL |  |
| CSV Export |  | PASS / FAIL |  |
| Python Viewer |  | PASS / FAIL |  |
| 3회 반복성 |  | PASS / FAIL |  |

---

## 18. 문서 변경 이력

| Version | Date | 변경 내용 |
|---|---|---|
| 1.0 | 2026-10-08 | RX26T Tracer 포팅, 센서리스 이론, Stage 3A~6B 검증 및 판정 기준 통합 |
