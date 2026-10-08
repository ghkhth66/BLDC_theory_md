# RX26T Sensorless Startup Tracer Porting Guide

작성일 : 2026-10-08

---

# 1. 문서 목적

본 문서는 Panasonic 기반 Sensorless Startup Tracer를
Renesas RX26T 플랫폼으로 이식하면서 변경한 내용과
검증 절차를 정리한다.

본 Tracer는 단순 Waveform Logger가 아니다.

센서리스 제어의 핵심 이벤트인

- Align
- I/F Startup
- FOC Transition
- Stable Sensorless

구간을 기준으로

```text
무엇을 볼 것인가?
언제를 기준으로 할 것인가?
언제 저장할 것인가?
```

를 분리하여 분석하기 위한 시스템이다.

---

# 2. Tracer 기본 철학

Tracer는 다음 3축 구조를 가진다.

## Signal Mode

무엇을 저장할 것인가?

예)

```text
theta_err

wo_hat

wr_hat

Ide

IdeRef
```

---

## Trigger Mode

언제를 기준 시점으로 잡을 것인가?

예)

```text
FOC_START

FOC_COMPLETE

IF_OK
```

---

## Capture Start Mode

언제 저장을 시작할 것인가?

예)

```text
즉시

100ms 후

값 안정 이후
```

---

# 3. Porting 개요

## 기존 구조

```c
test1[4064]
test2[4064]
test3[4064]
test4[4064]
```

---

구조

```text
16 bit

4 CH

4064 Samples
```

---

총 메모리

```text
4064 × 2 × 4

=

32512 Byte
```

---

## 신규 구조

```c
tracer_ch1[4064]
tracer_ch2[4064]
```

---

구조

```text
32 bit

2 CH

4064 Samples
```

---

총 메모리

```text
4064 × 4 × 2

=

32512 Byte
```

---

동일 메모리 용량 유지

Observer 내부 32bit 신호 저장 가능

---

# 4. DD_INV_Tracer.h 변경 내용

## Signal Mode

```text
0 MANUAL

1 DQ_INPUT

2 EEMF_RAW

3 EEMF_FILTER

4 THETA_ERROR

5 PLL_OUTPUT

6 SPEED_ESTIMATION

7 ANGLE_ESTIMATION

8 PHASE_CURRENT

9 ALIGN_D_CURRENT
```

---

## Trigger Mode

```text
0 MANUAL

1 EEMF_FIRST_CALL

2 IF_OK

3 FOC_START

4 FOC_COMPLETE

5 SPEED_CHANGE

6 STABLE

7 ALIGN_START

8 IF_START

9 ALIGN_CURRENT_START
```

---

## Capture Mode

```text
0 IMMEDIATE

1 TIME_DELAY

2 VALUE_STABLE
```

---

# 5. DD_INV_Tracer.c 변경 내용

## Buffer

```c
volatile SLong tracer_ch1[4064];
volatile SLong tracer_ch2[4064];
```

---

## 추가 기능

### Trigger Event

```text
수동 시작

자동 Trigger

Delay Trigger

Stable Trigger
```

---

### Service

```text
1ms Monitor Service

Time Delay

Value Stable

Stable Trigger
```

---

# 6. DD_INV_PWM.c 변경 내용

Tracer와 실제 센서리스 제어를 연결하였다.

즉

```text
Observer

PLL

Current Control

Transition Logic
```

와 연결된다.

---

# 7. Trigger 의미

## Trigger 1

EEMF_FIRST_CALL

---

발생 위치

```c
DD_EEMF_sensorless()
```

---

의미

```text
Observer 최초 시작
```

---

분석 목적

```text
센서리스 계산 시작
```

---

## Trigger 2

IF_OK

---

발생 위치

```c
DD_IF_pwm_int_start_fail()
```

---

의미

```text
IF 기동 성공
```

---

분석 목적

```text
Observer 사용 가능 여부
```

---

## Trigger 3

FOC_START

---

의미

```text
강제각

↓

센서리스 각도
```

전환 시작

