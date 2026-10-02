# BLDC Sensorless 제어 흐름 및 탈조 원인 분석용 핵심 변수 가이드

## 1. 문서 목적

이 문서는 RX26T Fan Drive의 센서리스 BLDC 제어에서 다음 흐름을 Analysis Chart와 Watch로 추적하기 위한 핵심 변수를 정리한다.

```text
운전 명령
→ Align
→ I/F 기동
→ I/F–FOC 전환
→ FOC 전류제어
→ PLL/EEMF 센서리스 추정
→ 정상 속도제어
→ 탈조·기동 실패 검출
→ 정지 또는 재기동
```

목적은 단순히 정상 파형을 보는 것이 아니라, 각 단계에서 이상값이 발생했을 때 원인을 다음 범주로 구분하는 것이다.

- 운전 명령 전달 이상
- Align 전류 형성 이상
- I/F 기동 실패
- I/F–FOC 전환 실패
- 전류제어 이상
- PLL Lock 실패
- 속도 추정 불일치
- 탈조 검출
- 보호 제한 동작
- 재기동 반복

> 변수명은 현재 확인된 프로젝트 코드와 MAP 파일을 기준으로 작성하였다. 일부 변수는 전체 MAP 또는 선언부에서 최종 자료형과 주소를 추가 확인해야 한다.

---

## 2. 운전 명령 및 전체 상태

### 확인 항목

```text
Input_Target
Fan_TargetRpm
TargetSpeed
rpm_tgt
rpm_tgt_comm
rfan_ctl.byte
rpwm_ctl.byte
fDrvRun
start_flag
run_start
```

### 이상값으로 판단할 수 있는 원인

| 관찰 내용 | 우선 의심 원인 |
|---|---|
| `Input_Target`는 정상인데 `Fan_TargetRpm`이 0 | 목표속도 전달 또는 상위 통신 처리 이상 |
| `Fan_TargetRpm`은 정상인데 `rpm_tgt`가 0 | Fan 제어에서 PWM 제어로의 명령 전달 이상 |
| 목표속도는 있는데 `fDrvRun` 또는 `start_flag`가 활성화되지 않음 | 기동 조건 또는 운전 허가 조건 불충족 |
| 목표값이 0과 65535 사이를 반복 | 부호·자료형·통신 갱신·다른 루틴의 재기록 확인 |

---

## 3. Align 단계

### 확인 항목

```text
OP_Align_Current
Align_Current_Step[0]
Align_Current_Step[1]
Align_Current_Step[2]
B_ImagRef
IDS_mori
IQS_mori
B_Align_Fail_Timer.wCount
OP_Kp_Align
OP_Ki_Align
```

### 정상 확인 관점

```text
OP_Align_Current 설정
→ B_ImagRef 형성
→ Align 방향의 전류 상태 형성
→ 지정 시간 유지
→ I/F 기동으로 진행
```

### 이상값으로 판단할 수 있는 원인

| 관찰 내용 | 우선 의심 원인 |
|---|---|
| `OP_Align_Current`는 정상인데 `B_ImagRef`가 0 | Align 상태 진입 실패 또는 전류지령 전달 이상 |
| `B_ImagRef`는 발생하지만 전류 관련 값이 거의 0 | PWM 출력, 게이트 구동, 전류센싱 또는 전류루프 이상 |
| 전류 관련 값이 제한값까지 급상승 | 모터 결선, 전류 극성, 전류센싱 Offset, 게인 또는 PWM 문제 |
| Align 시간 후 I/F로 진행하지 않음 | Align 완료 조건 또는 타이머 상태 확인 |
| Align 실패가 반복되며 전류 단계가 증가 | `Align_Current_Step[]`과 실패 재시도 로직 확인 |

### 프로젝트 스케일 참고

```text
I_align[A] = OP_Align_Current / 10
```

예:

```text
OP_Align_Current = 7
→ Align 명령값 약 0.7 A
```

`OP_Align_Current`는 명령값이며 실제 측정 전류와 동일하다고 단정하지 않는다.

---

## 4. I/F 기동 단계

### 확인 항목

```text
OP_IF_START
IF_ImagRef
IF_ImagRef_Step[0]
IF_ImagRef_Step[1]
IF_ImagRef_Step[2]
B_WeRef
IF_B_WeRef_start
IF_B_WeRef_end
IF_B_WeRef_Kf_A
IF_B_WeRef_Kf_B
First_comm
start_cnt
start_fail
IF_StartFail_Flag
```

