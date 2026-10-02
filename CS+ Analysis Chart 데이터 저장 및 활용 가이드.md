# CS+ Analysis Chart 데이터 저장 및 활용 가이드

## 1. 문서 목적

이 문서는 RX26T Fan Drive 프로젝트에서 CS+ Program Analyzer의 **Watch Category**와 **Analysis Chart**를 이용해 다음 과정을 안전하고 반복 가능하게 수행하는 방법을 정리한다.

- 모터 목표 회전수 입력
- Align 구간 확인
- I/F 기동 구간 확인
- FOC 전환 확인
- 센서리스 정상 운전 구간 확인
- 목표 회전수 0 rpm 지령 후 안전 정지
- Debug Stop 후 Analysis Chart 데이터 저장
- CSV/XLS/MTAC 파일 활용 및 재분석

이 문서는 다음 자료와 실제 확인 내용을 결합한다.

- Renesas `CS+ V8.11.00 Integrated Development Environment User’s Manual: Analysis Tool`
- RX26T + E2 Lite 실제 Analysis Chart 화면
- Watch Category 구성 및 채널별 `Val/Div` 조정 결과
- BLDC Sensorless Startup 분석 관점

---

## 2. 전체 측정 흐름

권장 측정 순서는 다음과 같다.

```text
1. Watch 변수 및 Category 구성
2. Analysis Chart 채널 등록
3. Analysis method = Real-time sampling 확인
4. Sampling 시작
5. 목표 회전수 입력
6. Align 진행 확인
7. I/F 기동 진행 확인
8. FOC 전환 확인
9. 정상 운전 구간 데이터 축적
10. 목표 회전수 0 rpm 입력
11. 실제 모터 정지 확인
12. PWM 및 운전 상태 해제 확인
13. Debug Stop
14. Analysis Chart 데이터 저장
15. CSV/XLS/MTAC 파일 확인 및 후처리
```

> **안전 핵심:** 모터가 회전 중인 상태에서 바로 Debug Stop을 누르지 않는다. 먼저 목표 회전수를 0 rpm으로 지정하고 실제 회전 정지 및 출력 해제를 확인한 뒤 Debug Stop을 수행한다.

---

## 3. Analysis Chart 데이터 수집 구조

Real-time sampling 사용 시 데이터 흐름은 다음과 같다.

```text
MCU SRAM 변수
    ↓
E2 Lite의 RRM/RAM Monitor 기능
    ↓
CS+ Program Analyzer
    ↓
Analysis Chart 내부 샘플 버퍼
    ↓
화면 그래프 및 파일 저장
```

### 3.1 MCU 측 변수

예를 들어 다음 변수는 MCU SRAM에 존재하는 실행 중 변수다.

```text
B_ImagRef
IF_ImagRef
IDS_mori
IQS_mori
B_WeRef
B_WeEst_temp
wo_hat_mori
wr_hat_mori
theta_err_mori
```

Analysis Chart는 이 변수들을 별도의 펌웨어 로그 배열에 복사하지 않고, 디버거를 통해 주기적으로 읽어 그래프로 표시한다.

### 3.2 CS+ 내부 버퍼

Analysis Chart는 취득한 그래프 데이터를 CS+ 내부 버퍼에 유지한다. 매뉴얼에서는 그래프 데이터가 버퍼 용량인 **10,000 plots**를 초과하면 오래된 데이터부터 덮어쓰는 ring buffer 방식이라고 설명한다.

따라서 장시간 연속 측정보다 다음과 같이 필요한 사건을 중심으로 구간을 관리하는 것이 좋다.

```text
Sampling 시작
→ 기동
→ FOC 전환
→ 정상구간 확보
→ 0 rpm 정지
→ Debug Stop
→ 즉시 저장
```

---

## 4. Watch Category 구성

Watch Category는 Watch 창에서 변수를 목적별로 정리하는 폴더 기능이다.

### 4.1 권장 Category 구조

```text
Speed
Current
PLL
Startup
Align
Protection
etc
```

### 4.2 Speed Category

```text
Input_Target
Fan_TargetRpm
TargetSpeed
B_WeRef
B_WeEst_temp
wo_hat_mori
wr_hat_mori
```

확인 목적:

- 외부 또는 수동 목표값이 운전 목표값으로 전달되는지
- I/F 속도 지령이 증가하는지
- 추정 속도가 지령을 따라오는지
- FOC 전환 후 속도가 안정되는지

