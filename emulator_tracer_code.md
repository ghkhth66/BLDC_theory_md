# RX26T Tracer Porting Strategy v1.0

## 문서 목적

본 문서는 Panasonic Micom 기반으로 개발 및 검증 완료된 Tracer Framework를 Renesas RX26T 프로젝트에 이식하기 위한 기준 문서이다.

중요한 전제는 다음과 같다.

```text
Golden Reference
=
Panasonic Micom Tracer

Porting Target
=
Renesas RX26T
```

즉,

```text
RX26T용 새로운 Tracer를 설계하는 것이 아니다.
```

현재 교재에 정리된

- Signal Mode
- Trigger Mode
- Capture Mode
- Scale
- Python Viewer
- 실측 검증 결과

를 Panasonic 버전의 표준(Specification)으로 정의하고 RX26T에 동일하게 이식한다.

---

# 개발 방향 재정의

잘못 이해하기 쉬운 방향

```text
RX26T Tracer
↓
표준
↓
Panasonic 적용
```

실제 방향

```text
Panasonic Tracer
↓
교재
↓
Python Viewer
↓
실측 검증 완료
↓
RX26T Porting
```

따라서 앞으로 진행 방향은

```text
Tracer 개발
```

이 아니라

```text
Panasonic Tracer Framework
↓
RX26T Porting
```

이다.

---

# Panasonic Tracer Framework 구조

Panasonic 버전은 이미 완성형 구조를 갖고 있다.

## Layer 1

Capture Engine

```c
Tracer_Start()
Tracer_Stop()
Tracer_Record()
```

---

## Layer 2

Trigger Engine

```c
Tracer_Trigger()
```

---

## Layer 3

Capture Start Mode

```c
IMMEDIATE
TIME_DELAY
VALUE_STABLE
```

---

## Layer 4

Monitor Service

```c
Tracer_MonitorService_1ms()
```

---

## Layer 5

STABLE Trigger

```c
Tracer_StableService_1ms()
```

---

## Layer 6

Signal Mode

```c
switch(tracer_signal_mode)
{
}
```

구조

---

# Signal Mode (고정)

다음 Signal 번호는 Panasonic과 RX26T가 반드시 동일하게 유지한다.

```text
0 THETA_SPEED

1 DQ_INPUT

2 EEMF_RAW

3 EEMF_FILTERED

4 THETA_ERROR

5 PLL_OUTPUT

6 SPEED_ESTIMATION

7 ANGLE_ESTIMATION

8 PHASE_CURRENT

9 ALIGN_D_CURRENT
```

---

# Trigger Mode (고정)

다음 Trigger 번호도 반드시 동일하게 유지한다.

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

# Capture Start Mode (고정)

```text
0 IMMEDIATE

1 TIME_DELAY

2 VALUE_STABLE
```

---

# RX26T 이식 시 변경하지 않는 것

다음 항목은 Panasonic 기준을 그대로 유지한다.

```text
Signal Mode 번호

Trigger Mode 번호

Capture Start Mode

Tracer State Machine

Python Viewer

CSV 구조

교재

시험 절차

그래프 정의
```

---

# RX26T 이식 시 변경하는 것

다음 항목만 변경한다.

```text
실제 변수명

실제 Trigger Hook 위치

Getter 함수

MCU 의존 코드

메모리 배치
```

---

# 버퍼 구조

## Panasonic

```c
volatile SLong tracer_ch1[];
volatile SLong tracer_ch2[];
```

구조

즉

```text
32bit
2ch
```

이다.

---

## RX26T 현재 임시 구조

현재는

```c
test1[4064]
test2[4064]
test3[4064]
test4[4064]
```

4채널 실험 구조를 사용하고 있다.

---

# 최종 권장 구조

RX26T도 Panasonic과 동일하게

```c
volatile SLong tracer_ch1[];
volatile SLong tracer_ch2[];
```

구조 유지

---

# 32bit 2CH를 유지하는 이유

현재 Tracer 변수들은 대부분 32bit 기반이다.

예

```c
theta_mori

theta_err_mori

Error_sum_mori

wo_hat_mori

wr_hat_mori

Error_VDS_hat_sl

Error_VQS_hat_sl
```

16bit 저장 시 발생 가능한 문제

```text
Overflow

Wrap Around

부호 반전

데이터 손실
```

따라서

```text
32bit
2CH
```

구조가 정답이다.

---

# 측정 주기

## Panasonic

기준

```text
PWM = 250 us
```

저장

```text
Decimation = 4
```

따라서

```text
250 us × 4

=

1 ms/sample
```