### 정상 확인 관점

```text
Align 종료
→ IF_ImagRef 형성
→ B_WeRef 증가
→ 회전자 가속
→ 추정속도 및 EEMF 증가
→ FOC 전환 조건 도달
```

### 이상값으로 판단할 수 있는 원인

| 관찰 내용 | 우선 의심 원인 |
|---|---|
| `IF_ImagRef`가 0 | I/F 기동 상태 미진입 또는 옵션 비활성 |
| `IF_ImagRef`는 존재하지만 `B_WeRef`가 증가하지 않음 | 속도 램프 또는 기동 타이머 이상 |
| `B_WeRef`는 증가하지만 추정속도가 0 부근 | 실제 회전 실패, 부하 과다, 전류 부족 또는 EEMF 추정 실패 |
| 전류는 증가하지만 회전수 추정이 반대 부호 | 상순서, 회전방향, 각도 진행 방향 또는 추정기 극성 확인 |
| 기동 중 `start_fail` 또는 `IF_StartFail_Flag` 활성 | 기동 실패 전류·속도·시간 판정 조건 확인 |
| 재시도마다 전류가 증가하지만 회전하지 않음 | 기계 구속, 과부하, 상결선, 게이트 출력 및 전류센싱 확인 |

---

## 5. I/F에서 FOC로의 전환

### 확인 항목

```text
IF_transition
IF_transition_Rpm_Start
IF_transition_Rpm_End
IF_Trans_pass
IF_Trans_fail
IF_trans_count
IF_Trans_Delay_Count
IF_Trans_Delay_Is
IF_Trans_Delay_Shift
B_WeRef
B_WeEst
B_WeEst_temp
theta_err_mori
```

### 정상 확인 관점

```text
I/F 속도 상승
→ 전환 시작속도 도달
→ PLL/EEMF 추정 유효성 확인
→ 전환 PASS 누적
→ FOC 제어 비중 증가
→ 전환 종료속도 도달
→ 센서리스 FOC 운전
```

### 프로젝트 기준 전환 영역

```text
I/F 기동 판단 시작/끝 : 120 / 150 rpm
FOC 전환 시작/끝      : 150 / 180 rpm
```

실제 EEPROM 및 코드 적용값과 일치하는지 반드시 확인한다.

### 이상값으로 판단할 수 있는 원인

| 관찰 내용 | 우선 의심 원인 |
|---|---|
| 전환속도에 도달해도 `IF_transition`이 변하지 않음 | 전환 조건, 옵션 또는 상태머신 이상 |
| `IF_Trans_pass`가 증가하지 않음 | 속도·각도오차·EEMF 판정 조건 불충족 |
| `IF_Trans_fail`이 증가 | PLL Lock 불안정 또는 속도추정 불일치 |
| 전환 직후 `IQS_mori` 및 속도가 급락 | I/F 전류와 FOC q축 전류의 연결 불연속 |
| 전환 직후 `theta_err_mori`가 급증 | 개루프각과 추정각 사이 불일치 |
| 전환 구간에서 반복 진입·이탈 | PASS/FAIL 카운터, 지연 및 히스테리시스 확인 |

---

## 6. FOC 전류제어

### 확인 항목

```text
B_ImagRef
IF_ImagRef
B_Ide
B_Iqe
B_Ias
B_Ibs
B_Ics
IDS_mori
IQS_mori
OP_INV_Limit.Ide
OP_INV_Limit.Iqe
OP_INV_Limit.Imag
OP_INV_Limit.OP_Iabcs
limit_cond.byte
VsMax
VsMid
VsMin
```

### 정상 확인 관점

```text
전류지령 생성
→ d/q축 전류제어
→ 상전류 형성
→ 전압지령 생성
→ 속도 및 부하에 따라 안정화
```

### 이상값으로 판단할 수 있는 원인