### 4.3 Current Category

```text
B_ImagRef
IF_ImagRef
IDS_mori
IQS_mori
OP_Align_Current
```

확인 목적:

- Align 전류 명령이 생성되는지
- I/F 기동 전류 명령이 생성되는지
- d축 및 q축 내부 전류 관련 값이 변화하는지

> `IDS_mori`, `IQS_mori`는 Observer 내부 Q-format이므로 추가 검증 전 A 단위로 직접 환산하지 않고 원시값으로 분석한다.

### 4.4 PLL Category

```text
theta_err_mori
theta_mori_int
wo_hat_mori
wr_hat_mori
Error_sum_mori
Error_VDS_hat_sl
Error_VQS_hat_sl
VDS_sl_mori
VQS_sl_mori
```

확인 목적:

- 각도 오차 수렴 여부
- PLL 속도 상태 변화
- 추정각 및 추정속도 안정도
- FOC 전환 후 센서리스 Lock 상태

### 4.5 Startup Category

```text
start_flag
start_cnt
start_fail
IF_transition
IF_Trans_pass
IF_Trans_fail
First_comm
taljo_restart_cnt
```

확인 목적:

- 기동 루틴 진입 여부
- Align에서 I/F로 진행되는지
- I/F에서 FOC로 전환되는지
- 전환 PASS/FAIL 및 재기동 발생 여부

### 4.6 Align Category

```text
OP_Kp_Align
OP_Ki_Align
OP_Align_Current
Align_Current_Step
B_Tmin
```

확인 목적:

- EEPROM에서 로딩된 Align 관련 설정
- Align 전류 명령 및 단계값
- PWM 최소시간 관련 값

### 4.7 Category에 변수 넣기

1. Watch 창에서 `Create Category`로 Category를 만든다.
2. 기존 변수를 마우스로 선택한다.
3. 여러 변수를 옮길 경우 Ctrl 또는 Shift로 다중 선택한다.
4. 선택한 변수를 원하는 Category 이름 위로 Drag & Drop한다.
5. Category 왼쪽의 삼각형으로 접기/펼치기를 확인한다.

정상적으로 들어가면 다음처럼 하위 노드로 표시된다.

```text
Speed
 ├ Input_Target
 ├ Fan_TargetRpm
 ├ B_WeRef
 └ B_WeEst_temp
```

### 4.8 Category의 역할과 제한

Category는 Watch 창의 정리 및 탐색을 위한 기능이다.

```text
Category 펼침/접힘
    ├ Watch 화면 표시에는 영향 있음
    └ Analysis Chart의 데이터 수집에는 직접 영향 없음
```

즉 Category를 접었다고 그래프 데이터 수집이 중단되지 않으며, 펼쳤다고 자동으로 해당 Category만 Plot되는 것도 아니다.

---

## 5. Watch 변수와 Analysis Chart 채널 등록

Analysis Chart는 최대 32개 채널을 제공한다.

```text
ch1 ~ ch32
```

각 채널에는 변수, 레지스터 또는 주소 표현식 하나를 등록할 수 있다.

### 5.1 Reflect 사용

Analysis Chart의 `Reflect` 버튼은 Watch1에 등록된 Watch Expression을 위에서부터 최대 32개까지 채널에 자동 등록한다.

```text
Watch1
    ↓ Reflect
Analysis Chart ch1~ch32
```

주의사항:

- `Reflect`를 누르면 기존 채널 등록과 기존 그래프 데이터가 초기화될 수 있다.
- 여러 Category가 하나의 Watch1에 있으면 Category별 선택이 아니라 Watch1의 등록 항목이 채널에 반영된다.
- 필요한 채널만 정확히 구성하려면 변수별 Drag & Drop 또는 Property의 `Variable/Address` 직접 입력이 더 확실하다.

### 5.2 직접 채널 등록

다음 패널에서 변수를 Analysis Chart의 채널 번호 또는 변수명 영역으로 Drag & Drop할 수 있다.

```text
Variable List
Editor
Watch
CPU Register
IOR
```

또는 다음 경로에서 직접 입력할 수 있다.

```text
Project Tree
→ Program Analyzer (Analyze Tool)
→ Property
→ Variable Value Changing
→ Channel 1~32
→ Variable/Address
```

### 5.3 표시할 채널 선택

Analysis Chart 하단의 각 채널에는 체크박스가 있다.

```text
체크 ON  = 그래프 표시
체크 OFF = 그래프 숨김
```

