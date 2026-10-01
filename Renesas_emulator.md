# RX26T IEC60730 Checksum / Safety CRC 수정 및 검증 사례

- **작성일:** 2026-10-01
- **프로젝트:** RX26T_Fan
- **대상 MCU:** Renesas RX26T
- **대상 환경:** CS+ / E2 Lite
- **변경 배경:** 1차 수령 코드에 Tracer 기능 추가

---

## 1. 문서 목적

이 문서는 RX26T 펌웨어에 Tracer 기능을 추가한 뒤 발생한 다음 문제의 원인, 수정 방법 및 최종 검증 결과를 정리한다.

1. E2 Lite Download 실패 오류 `E1815001`
2. IEC60730 전체 ROM Checksum 불일치
3. IEC60730 Safety ROM CRC 불일치
4. Checksum 및 CRC 수정 후 POST와 Main Loop 정상 진입 확인
5. 향후 코드 변경 시 재검증 절차

---

## 2. 문제 발생 배경

1차 수령 코드에 다음과 같은 Tracer 관련 기능을 추가하였다.

```text
- Tracer Signal 추가
- Tracer Buffer 추가
- Tracer Trigger 추가
- Logging 변수 추가
- 상태 모니터링 변수 추가
```

코드 변경 후 Build는 완료되었으나, 초기에는 Download 실패가 발생했고 배선 문제 해결 후에는 프로그램이 반복적으로 Reset되는 현상이 발생하였다.

---

## 3. 최초 문제: E1815001 Download 실패

발생 메시지:

```text
Error(E1815001) : Download failed.

[Direct Error Cause]
A timeout error has occurred in emulator firmware processing.(E1815001)
```

### 3.1 확인된 원인

E2 Lite와 RX26T 사이의 디버그 신호선 결선이 꼬여 있었다.

```text
E2 Lite <-> RX26T 디버그 신호선 결선 오류
```

신호선 결선을 수정한 뒤 다음 항목이 정상화되었다.

```text
- E2 Lite 연결 정상
- RX26T Download 정상
- Debug 진입 정상
```

---

## 4. 두 번째 문제: Startup 코드로 반복 복귀

Download 후 프로그램을 실행하면 정상 운전으로 진행하지 못했다. 실행 중 Stop을 누르면 다음 Startup 코드의 `RAM_TEST()` 부근이 반복적으로 표시되었다.

```c
void PowerON_Reset_PC(void)
{
    set_intb(__sectop("C$VECT"));

    RAM_TEST();
    _INITSCT();
    set_psw(PSW_init);

    main();

    brk();
}
```

초기에는 `RAM_TEST()` 자체에서 멈춘 것으로 보였으나, 실제 원인은 IEC60730 검사 실패 후 발생한 Software Reset이었다.

```text
IEC60730 POST 실패
  -> SOFTWARE_RESET_BY_WATCHDOG() 실행
  -> RX26T Software Reset
  -> PowerON_Reset_PC() 재진입
  -> RAM_TEST() 재실행
  -> 동일 과정 반복
```

따라서 관찰된 현상은 `RAM_TEST()` 정지가 아니라 **Reset Loop**였다.

---

## 5. 전체 ROM Checksum 검사 실패

### 5.1 문제 위치

`SYSI_IEC60730_RunInitalRomCheck()`에서 계산한 ROM Checksum과 기준값을 비교한다.

```c
if (gunIEC60730_CalculatedChkSumresult_POST
        != IEC60730_ROM_BASE_CHECKSUM)
{
    if (i >= 4)
    {
        while (1)
        {
#ifdef CHECKSUM_CHECK
            nop();
            nop();
            nop();
            nop();
            nop();
#else
            SOFTWARE_RESET_BY_WATCHDOG();
#endif
            nop();
            nop();
            nop();
            nop();
        }
    }
}
```

### 5.2 실제 측정값

CS+ Watch에서 확인한 값:

```text
gunIEC60730_CalculatedChkSumresult_POST = 0x0124E840
IEC60730_ROM_BASE_CHECKSUM              = 0x0124D4BE
```