| 관찰 내용 | 우선 의심 원인 |
|---|---|
| 전류지령은 정상인데 `B_Iqe` 또는 상전류가 0 | PWM, 게이트, 전류센싱 또는 제어 실행 이상 |
| `B_Iqe`가 제한값에 지속적으로 붙음 | 부하 과다, 전류 제한, 속도 램프 과대 또는 토크 부족 |
| `B_Ide`가 예상과 다르게 크게 발생 | 각도 오차, dq축 변환 방향 또는 약계자 동작 확인 |
| 상전류 중 한 상만 비정상 | 상결선, 전류센서, ADC 입력 또는 PWM 상출력 확인 |
| `limit_cond.byte`가 활성화되며 속도 추종 악화 | 전류·전압 제한이 원인인지 확인 |
| `VsMax/VsMid/VsMin`이 포화 양상 | DC Link 부족, 전압포화, PWM 제한 또는 역기전력 증가 확인 |

### 스케일 참고

```text
B_Ias/B_Ibs/B_Ics = count / 2048 A
B_Ide/B_Iqe       = count / 2048 A
```

```text
IDS_mori/IQS_mori
→ Observer 내부 Q-format
→ 추가 검증 전 A 단위 직접 환산 금지
```

---

## 7. PLL 및 센서리스 추정 핵심

### 확인 항목

```text
theta_err_mori
theta_mori_int
wo_hat_mori
wr_hat_mori
B_WeEst
B_WeEst_temp
Error_sum_mori
Error_VDS_hat_sl
Error_VQS_hat_sl
VDS_sl_mori
VQS_sl_mori
OP_Filter_emf_ob
OP_Filter_wr
```

### 정상 확인 관점

```text
EEMF 성분 형성
→ theta_err_mori 감소
→ wo_hat_mori 안정화
→ wr_hat_mori가 속도지령을 추종
→ FOC 전환 후 각도오차가 제한범위 내 유지
```

### 이상값으로 판단할 수 있는 원인

| 관찰 내용 | 우선 의심 원인 |
|---|---|
| `theta_err_mori`가 감소하지 않고 발산 | PLL 부호, 회전방향, 좌표변환 또는 EEMF 오차 극성 확인 |
| `theta_err_mori`가 큰 진동을 반복 | PLL 게인 과대, 필터 부족 또는 EEMF 신호 품질 불량 |
| `wo_hat_mori`는 증가하지만 `wr_hat_mori`가 불안정 | 속도 필터, 스케일 또는 기계속도 변환 확인 |
| `B_WeEst_temp`와 `wr_hat_mori`의 추세가 불일치 | 추정속도 스케일, 필터 또는 변수 정의 차이 확인 |
| `Error_VDS_hat_sl` 또는 `Error_VQS_hat_sl`이 급증 | 모델 파라미터, 전압·전류 입력, 좌표각 또는 EEMF 추정 이상 |
| 저속에서는 불안정하고 고속에서 안정 | 저속 EEMF 부족 또는 전환 시점이 너무 이른 가능성 |
| 정상운전 중 갑자기 각도오차와 추정속도가 붕괴 | 부하 급변, 전압포화, 전류 제한 또는 탈조 진입 가능성 |

### 스케일 참고

```text
theta_err_mori [electrical degree]
= count × 180 / (π × 8192)
```

```text
B_WeEst [rpm]
= count × 0.15
```

```text
wo_hat_mori >> 6 [rpm]
= logged count × 0.01845703125

wr_hat_mori >> 6 [rpm]
= logged count × 0.01845703125
```

저장 대상이 원본 변수인지 Shift 결과인지 확인한 뒤 환산한다.

---

## 8. 속도 추종 및 정상운전

### 확인 항목

```text
Input_Target
Fan_TargetRpm
TargetSpeed
rpm_tgt
rpm_tgt_comm
B_WeRef
B_WeEst
B_WeEst_temp
wr_hat_mori
wo_hat_mori
mech_speed
Speed_Slope
Fan_CurrentRpm
```

### 이상값으로 판단할 수 있는 원인

| 관찰 내용 | 우선 의심 원인 |
|---|---|
| 목표값과 `B_WeRef`가 불일치 | 목표속도 경로 또는 스케일 문제 |
| `B_WeRef`는 정상인데 모든 추정속도가 낮음 | 토크 부족, 부하 과다 또는 전압·전류 제한 |
| 추정속도끼리 스케일만 다르고 추세는 동일 | 변수별 환산계수 차이 확인 |
| 정상운전 중 속도가 주기적으로 출렁임 | 속도 PI, PLL, 전류 제한 또는 부하주기 영향 확인 |
| 감속명령 후 속도가 늦게 감소 | Speed Slope, 관성, 회생제한 또는 감속 제어 확인 |

---