32개 채널을 모두 등록하더라도 실제 화면에는 필요한 몇 개만 체크해서 볼 수 있다.

예시:

```text
Speed 확인
✓ Input_Target
✓ Fan_TargetRpm
✓ B_WeRef
✓ B_WeEst_temp
✓ wr_hat_mori

Current 및 PLL 변수는 표시 체크 해제
```

> 체크 해제는 화면 표시만 숨기는 것이며, 등록 채널 자체를 삭제하는 것과는 다르다.

---

## 6. Analysis Chart 기본 설정

### 6.1 Analysis Chart 열기

```text
View
→ Program Analyzer
→ Analysis Chart
→ Variable Value Changing Chart
```

### 6.2 Real-time Sampling 설정

Property 경로:

```text
Project Tree
→ Program Analyzer (Analyze Tool)
→ Property
→ Variable Value Changing
→ General
```

주요 항목:

```text
Analysis method = Real-time sampling
Start/stop real-time sampling = Sync 또는 Manual
Auto adjustment = 필요에 따라 설정
Time per grid [Time/Div]
Chart type
```

### 6.3 Sync와 Manual

```text
Sync
- 프로그램 실행/정지에 맞춰 Sampling 시작/정지

Manual
- Analysis Chart의 Sampling 버튼으로 별도 시작/정지
```

시험 절차를 명확히 통제하려면 Manual이 편리할 수 있고, 프로그램 Run/Break와 연동하려면 Sync가 편리하다.

---

## 7. Y축 스케일 조정

서로 크기가 다른 변수를 한 화면에 표시하면 작은 값이 평평하게 보일 수 있다.

예:

```text
Fan_TargetRpm      : 수십 count
OP_Rs              : 수백~수천 count
IDS_mori/IQS_mori  : 큰 내부 Q-format 값
```

각 채널은 별도의 `Val/Div`와 `Offset`을 가진다.

### 7.1 수동 스케일

```text
Program Analyzer
→ Property
→ Variable Value Changing
→ General
→ Auto adjustment = None
```

그 다음 원하는 채널에서:

```text
Channel N
→ Value per grid [Val/Div] N
→ Offset N
```

을 조정한다.

### 7.2 자동 맞춤

프로그램 정지 상태에서 채널 하단의 `Val/Div` 표시를 더블 클릭하면 해당 채널 데이터에 맞춰 Y축 범위를 자동 조정할 수 있다.

### 7.3 마우스 조정

프로그램 정지 상태에서 대상 그래프를 선택하고 Ctrl + Mouse Wheel로 Y축 확대/축소가 가능하다.

### 7.4 표시 해석

`OP_Align_Current = 7`이 계속 유지된다면 그래프는 값 7 위치의 수평선으로 나타난다.

```text
값이 고정됨 = 수평선
값이 변화함 = 시간축에 따른 파형
```

프로젝트 기준 Align 전류 명령의 물리값은 다음과 같다.

```text
I_align[A] = OP_Align_Current / 10
```

따라서 원시값 7은 명령 기준 0.7 A이다. 단, `OP_Align_Current`는 전류 명령값이며 실제 측정 전류와 동일하다고 단정하면 안 된다.

---

## 8. 권장 채널 구성

### 8.1 Startup Overview

```text
ch1  Input_Target
ch2  Fan_TargetRpm
ch3  B_WeRef
ch4  B_WeEst_temp
ch5  B_ImagRef
ch6  IF_ImagRef
ch7  IDS_mori
ch8  IQS_mori
ch9  theta_err_mori
ch10 wo_hat_mori
ch11 wr_hat_mori
ch12 IF_transition
ch13 IF_Trans_pass
ch14 IF_Trans_fail
ch15 start_flag
ch16 start_fail
```

### 8.2 Align 및 전류 확인

```text
OP_Align_Current
B_ImagRef
IF_ImagRef
IDS_mori
IQS_mori
```

### 8.3 PLL 및 속도 확인

```text
B_WeRef
B_WeEst_temp
theta_err_mori
wo_hat_mori
wr_hat_mori
Error_sum_mori
```

### 8.4 정적 파라미터 취급

다음 변수는 EEPROM에서 로딩된 정적 파라미터이므로 운전 중 변화 분석용 Plot에는 우선순위가 낮다.

```text
OP_Rs
OP_Ld
OP_Lq
OP_KE
OP_Kp_Align
```