비교 결과:

```text
0x0124E840 != 0x0124D4BE
```

따라서 전체 ROM Checksum 검사가 실패하였다.

### 5.3 Software Reset 동작

Reset 매크로는 다음과 같다.

```c
#define SOFTWARE_RESET_BY_WATCHDOG() {                                          SYSTEM.PRCR.WORD = 0xA502;             SYSTEM.SWRR      = 0xA501;         }
```

전체 ROM Checksum 불일치가 5회 반복되면 위 매크로가 실행되어 MCU가 즉시 Reset된다.

---

## 6. 전체 ROM Checksum 검사 범위

전체 ROM Checksum 계산 범위:

```c
#define ROM_START_ADDR  0xFFFE0200
#define ROM_END_ADDR    0xFFFF9FFF
```

계산 방식:

```c
mpucRomAddr = (unsigned char *)ROM_START_ADDR;

gunIEC60730_CalculatedChkSumresult_POST = 0;

for (j = ROM_START_ADDR; j <= ROM_END_ADDR; j++)
{
    gunIEC60730_CalculatedChkSumresult_POST += *mpucRomAddr;

    if (mpucRomAddr == (unsigned char *)ROM_END_ADDR)
    {
        break;
    }

    mpucRomAddr++;
}
```

검사 범위 크기:

```text
0xFFFF9FFF - 0xFFFE0200 + 1
= 0x19E00 bytes
= 105,984 bytes
```

---

## 7. Tracer 기능 추가로 Checksum이 변경되는 이유

전체 ROM Checksum은 검사 범위의 Flash 데이터를 1 Byte씩 합산한다. 따라서 다음 변경으로 실행 이미지가 달라지면 Checksum도 달라질 수 있다.

```text
- Tracer Signal 추가
- Tracer Trigger 추가
- Tracer Buffer 추가
- 함수 추가 또는 수정
- 상수 및 문자열 추가
- Logging 코드 추가
- 전처리 매크로 변경
- 컴파일 최적화 변경
- 링커 섹션 및 코드 배치 변경
```

이번 사례에서는 Tracer 기능 추가 후 Flash 이미지가 변경되었지만 기존 `IEC60730_ROM_BASE_CHECKSUM` 값이 유지되어 POST 검사가 실패하였다.

---

## 8. Checksum 기준값 저장 위치 확인

`RX26T_Fan.map`에서 확인한 결과:

```text
SECTION=CCheckSumAddr
FILE=D:\Fan_Drive_Renesas_260922\CubeSuite+\DefaultBuild\SYS_IEC60730_micom_check.obj
                                  fffe0100  fffe0107         8
  _IEC60730_ROM_BASE_CHECKSUM
                                  fffe0100         4   data,g
  _IEC60730_ROM_SAFETY_BASE_CRC
                                  fffe0104         4   data,g
```

주소 배치:

```text
0xFFFE0100 ~ 0xFFFE0103 : IEC60730_ROM_BASE_CHECKSUM
0xFFFE0104 ~ 0xFFFE0107 : IEC60730_ROM_SAFETY_BASE_CRC
0xFFFE0200               : 전체 ROM Checksum 검사 시작
```

전체 ROM 검사 범위는 `0xFFFE0200 ~ 0xFFFF9FFF`이므로 두 기준값은 검사 범위 밖에 있다.

따라서 기준값을 수정해도 그 값 자체가 전체 ROM Checksum 계산에 다시 포함되는 자기참조 문제는 없다.

---

## 9. 전체 ROM Checksum 기준값 수정

기존 값:

```c
const unsigned long IEC60730_ROM_BASE_CHECKSUM = 0x0124D4BE;
```

현재 실행 이미지에서 확인된 값으로 수정:

```c
const unsigned long IEC60730_ROM_BASE_CHECKSUM = 0x0124E840;
```

수정 후 전체 ROM Checksum 검사를 통과했고 다음 함수까지 정상 진입하였다.

```c
SYSI_IEC60730_RunInitalRomCheckSafety();
```

---