## 9. 기동 실패 및 탈조 검출

### 확인 항목

```text
start_fail
IF_StartFail_Flag
fdetect_pos_err
f_sensorless
IF_Trans_fail
taljo_restart_cnt
SYS_tmr_bldc_taljo1_1s
SYS_tmr_FAN1_taljo_cnt_clear_1s
SYS_tmr_FAN1_mech_speed_err_1s
rdc_restart_cnt
SYS_tmr_bldc_restart_1s
SYS_tmr_bldc_wind_restart
Inverse_rotation_Flag
First_comm
```

### 탈조 전후 핵심 이상값

| 관찰 내용 | 원인 추적 방향 |
|---|---|
| `theta_err_mori` 급증 후 속도추정 붕괴 | PLL Lock 상실 가능성 |
| `IQS_mori` 또는 `B_Iqe` 급증 후 제한 진입 | 부하 급증 또는 각도오차에 따른 토크전류 증가 |
| `B_WeRef`는 유지되지만 `B_WeEst_temp`와 `wr_hat_mori` 급감 | 실제 감속·정지 또는 추정기 붕괴 |
| `fdetect_pos_err` 활성 | 위치 또는 각도 추정오차 판정 로직 확인 |
| `IF_Trans_fail` 증가 후 재기동 | FOC 전환 실패 및 I/F 복귀 여부 확인 |
| `taljo_restart_cnt` 증가 | 탈조 판정 후 재기동 발생 |
| `rdc_restart_cnt` 증가 | 재기동 원인과 RDC 관련 상태 확인 |
| `Inverse_rotation_Flag` 활성 | 역회전 또는 풍차기동 조건 확인 |
| 재기동이 반복되고 정상운전 구간이 짧음 | 전환조건, PLL, 부하·전압 제한 및 보호조건 동시 확인 |

---

## 10. 보호 및 제한 원인

### 확인 항목

```text
OP_Start_Fail_Imag_Curr
OP_Start_Fail_Freq
OP_Speed_Reach_Fail_Imag
OP_SpeedErrorLimit
OP_CurrentErrorLimit
OP_current_limit
limit_cond.byte
FW_flag
VsMax
VsMid
VsMin
uc_error_code
mucCommErrorCode
```

### 이상값으로 판단할 수 있는 원인

| 관찰 내용 | 우선 의심 원인 |
|---|---|
| 기동 전류가 실패 판정값을 초과 | 기계구속, 과부하, 전류스케일 또는 판정값 확인 |
| 속도 도달 실패 플래그 또는 타이머 활성 | 기동토크 부족, 속도 램프 과대 또는 추정속도 오류 |
| 속도오차 제한 초과 | 목표·추정속도 불일치와 PLL 상태 확인 |
| 전류오차 제한 초과 | 전류루프, 전류센싱 또는 전압포화 확인 |
| `FW_flag` 활성과 동시에 추종 악화 | 약계자 진입조건 및 전압여유 확인 |
| 오류코드 발생 | 오류코드 정의표와 해당 보호 로직 확인 |

---

## 11. Analysis Chart 핵심 16채널

정상 제어 흐름과 1차 이상 원인을 동시에 보는 권장 세트다.

```text
ch1  Input_Target
ch2  Fan_TargetRpm
ch3  start_flag
ch4  OP_Align_Current
ch5  B_ImagRef
ch6  IF_ImagRef
ch7  B_WeRef
ch8  B_WeEst_temp
ch9  IDS_mori
ch10 IQS_mori
ch11 theta_err_mori
ch12 wo_hat_mori
ch13 wr_hat_mori
ch14 IF_transition
ch15 IF_Trans_fail
ch16 taljo_restart_cnt
```

---

## 12. 탈조 상세분석 추가 8채널

```text
ch17 Error_sum_mori
ch18 Error_VDS_hat_sl
ch19 Error_VQS_hat_sl
ch20 fdetect_pos_err
ch21 start_fail
ch22 IF_StartFail_Flag
ch23 mech_speed
ch24 rdc_restart_cnt
```

16채널 기본 세트와 추가 8채널을 함께 사용하면 다음을 시간축으로 연결할 수 있다.

```text
명령 입력
→ Align 전류
→ I/F 전류 및 속도 램프
→ FOC 전환
→ PLL 각도오차 및 추정속도
→ 전류 급변
→ 탈조 플래그
→ 재기동 카운트
```