정적 파라미터 확인이 목적이 아니라면 그래프 표시 체크를 해제하고, 변화하는 운전 변수를 우선 표시한다.

---

## 9. 안전한 기동 및 정지 절차

### 9.1 기동 전

```text
1. 비상 정지 및 전원 차단 수단 확인
2. 목표 회전수 0 확인
3. Watch 변수 값 확인
4. Analysis Chart 채널 확인
5. Sampling 시작
```

### 9.2 기동 중

```text
1. 목표 회전수 입력
2. Align 전류 명령 확인
3. I/F 속도 및 전류 지령 확인
4. FOC 전환 상태 확인
5. 정상 속도 구간 확인
6. 필요한 구간만 데이터 축적
```

### 9.3 정지 및 저장

```text
1. 목표 회전수를 0 rpm으로 지정
2. 목표속도와 추정속도가 감소하는지 확인
3. 실제 모터가 완전히 정지했는지 확인
4. PWM 및 운전 출력이 해제되었는지 확인
5. Debug Stop 수행
6. Analysis Chart에 데이터가 남아 있는지 확인
7. Analysis Chart에 포커스를 둔 상태에서 저장
```

> 전력회로 및 회전체의 안전은 소프트웨어 상태만으로 판단하지 않는다. 실제 회전체 정지, 게이트 출력 해제 및 장비의 안전 절차를 함께 확인한다.

---

## 10. Analysis Chart 데이터 저장

### 10.1 포커스가 중요한 이유

CS+의 `File` 메뉴는 현재 활성화된 패널에 따라 저장 메뉴가 달라진다.

```text
Watch1 활성화
→ Save Watch Data / Save Watch Data As...

Analysis Chart 활성화
→ Save Analysis Chart Data / Save Analysis Chart Data As...
```

따라서 그래프 시간축 데이터를 저장하려면 반드시 Analysis Chart 내부를 클릭하여 Analysis Chart에 포커스를 준다.

### 10.2 저장 절차

```text
1. 모터 0 rpm 정지 확인
2. Debug Stop
3. Analysis Chart 그래프 영역 클릭
4. 상단 File 메뉴 선택
5. Save Analysis Chart Data As... 선택
6. 저장 형식 선택
7. 파일명과 저장 위치 지정
```

### 10.3 저장 형식

CS+ Analysis Tool 매뉴얼에 기재된 Analysis Chart 저장 형식은 다음과 같다.

```text
Text file (*.txt)
CSV (*.csv)
Microsoft Office Excel Workbook (*.xls)
Analysis Chart Data (*.mtac)
Bitmap (*.bmp)
JPEG (*.jpg)
PNG (*.png)
EMF (*.emf)
```

파일 형식별 목적:

| 형식 | 주 용도 |
|---|---|
| TXT | 텍스트 데이터 확인 및 단순 가공 |
| CSV | Python, Excel, 데이터 분석 도구에서 후처리 |
| XLS | Excel에서 직접 검토 |
| MTAC | CS+에서 그래프 데이터와 채널 설정 복원 |
| BMP/JPEG/PNG/EMF | 보고서 및 화면 이미지 저장 |

### 10.4 Watch Data와 Analysis Chart Data 구분

```text
Save Expanded Watch Data...
- 현재 Watch 값 및 확장 결과의 텍스트 출력
- Watch Expression 목록 백업 파일이 아님
- Import Watch Expression으로 다시 읽을 수 없음

Save Watch Data As...
- Watch 창의 현재 표시 데이터 저장
- Analysis Chart 시간축 샘플 저장과 다름

Save Analysis Chart Data As...
- Analysis Chart의 시간축 그래프 데이터 저장
- CSV/XLS/MTAC 등으로 저장 가능
```

`Save Expanded Watch Data`로 저장한 TXT를 `Import Watch Expression`으로 불러오면 다음 오류가 발생할 수 있다.

```text
Failed to import watch data.
This is not an importable watch data file.
(E0615000)
```

이 오류는 저장 파일의 목적과 Import 파일 형식이 다르기 때문에 발생한다.

---

## 11. 저장 파일 검증

저장 직후 CSV 또는 XLS 파일을 열어 다음을 확인한다.

### 11.1 필수 확인

```text
1. 시간 정보가 존재하는가
2. 여러 행의 샘플이 저장되었는가
3. 선택 채널의 변수명이 포함되어 있는가
4. 기동 전, Align, I/F, FOC 전환, 정상 운전, 정지 구간이 포함되었는가
5. 값이 동일한 마지막 상태 한 줄만 저장된 것은 아닌가
```

