# RX26T Tracer Porting 변경 내역 및 검증 절차

작성일: 2026-10-08

---

# 1. 목적

본 문서는 Panasonic Micom 기반 Tracer Framework를 Renesas RX26T 프로젝트에 이식하면서 변경한 내용을 정리하고 향후 검증 절차를 정의하기 위한 문서이다.

현재 상태:

```text
컴파일 성공
링크 성공
HEX 생성 성공

Build Error = 0
```

따라서 현재 단계는

```text
구현 단계
→ 완료

검증 단계
→ 진행
```

이다.

---

# 2. Porting 기본 원칙

## 기준

```text
Panasonic Micom
=
Golden Reference
```

## 대상

```text
Renesas RX26T
```

---

# 유지한 항목

```text
Signal Mode

Trigger Mode

Capture Start Mode

Trace State Machine

Stable Trigger

Value Stable

Time Delay

Python Viewer
```

---

# 변경한 항목

```text
Signal Source

Trigger Hook 위치

Buffer 구현 방식

RAM Allocation
```

---

# 3. DD_INV_Tracer.h 변경 내용

## 추가

### Signal Mode 정의

```c
TRACER_SIGNAL_MANUAL

TRACER_SIGNAL_DQ_INPUT

TRACER_SIGNAL_EEMF_RAW

TRACER_SIGNAL_EEMF_FILTERED

TRACER_SIGNAL_THETA_ERROR

TRACER_SIGNAL_PLL_OUTPUT

TRACER_SIGNAL_SPEED_ESTIMATION

TRACER_SIGNAL_ANGLE_ESTIMATION

TRACER_SIGNAL_PHASE_CURRENT

TRACER_SIGNAL_ALIGN_D_CURRENT
```

---

### Trigger Mode 정의

```c
TRACER_TRIGGER_MANUAL

TRACER_TRIGGER_EEMF_FIRST_CALL

TRACER_TRIGGER_IF_OK

TRACER_TRIGGER_FOC_START

TRACER_TRIGGER_FOC_COMPLETE

TRACER_TRIGGER_SPEED_CHANGE

TRACER_TRIGGER_STABLE

TRACER_TRIGGER_ALIGN_START

TRACER_TRIGGER_IF_START

TRACER_TRIGGER_ALIGN_CURRENT_START
```

---

### Capture Start Mode 정의

```c
TRACER_CAPTURE_START_IMMEDIATE

TRACER_CAPTURE_START_TIME_DELAY

TRACER_CAPTURE_START_VALUE_STABLE
```

---

### Tracer RAM 선언

기존

```c
test1
test2
test3
test4
```

사용 중지

신규

```c
extern volatile SLong tracer_ch1[];

extern volatile SLong tracer_ch2[];
```

---

### Condition Signal 추가

```c
TRACER_CONDITION_B_WE_EST

TRACER_CONDITION_WO_HAT_LOGGED

TRACER_CONDITION_WR_HAT_LOGGED

TRACER_CONDITION_SPEED_GAP_LOGGED

TRACER_CONDITION_THETA_ERROR

TRACER_CONDITION_B_IDE

TRACER_CONDITION_B_IDE_REF

TRACER_CONDITION_MAX_PHASE_CURRENT

TRACER_CONDITION_INPUT_TARGET
```

---

# 4. DD_INV_Tracer.c 변경 내용

## 변경 전

```text
test1
test2
test3
test4
```

16bit 기반 임시 구조

---

## 변경 후

```c
volatile SLong tracer_ch1[TRACER_LOG_SIZE];

volatile SLong tracer_ch2[TRACER_LOG_SIZE];
```

---

## 버퍼 구조

```text
32 bit

2 CH
```

---

## 기록 함수

```c
Tracer_Record()
```

유지

---

## Trigger 함수

```c
Tracer_Trigger()
```

유지

---

## 제어 함수

```c
Tracer_Start()

Tracer_Stop()
```

유지

---

## Service 함수

```c
Tracer_MonitorService_1ms()
```

유지

---

## Stable Trigger

```c
Tracer_StableService_1ms()
```

이식

---

## Value Stable

```c
Tracer_ReadConditionValue()
```

이식

---

## RAM 사용량

```text
4064 samples

×

4 bytes

×

2 channels

=

32512 bytes
```

---

# 5. DD_INV_PWM.c 변경 내용

## Trigger 1

### EEMF_FIRST_CALL

위치

```c
DD_EEMF_sensorless()
```

---

추가

```c
Tracer_Trigger(
    TRACER_TRIGGER_EEMF_FIRST_CALL,
    (Word)B_WeEst,
    (Word)Input_Target
);
```

---

# Trigger 2

## IF_OK

위치

```c
DD_IF_pwm_int_start_fail()
```

---

조건

```c
IF_Fan_locking >= IF_Fan_locking_limit
```

---

추가