## 10. Safety ROM CRC 검사 실패

### 10.1 Safety ROM 검사 범위

```c
#define SAFETY_ROM_START  0xFFFF83A0
#define SAFETY_ROM_END    0xFFFF8FFF
```

### 10.2 문제 위치

```c
if (mulSafetyDisplayCRC != IEC60730_ROM_SAFETY_BASE_CRC)
{
    if (i >= 4)
    {
        while (1)
        {
            SOFTWARE_RESET_BY_WATCHDOG();
        }
    }
}
```

### 10.3 실제 측정값

CS+ Watch에서 확인한 값:

```text
mulSafetyDisplayCRC           = 0x638D482F
IEC60730_ROM_SAFETY_BASE_CRC  = 0xCF6587A6
```

비교 결과:

```text
0x638D482F != 0xCF6587A6
```

따라서 Safety ROM CRC 검사도 실패하였다.

---

## 11. Safety ROM CRC 기준값 수정

기존 값:

```c
const unsigned long IEC60730_ROM_SAFETY_BASE_CRC = 0xCF6587A6;
```

현재 실행 이미지에서 확인된 값으로 수정:

```c
const unsigned long IEC60730_ROM_SAFETY_BASE_CRC = 0x638D482F;
```

---

## 12. 최종 적용 코드

```c
#pragma section CheckSumAddr

const unsigned long IEC60730_ROM_BASE_CHECKSUM = 0x0124E840;
const unsigned long IEC60730_ROM_SAFETY_BASE_CRC = 0x638D482F;

#pragma section
```

---

## 13. 수정 후 적용 절차

기준값 수정 후 반드시 다음 절차를 수행한다.

```text
1. 소스 저장
2. Build -> Clean Project
3. Build -> Rebuild Project
4. 새 실행 이미지 Download
5. CS+ Watch에서 두 검사 결과 재확인
6. POST 통과 확인
7. Main Loop 진입 확인
```

---

## 14. 최종 검증 결과

### 14.1 전체 ROM Checksum

```text
gunIEC60730_CalculatedChkSumresult_POST = 0x0124E840
IEC60730_ROM_BASE_CHECKSUM              = 0x0124E840
```

판정:

```text
0x0124E840 == 0x0124E840
ROM Checksum PASS
```

### 14.2 Safety ROM CRC

```text
mulSafetyDisplayCRC          = 0x638D482F
IEC60730_ROM_SAFETY_BASE_CRC = 0x638D482F
```

판정:

```text
0x638D482F == 0x638D482F
Safety ROM CRC PASS
```

### 14.3 POST 완료 지점

다음 함수까지 정상적으로 도달하였다.

```c
SYSI_IEC60730_InitOperatingCheck();
```

이에 따라 전체 ROM Checksum과 Safety ROM CRC 검사가 모두 통과했음을 확인하였다.

### 14.4 Main Loop 진입

다음 Main Loop까지 정상적으로 진입하였다.

```c
while (1)
{
    SYS_init_redefine();

    /* Main control logic */
}
```

최종 확인 결과:

```text
ROM Checksum PASS
Safety ROM CRC PASS
IEC60730 POST PASS
Main Loop 진입 PASS
Reset Loop 해제
```

---

## 15. CS+ Debug 시 해석 방법

### 15.1 F10 또는 F11 이후 Startup으로 돌아가는 경우

다음 코드가 실행되면 CPU가 즉시 Reset되므로 다음 소스 줄로 진행하지 않는다.

```c
SOFTWARE_RESET_BY_WATCHDOG();
```

따라서 F10 또는 F11 이후 `RAM_TEST()`가 표시되는 것은 에뮬레이터 고장이 아니라 Software Reset의 결과일 수 있다.

### 15.2 다음 단계 도달 여부 확인

확인하려는 다음 함수 호출 줄에 Breakpoint를 설정한 후 F5를 실행한다.

예:

```c
SYSI_IEC60730_RunInitalRomCheckSafety();
```

또는:

```c
SYSI_IEC60730_InitOperatingCheck();
```