### 11.2 저장 대상 판별

```text
시간에 따라 여러 행이 존재
→ Analysis Chart 데이터일 가능성 높음

변수별 현재값 중심의 단일 상태 목록
→ Watch Data 저장 가능성 높음
```

### 11.3 데이터 간격 확인

Real-time sampling의 값은 CS+ 및 디버거의 표시 업데이트/샘플링 조건에 따른다. 따라서 기존 펌웨어 Tracer의 고정 `1 ms/sample`과 동일하다고 가정하지 말고, 저장 파일의 실제 시간 정보를 기준으로 분석한다.

---

## 12. 저장 데이터 불러오기

### 12.1 MTAC 불러오기

```text
Project Tree
→ Program Analyzer (Analyze Tool)
→ Property
→ Variable Value Changing
→ General
→ Analysis method = Load from file
→ Analysis chart data file = 저장한 *.mtac
```

이 방식은 CS+에서 저장한 그래프와 설정을 복원할 때 사용한다.

MTAC에 저장되는 주요 항목:

```text
채널별 그래프 데이터
채널 변수명
Type/Size
Val/Div
Offset
색상
Time/Div
채널 표시 상태 관련 정보
```

그래프 데이터가 없는 채널은 저장되지 않을 수 있으며, 해당 채널에는 기본 설정이 적용될 수 있다.

### 12.2 실시간 측정으로 복귀

MTAC 확인을 마친 뒤 새 측정을 시작하려면:

```text
Analysis method = Real-time sampling
```

으로 되돌린다.

### 12.3 CSV/XLS 불러오기

CSV/XLS는 CS+에서 그래프 복원보다는 외부 분석용으로 사용하는 것이 적합하다.

```text
CSV
→ Excel 확인
→ Python/pandas 로딩
→ 시간축 정리
→ 구간 검출
→ KPI 계산
→ 그래프 생성
```

---

## 13. CSV 후처리 분석 방향

저장된 Analysis Chart CSV에 시간 및 채널 값이 포함되면 기존 Tracer 분석 절차와 유사하게 사용할 수 있다.

### 13.1 구간 분할

```text
정지 초기조건
Align
I/F 가속
FOC 전환
센서리스 정상 운전
감속 및 0 rpm 정지
```

### 13.2 주요 분석값

#### 속도

```text
Input_Target
Fan_TargetRpm
B_WeRef
B_WeEst_temp
wo_hat_mori
wr_hat_mori
```

프로젝트 기준 환산 참고:

```text
Input_Target : 1 count = 10 rpm
B_WeEst      : rpm = count × 0.15
wo_hat_mori >> 6 : rpm = logged count × 0.01845703125
wr_hat_mori >> 6 : rpm = logged count × 0.01845703125
```

저장 대상이 `wo_hat_mori`, `wr_hat_mori` 원본인지 shift 결과인지 반드시 확인한다.

#### 각도 오차

```text
theta_err_mori [electrical degree]
= count × 180 / (π × 8192)
```

#### 전류

```text
B_ImagRef 등 OP_Iscale 기반 값
I[A] = count / 2048
```

단:

```text
IDS_mori
IQS_mori
```

는 Observer 내부 Q-format이므로 별도 검증 전 A 단위로 직접 변환하지 않는다.

### 13.3 권장 KPI

```text
Align 유지시간
I/F 기동시간
FOC 전환 시점
목표속도 도달시간
속도 오버슈트
정상상태 속도 오차
전류 피크
전류 안정화시간
theta_err_mori 피크 및 수렴시간
재기동 또는 전환 실패 횟수
```

실제 샘플링 간격이 일정하지 않거나 손실 구간이 있으면, 행 번호가 아니라 저장된 시간 정보를 기준으로 계산한다.

---

## 14. Trigger 기능 활용

Real-time sampling에서는 Trigger 기능을 사용할 수 있다.

Property 경로:

```text
Program Analyzer
→ Property
→ Variable Value Changing
→ Trigger
```

주요 항목:

```text
Use trigger function
Trigger mode
Trigger source
Trigger level
Direction of trigger edge
Trigger position
```

Trigger mode:

```text
Auto
- 주기적으로 그래프를 갱신하고 Trigger 발생 전후 데이터를 표시

Single
- 첫 Trigger에 대해 한 번 표시 후 Sampling 정지

Normal
- Trigger가 발생할 때마다 그래프 갱신
```