```c
Tracer_Trigger(
    TRACER_TRIGGER_IF_OK,
    (Word)B_WeEst,
    (Word)rpm_tgt
);
```

---

# Trigger 3

## FOC_START

위치

```c
DD_IF_transition()
```

---

조건

```c
f_sensorless = 2

IF_transition = 1
```

직전

---

추가

```c
Tracer_Trigger(
    TRACER_TRIGGER_FOC_START,
    (Word)B_WeEst,
    (Word)rpm_tgt
);
```

---

# Trigger 4

## FOC_COMPLETE

위치

```c
DD_EEMF_Current_Control()
```

---

조건

```c
IF_transition

1

↓

0
```

---

추가

```c
Tracer_Trigger(
    TRACER_TRIGGER_FOC_COMPLETE,
    (Word)B_WeEst,
    (Word)rpm_tgt
);
```

---

# Trigger 9

## ALIGN_CURRENT_START

위치

```c
DD_EEMF_pwm_int()
```

---

조건

```c
B_IdeRef

+

실제 상전류 존재
```

---

추가

```c
Tracer_Trigger(
    TRACER_TRIGGER_ALIGN_CURRENT_START,
    (Word)B_WeEst,
    (Word)rpm_tgt
);
```

---

# Signal Mode 이식

## Mode 0

```c
theta_err_mori

wr_hat_mori >> 6
```

---

## Mode 1

```c
IDS_mori

IQS_mori
```

---

## Mode 2

```c
Error_VDS_sl

Error_VQS_sl
```

---

## Mode 3

```c
Error_VDS_hat_sl

Error_VQS_hat_sl
```

---

## Mode 4

```c
theta_err_mori

Error_sum_mori >> 6
```

---

## Mode 5

```c
theta_err_mori

wo_hat_mori >> 6
```

---

## Mode 6

```c
wo_hat_mori >> 6

wr_hat_mori >> 6
```

---

## Mode 7

```c
theta_mori

theta_err_mori
```

---

## Mode 8

```c
B_Ias

B_Ibs
```

---

## Mode 9

```c
B_IdeRef

B_Ide
```

---

# 6. Scale 정리

## 확인 완료

### Speed

```c
B_WeEst
```

```text
rpm = count × 0.15
```

---

```c
wo_hat_mori >> 6
```

```text
rpm = count × 0.01845703125
```

---

```c
wr_hat_mori >> 6
```

```text
rpm = count × 0.01845703125
```

---

### Current

```c
B_Ias

B_Ibs

B_Ics
```

```text
A = count / 2048
```

---

```c
B_Ide

B_Iqe
```

```text
A = count / 2048
```

---

### Angle

```c
theta_err_mori
```

```text
deg_e

=

count × 180

/

(PI × 8192)
```

---

## 보류

```c
IDS_mori

IQS_mori

Error_VDS_*

Error_VQS_*

Error_sum_mori
```

현재 Count 유지

---

# 7. 샘플링 정책

## Panasonic

```text
PWM = 250 us

Decimation = 4

↓

1 ms/sample
```

---

## RX26T

```text
PWM = 62.5 us
```

---

### 표준

```c
tracer_sample_decimation = 16;
```

```text
1 ms/sample
```

---

### 고속 분석

```c
tracer_sample_decimation = 4;
```

```text
250 us/sample
```

---

### 장시간 분석

```c
tracer_sample_decimation = 64;
```

```text
4 ms/sample
```

---

# 8. 검증 절차

## Step 1

Watch

```text
tracer_monitor_enable = 1
```

설정

---

## Step 2

Signal 선택

예

```text
tracer_signal_mode = 4
```

---

## Step 3

Trigger 선택

예

```text
tracer_trigger_mode = 3
```

---

## Step 4

운전

---

## Step 5

확인

```text
tracer_enable

tracer_index

tracer_done

tracer_sequence
```

---

## Step 6

배열 확인

```text
tracer_ch1[0]

tracer_ch2[0]
```

---

## Step 7

CSV Export

---

## Step 8

Python Viewer 분석

---

# 9. 우선 검증 항목

## Test 1

```text
Trigger 1

Signal 4
```

---

## Test 2

```text
Trigger 3

Signal 4
```

---

## Test 3

```text
Trigger 3

Signal 6
```

---

## Test 4

```text
Trigger 4

Signal 6
```

---

## Test 5

```text
Trigger 9

Signal 9
```

---

# 최종 결론

✅ Panasonic Tracer Framework를 RX26T에 1차 포팅 완료

✅ 32bit 2CH 구조 적용 완료

✅ Signal Mode 이식 완료

✅ Trigger Hook 이식 완료

✅ Build / Link 성공

✅ 다음 단계는 Trigger 검증 → CSV Export → Python Viewer 검증

✅ 현재부터는 개발 단계가 아닌 검증 단계이다.