Breakpoint에 도달하지 않고 Startup으로 돌아가면 그 이전 검사에서 Reset이 발생한 것이다.

---

## 16. 디버그용 임시 리셋 방지

전체 ROM Checksum 검사에는 다음 분기가 존재한다.

```c
#ifdef CHECKSUM_CHECK
    nop();
    nop();
    nop();
    nop();
    nop();
#else
    SOFTWARE_RESET_BY_WATCHDOG();
#endif
```

Debug 빌드에서만 `CHECKSUM_CHECK`를 정의하면 전체 ROM Checksum 실패 시 Reset 대신 `nop()` 반복 위치에서 Watch 값을 확인할 수 있다.

> **주의:** `CHECKSUM_CHECK`를 활성화한 빌드는 IEC60730 보호 동작을 우회하므로 제품용 Release에 사용하지 않는다.

Safety ROM CRC 실패 경로에는 동일한 우회 분기가 없으므로 별도로 주의해야 한다.

---

## 17. 향후 코드 변경 시 재검증 절차

Tracer 또는 애플리케이션 코드를 변경한 뒤에는 다음 순서로 검증한다.

```text
1. 기능 변경 완료
2. Clean Project
3. Rebuild Project
4. Download
5. 전체 ROM Checksum 측정
6. Safety ROM CRC 측정
7. 이전 기준값과 비교
8. 필요한 기준값 갱신
9. 다시 Clean / Rebuild / Download
10. POST 통과 확인
11. Main Loop 진입 확인
12. 운전 중 BIST 정상 여부 확인
```

기준값은 추가 코드 변경이 끝난 최종 이미지에서 산출해야 한다. 기준값을 먼저 갱신한 뒤 코드를 다시 수정하면 Checksum 또는 CRC가 다시 달라질 수 있다.

---

## 18. 코드 변경 후 체크리스트

### 18.1 빌드 및 다운로드

- [ ] 최종 코드 변경이 완료되었는가?
- [ ] Clean Project를 수행했는가?
- [ ] Rebuild Project를 수행했는가?
- [ ] 새 실행 이미지를 Download했는가?

### 18.2 전체 ROM Checksum

- [ ] `gunIEC60730_CalculatedChkSumresult_POST`를 확인했는가?
- [ ] `IEC60730_ROM_BASE_CHECKSUM`과 비교했는가?
- [ ] 동일 빌드에서 계산값이 반복해서 동일한가?
- [ ] 필요한 경우 기준값을 갱신했는가?

### 18.3 Safety ROM CRC

- [ ] `mulSafetyDisplayCRC`를 확인했는가?
- [ ] `IEC60730_ROM_SAFETY_BASE_CRC`와 비교했는가?
- [ ] 필요한 경우 기준 CRC를 갱신했는가?

### 18.4 최종 동작

- [ ] `SYSI_IEC60730_InitOperatingCheck()`까지 도달하는가?
- [ ] Main Loop에 진입하는가?
- [ ] Reset Loop가 재발하지 않는가?
- [ ] 운전 중 IEC60730 BIST가 정상 동작하는가?
- [ ] Debug 전용 `CHECKSUM_CHECK`가 Release에서 해제되었는가?

---

## 19. 최종 결론

이번 문제의 전체 흐름은 다음과 같다.

```text
Tracer 기능 추가
  -> Flash 실행 이미지 변경
  -> 전체 ROM Checksum 변경
  -> Safety ROM CRC 변경
  -> 기존 IEC60730 기준값 유지
  -> POST 검사 실패
  -> SOFTWARE_RESET_BY_WATCHDOG() 실행
  -> Startup 재진입
  -> Reset Loop 반복
```

최종 수정값:

```text
IEC60730_ROM_BASE_CHECKSUM     = 0x0124E840
IEC60730_ROM_SAFETY_BASE_CRC   = 0x638D482F
```

최종 검증 결과:

```text
전체 ROM Checksum PASS
Safety ROM CRC PASS
IEC60730 POST PASS
Main Loop 진입 PASS
Reset Loop 해제
```