---

# RX26T

현재

```text
PWM INT = 62.5 us
```

이다.

동일한 시간축을 유지하려면

```text
62.5 us × 16

=

1000 us

=

1 ms/sample
```

따라서

```c
TRACER_DECIMATION = 16
```

가 된다.

---

# 핵심 결론

Panasonic

```text
250 us PWM

Decimation 4

↓

1 ms/sample
```

RX26T

```text
62.5 us PWM

Decimation 16

↓

1 ms/sample
```

결과

```text
동일한 시간축
```

이 된다.

---

# 저장 길이

## Panasonic

```text
64 samples
```

```text
1 ms/sample
```

```text
64 ms
```

관측

---

## RX26T

현재

```text
4064 samples
```

사용 가능

```text
1 ms/sample
```

기준

```text
4064 ms

=

4.064 sec
```

관측 가능

---

# RX26T 권장 운용

## 고속 분석

```text
Decimation = 4

62.5 us × 4

=

250 us/sample
```

```text
4064 samples

=

1.016 sec
```

FOC 전환 직전/직후 분석

---

## 표준 분석

```text
Decimation = 16

62.5 us × 16

=

1 ms/sample
```

```text
4064 samples

=

4.064 sec
```

Panasonic 표준 시간축

---

## 장시간 분석

```text
Decimation = 64

62.5 us × 64

=

4 ms/sample
```

```text
4064 samples

=

16.256 sec
```

---

## 초장시간 분석

```text
Decimation = 256

62.5 us × 256

=

16 ms/sample
```

```text
4064 samples

=

65.024 sec
```

---

# Scale 정책

## 확정된 변수

### 속도

```c
Input_Target
```

```text
rpm = count × 10
```

---

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

### 전류

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

```c
B_IdeRef
B_IqeRef
```

```text
A = count / 2048
```

---

### Align Current

```c
OP_Align_Current
```

```text
A = value / 10
```

---

### 각도

```c
theta_err_mori
```

```text
deg_e

=

count × 180

/

(π × 8192)
```

---

# 내부 Q-format 확인됨

다음은 Q-format 자체가 확인된 변수이다.

```c
IDS_mori
IQS_mori
```

```text
Q8
```

---

```c
VDS_sl_mori
VQS_sl_mori
```

```text
Q10
```

---

```c
Error_VDS_sl
Error_VQS_sl
```

```text
Q10
```

---

```c
Error_VDS_hat_sl
Error_VQS_hat_sl
```

```text
Q10
```

---

# 아직 물리 단위 환산 미확정

아래 변수들은 Count 유지

```c
IDS_mori

IQS_mori

Error_VDS_sl

Error_VQS_sl

Error_VDS_hat_sl

Error_VQS_hat_sl

Error_sum_mori
```

---

# Data 수집 방식

## Panasonic

```text
Watch

↓

Tracer

↓

CSV

↓

Python Viewer
```

---

## RX26T

```text
CS+

Watch

↓

Tracer

↓

CSV

↓

Python Viewer
```

---

수집 방법은 다를 수 있다.

하지만 최종적으로 필요한 것은 동일하다.

```text
Signal Mode

Trigger Mode

CH1

CH2

Time Axis
```

즉

```text
교재

Python Viewer

분석 절차
```

를 그대로 사용할 수 있다.

---

# 최종 목표

Panasonic

```text
Golden Reference
```

---

RX26T

```text
Porting Target
```

---

최종 상태

```text
Signal Mode 0~9

Trigger Mode 0~9

Capture Mode

32bit 2CH

동일 Python Viewer

동일 교재

동일 분석 절차
```

완전 호환

---

# 최종 결론

✅ RX26T도 최종적으로 32bit 2CH 구조를 사용한다.

✅ Signal / Trigger / Capture 정의는 Panasonic 기준을 그대로 사용한다.

✅ 측정 시간축은 Panasonic과 동일하게 1 ms/sample 기준으로 맞춘다.

✅ RX26T에서는 PWM = 62.5 us 이므로 Decimation = 16 이 기본값이다.

✅ 필요 시 Decimation = 4 를 사용하여 250 us 분석도 지원한다.

✅ Scale 정의는 Panasonic 교재를 기준으로 유지한다.

✅ 내부 Q-format 변수는 추가 확인 전 Count 유지한다.

✅ MCU가 달라도 Watch 설정 → Capture → CSV → Python 분석 흐름은 동일하다.

✅ 앞으로의 작업은 Tracer 재설계가 아니라 Panasonic Tracer Framework의 RX26T Porting이다.