기동시험 예:

```text
Trigger source = start_flag 또는 IF_transition
Trigger level  = 상태가 바뀌는 기준값
Edge           = Rising
Mode           = Single
```

단, 실제 적용 전 해당 상태변수의 값 범위와 전환 방향을 확인한다.

---

## 15. Zoom1~Zoom4 활용

Zoom1~Zoom4는 서로 다른 변수 그룹을 가지는 기능이 아니다.

```text
Zoom1~Zoom4
= 같은 채널 데이터의 서로 다른 확대 영역
```

활용 예:

```text
Zoom1 = Align 구간
Zoom2 = I/F 가속 구간
Zoom3 = FOC 전환 구간
Zoom4 = 정상 운전 또는 정지 구간
```

즉 Speed, Current, PLL 그룹 분리는 채널 체크 ON/OFF 또는 별도 채널 패턴으로 수행하고, Zoom은 시간/값 구간 확대에 사용한다.

---

## 16. 결과 저장 및 파일명 규칙 권장안

파일명 예:

```text
YYYYMMDD_TestID_TargetRPM_AnalysisChart.csv
YYYYMMDD_TestID_TargetRPM_AnalysisChart.mtac
YYYYMMDD_TestID_TargetRPM_AnalysisChart.png
```

예:

```text
20261002_STARTUP_300rpm_AnalysisChart.csv
20261002_STARTUP_300rpm_AnalysisChart.mtac
```

같이 기록할 메타정보:

```text
MCU / 프로젝트 버전
EEPROM 버전
대상 모터
목표 회전수
PWM 설정
Sampling 방식
사용 채널
시험 조건
PASS/FAIL
특이사항
```

---

## 17. 문제 해결 체크리스트

### 그래프가 평평함

```text
□ 값이 실제로 고정되어 있는가
□ Val/Div가 너무 큰가
□ Offset이 부적절한가
□ 다른 대형 스케일 채널 때문에 상대적으로 작아 보이는가
□ 정적 EEPROM 파라미터만 표시 중인가
```

### 그래프가 보이지 않음

```text
□ 채널 체크박스가 선택되었는가
□ 변수/주소가 올바른가
□ Type/Size가 맞는가
□ 프로그램이 실행 중인가
□ Sampling이 시작되었는가
□ RRM/RAM Monitor가 활성화되었는가
□ 측정값이 Y축 표시범위 밖에 있는가
```

### 저장 메뉴가 Watch Data로 표시됨

```text
원인:
Watch1 패널에 포커스가 있음

조치:
Analysis Chart 그래프 영역 클릭
→ File 메뉴 다시 확인
```

### 저장한 TXT를 Watch로 Import할 수 없음

```text
원인:
Save Expanded Watch Data 파일은 Watch Expression Import 형식이 아님

조치:
Watch 구성은 프로젝트 저장으로 보존
Analysis Chart 데이터는 Save Analysis Chart Data As로 별도 저장
```

### Sampling 손실 또는 빈 구간

CS+ 매뉴얼은 Real-time sampling 중 데이터 획득 실패 시 선이 연결되지 않고 시간 정보만 표시될 수 있다고 설명한다. 대상 변수 수를 줄이고, 업데이트 간격 및 디버거 연결 상태를 점검한다.

---

## 18. 최종 권장 운용 방식

매 시험마다 다음 순서를 표준으로 사용한다.

```text
[준비]
Watch Category 확인
→ Analysis Chart 채널 확인
→ Val/Div 확인
→ Sampling 시작

[시험]
목표 rpm 입력
→ Align
→ I/F
→ FOC 전환
→ 정상운전 데이터 확보

[안전 정지]
목표 0 rpm
→ 실제 정지 확인
→ 출력 해제 확인
→ Debug Stop

[저장]
Analysis Chart 포커스
→ File
→ Save Analysis Chart Data As
→ CSV + MTAC 저장

[검증]
CSV 시간열/샘플행/채널 확인
→ MTAC 복원 확인
→ Python 후처리 또는 Excel 검토
```

핵심은 다음 세 가지다.

1. **Watch Category는 변수 정리용이다.**
2. **Analysis Chart의 채널 체크박스는 표시할 파형 선택용이다.**
3. **모터를 0 rpm으로 안전 정지한 뒤 Debug Stop하고 Analysis Chart 데이터를 저장한다.**