---

분석 목적

```text
탈조 발생 구간
```

---

## Trigger 4

FOC_COMPLETE

---

의미

```text
전환 완료
```

---

분석 목적

```text
FOC 안정성 검토
```

---

## Trigger 9

ALIGN_CURRENT_START

---

의미

```text
실제 D축 전류 흐름 시작
```

---

분석 목적

```text
Align 전류 확인
```

---

# 8. Signal Mode 의미

## Mode 4

Theta Error

```text
theta_err

Error_sum
```

---

관찰 목적

```text
PLL 상태
```

---

정상

```text
0으로 수렴
```

---

비정상

```text
진동

발산

포화
```

---

## Mode 5

PLL Output

```text
theta_err

wo_hat
```

---

관찰 목적

```text
PLL 응답
```

---

## Mode 6

Speed Estimation

```text
wo_hat

wr_hat
```

---

관찰 목적

```text
속도 추정 안정성
```

---

정상

```text
두 곡선 수렴
```

---

비정상

```text
지속적인 Gap
```

---

## Mode 9

Align D Current

```text
IdeRef

Ide
```

---

관찰 목적

```text
Align Current 추종
```

---

정상

```text
Ide ≈ IdeRef
```

---

비정상

```text
무응답

과도 진동

포화
```

---

# 9. Scale

## Current

```text
A

=

count / 2048
```

---

## Observer Speed

```text
rpm

=

logged count

×

0.01845703125
```

---

## B_WeEst

```text
rpm

=

count × 0.15
```

---

## Theta Error

```text
deg

=

count ×180

/

(pi×8192)
```

---

# 10. 센서리스 제어 관점 검증

센서리스 기동은 다음 4단계로 해석한다.

```text
Align

↓

I/F Startup

↓

FOC Transition

↓

Stable Sensorless
```

---

# 11. 검증 순서

## STEP 1

Framework 검증

```text
Manual Trigger
```

---

확인

```text
tracer_enable

tracer_done

tracer_index
```

---

PASS

```text
index=4064

done=1
```

---

## STEP 2

Signal 검증

Mode

```text
0~9
```

전부 확인

---

## STEP 3

Trigger 검증

```text
1

2

3

4

9
```

확인

---

PASS

```text
상승 Edge 1회
```

---

## STEP 4

FOC Transition 해석

권장

```text
Trigger 3

Mode 4
```

---

분석

```text
theta_err

Error_sum
```

---

정상

```text
theta_err 감소
```

---

비정상

```text
theta_err 발산
```

---

## STEP 5

PLL 분석

권장

```text
Trigger 3

Mode 6
```

---

정상

```text
wo_hat

↓

wr_hat

수렴
```

---

비정상

```text
Gap 지속
```

---

## STEP 6

Align 검증

권장

```text
Trigger 9

Mode 9
```

---

정상

```text
IdeRef

↓

Ide 추종
```

---

비정상

```text
Current saturation
```

---

# 12. 탈조 판정 기준

다음 중 하나 이상 발생

```text
theta_err 발산

PLL 포화

wo_hat 급반전

wr_hat 급락

Current 급증

FOC 전환 직후 재시동
```

---

판정

```text
Sensorless Transition Fail
```

---

# 13. 최종 완료 조건

## Build

```text
Error = 0
```

---

## Trigger

```text
1

2

3

4

9
```

PASS

---

## Signal

```text
0~9
```

PASS

---

## CSV Export

PASS

---

## Python Viewer

PASS

---

## 반복성

3회 이상 동일 결과

PASS

---

# 최종 결론

이번 RX26T Tracer는

```text
Panasonic Tracer Framework

↓

Renesas RX26T
```

로 성공적으로 포팅되었다.

핵심 목적은

```text
Align

I/F Startup

FOC Transition

Stable Sensorless
```

구간을 정량적으로 분석하여

센서리스 기동 실패,
PLL 불안정,
Gain 과대/과소,
Motor Parameter 불일치,
Current Sensing 이상

등을 진단하는 것이다.