---

## 13. 원인 판별용 최소 세트

채널 수를 최소화해야 할 경우 다음 10개를 우선 사용한다.

```text
Input_Target
OP_Align_Current
IF_ImagRef
B_ImagRef
B_WeRef
B_WeEst_temp
IQS_mori
theta_err_mori
wr_hat_mori
taljo_restart_cnt
```

FOC 전환 실패를 집중적으로 볼 때 추가한다.

```text
IF_transition
IF_Trans_pass
IF_Trans_fail
```

PLL 탈조를 집중적으로 볼 때 추가한다.

```text
wo_hat_mori
Error_sum_mori
Error_VDS_hat_sl
Error_VQS_hat_sl
fdetect_pos_err
```

---

## 14. 단계별 원인 판별 흐름

```text
[1] 목표 명령 정상?
Input_Target → Fan_TargetRpm → B_WeRef

아니오:
명령 경로·통신·자료형 확인

예:
↓

[2] Align 전류 형성?
OP_Align_Current → B_ImagRef → 전류 관련 값

아니오:
Align 상태·PWM·전류루프 확인

예:
↓

[3] I/F 기동 및 실제 가속?
IF_ImagRef + B_WeRef → B_WeEst_temp/wr_hat_mori 증가

아니오:
토크 부족·부하·상결선·EEMF 추정 확인

예:
↓

[4] FOC 전환 성공?
IF_transition + IF_Trans_pass + theta_err_mori

아니오:
전환속도·PLL Lock·각도 연속성 확인

예:
↓

[5] 정상운전 유지?
B_WeRef ≈ 추정속도, theta_err_mori 안정

아니오:
전류/전압 제한·PLL 게인·부하변동 확인

예:
↓

[6] 탈조 발생?
theta_err_mori 급증
+ 추정속도 급락
+ 전류 급변
+ fdetect_pos_err 또는 taljo_restart_cnt 증가

발생:
파형의 최초 이상 시점을 기준으로 원인 분류
```

---

## 15. 분석 시 주의사항

1. `OP_Rs`, `OP_Ld`, `OP_Lq`, `OP_KE` 등은 정적 EEPROM 파라미터이므로 실시간 이상 추적에서는 우선순위가 낮다.
2. 정적 파라미터는 모델 불일치가 의심될 때 별도 확인한다.
3. `IDS_mori`, `IQS_mori`는 현재 원시 Q-format으로 취급한다.
4. 서로 다른 스케일의 변수를 한 화면에서 볼 때 채널별 `Val/Div`와 `Offset`을 조정한다.
5. 탈조 원인은 마지막 플래그만 보지 말고, 플래그가 활성화되기 전 `theta_err_mori`, 속도추정, 전류, 제한 상태의 최초 변화를 확인한다.
6. 모터 정지 시에는 목표 회전수를 0 rpm으로 지정하고 실제 정지 및 출력 해제를 확인한 뒤 Debug Stop한다.
7. Analysis Chart CSV 분석 시 행 번호가 아니라 저장된 시간 정보를 기준으로 사건 순서를 판단한다.

---

## 16. 최종 권장 구성

### Watch Category

```text
Command
Align
IF Startup
FOC Current
PLL Observer
Speed
Fault & Restart
Protection
```

### Analysis Chart 기본 구성

```text
Input_Target
Fan_TargetRpm
OP_Align_Current
B_ImagRef
IF_ImagRef
B_WeRef
B_WeEst_temp
IDS_mori
IQS_mori
theta_err_mori
wo_hat_mori
wr_hat_mori
IF_transition
IF_Trans_fail
fdetect_pos_err
taljo_restart_cnt
```

### 탈조 분석 핵심 연결

```text
theta_err_mori 이상
→ wo_hat_mori/wr_hat_mori 불안정
→ B_WeEst_temp 급락 또는 진동
→ IQS_mori/B_ImagRef 급변
→ IF_Trans_fail 또는 fdetect_pos_err 활성
→ taljo_restart_cnt/rdc_restart_cnt 증가
```

이 연결을 시간축에서 확인하면 탈조가 **PLL Lock 상실에서 시작했는지**, **전류·전압 제한에서 시작했는지**, **기계 부하 또는 실제 회전 실패에서 시작했는지**, **FOC 전환 순간의 각도 불연속에서 시작했는지**를 구분하는 데 사용할 수 있다.
