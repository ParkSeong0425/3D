# threeD_roder (3D 로더) 펌웨어 — Claude 작업 안내서

STM32G473VCT6 (170MHz) + FreeRTOS(CMSIS-RTOS2) + STM32CubeIDE 1.19 프로젝트.
3축 로봇(X / Y / TILT)을 RS485 Modbus RTU 드라이브로 제어하고, PC와 TCP(W6100)로 통신한다.
사용자는 펌웨어를 배우는 중이다. 설명은 **한국어, 쉬운 비유, 그림(다이어그램) 포함**으로 한다.

## 작업 방식 (반드시 지킬 것)

1. **바꾸기 전에 계획부터**: "어느 파일 → 어느 함수 → 무엇을" 표 + 개념 + 왜 그렇게 하는지 + 그림.
2. 사용자가 **"그렇게 하자"** 라고 하면 그 단계 전체를 한 번에 적용한다.
3. 적용 후 **파일별로 바뀐 줄**을 보여준다. 가능하면 ARM GCC로 문법 검사.
4. 하루 작업이 끝나면 **노션 "코텍전자" 페이지 맨 아래**에 날짜 페이지로 정리한다 (질문·해결·개념·변경·다음 할 일).
5. **코드 구조는 사용자가 먼저 짠다.** Claude는 추천과 설명, 그리고 수정.
6. GitHub `main`에는 사용자 허락 없이 올리지 않는다. 리뷰·제안은 별도 브랜치나 REVIEW 문서로.
7. **사용자가 직접 다시 짠 코드를 리뷰하는 방식**(10/2부터): 사용자가 짠 코드를 읽고 → 맞는지 / 왜 안 되는지 설명 → 고칠 때도 **사용자 스타일 그대로**. 문제가 생길 때마다 변수·플래그·함수를 덧붙이지 말고, 사용자가 만든 함수 안에서 해결한다.
8. 판단 기준은 **CPU 효율과 메모리(RAM, 스택, 버스 점유) 효율**. 리뷰할 때 "이 코드가 CPU를 얼마나 쓰는지, 버스를 얼마나 잡는지, 메모리를 얼마나 쓰는지"를 같이 설명한다.
9. **하루 작업이 끝나면 Claude가 이 CLAUDE.md를 업데이트한다**: "현재 상태", "그날 바뀐 것", "남은 일"을 그날 기준으로 고치고, 바뀐 부분을 사용자에게 보여준다. (집 ↔ 회사는 zip으로 옮기므로, 이 파일이 다음 세션의 유일한 기억이다.) 사용자가 "끝", "마무리", "오늘은 여기까지" 같은 말을 하면 반드시 먼저 한다.

## 코드 스타일 (사용자 스타일 유지)

- `run.c`의 `I()` 수준으로 **짧고 쉽게**, 단계마다 한글 한 줄 주석.
- `if (!Func())` 금지 → `if (Func() == 0)` 처럼 쓴다.
- 성공 1 / 실패 0 반환, 실패는 `return 0`으로 위로 전달.
- **새 파일 · 새 함수 · 새 상수는 꼭 필요할 때만**, 반드시 계획에서 승인받고. 구조를 크게 바꾸지 않는다 (예전에 bus.c 분리했다가 되돌림).
- **모든 while 기다림에는 시간 제한** (`osKernelGetTickCount()` 시작 시각 비교). 예외 : 일시정지(PAUSE) 대기.
- `save.c`, `cmd.c`의 풀어서 쓴 구조(left_x1 ~ left_x9 등)는 **일부러 그런 것** → 배열로 바꾸자고 하지 않는다.
- CubeMX 생성 파일은 **USER CODE BEGIN ~ END 안에만** 쓴다. 설정은 ioc에서.
- 파일은 CRLF 줄바꿈. sed로 수정하면 LF로 바뀌니 주의.
- 디버그 출력은 원인 찾으면 지운다.

## 구조

```
태스크 층   app_freertos.c : Event_Task(High) / Motor_Task(AboveNormal) / Comm_Task(Normal)
명령 층     cmd.c (TCP 명령 해석) · cli.c (UART1 CLI) · hmi.c (UART4 터치패드 버튼)
이벤트      event.c : 100ms 순회 (긴급정지, HMI, RFID, 만재, 램프, RUN 확인). 긴급정지 EXTI(PD11)는 플래그로 즉시 깨움
동작 층     run.c : MI / MO / PO / I(원점) / S(일시정지 = status PAUSE) / Motor_Busy / Motor_Run
장치 층     i_motor.c (X, ID3) · a_motor.c (Y, ID1) · t_motor.c (TILT, ID2) · fram.c · net.c · save.c · rfid.c · lamp.c
통신 부품   usart.c USER CODE : UART2_Xfer (모터, DMA + 인터럽트 + 뮤텍스) · UART4_Xfer (HMI, 인터럽트)
```

- 명령 경로: W6100 → `net.c NET_Run`(STX~ETX로 끊기) → `cmd.c CMD_Run`(해석, Motor_Busy면 NAK `can_not_command`) → `MotorQueue` → `run.c Motor_Run` → `motor_basic_run` → 모터 파일 → `bus_xfer` → `UART2_Xfer`(DMA, 응답 끝 IDLE 인터럽트가 태스크 깨움).
- RTOS 도구는 4가지만: 시간(`osDelay`, `osKernelGetTickCount`) / 큐 / 플래그(인터럽트→태스크, usart.c·event.c 안에만) / 뮤텍스(usart.c 안에만).
- 인터럽트 우선순위: FreeRTOS 함수를 부르면 5 이상 ("Uses FreeRTOS functions"). DMA 5, EXTI15_10 5, FDCAN 5, USART2 6, USART1 7, UART4 7.
- 시계: FreeRTOS tick(SysTick, 1ms) / HAL tick(**TIM6 — 지우면 안 됨**) / DWT(µs 측정 전용, `dwt.h`, `uart2_us`).
- 대기 규칙: 이동 30초, 원점 60초 제한. `wait_xy`/`wait_tilt`는 osDelay 없이 반복 읽기 (통신 중 잠들어 CPU 낭비 없음).
- 폴링이 맞는 것: FRAM, EEPROM (짧고 드묾).

## 10/2 회사에서 바뀐 것 (Claude 가 수정, 사용자가 집에서 다시 읽고 정리할 예정)

- Y 배율 원인: Y 드라이브 H05_09 = 320 (1회전 320펄스) → `Y_PULSE 320`, `Y_GEAR 5`, `Y_PI 160`. X 는 `X_PI 100`. 원점 후 이동 Y `AM_Move(150, 50)`, X `IM_Move(100, 30)` (사용자 값).
- 상태(status)는 W 대기 / R 운전 / I 원점 / P 일시정지 / A 알람 / N 초기상태(10/7 부터 `RESTART`)만. 만재·카드는 상태가 아니라 `full_1/2`, `card_ok_1/2` 변수로만 본다.
- **렉1 = 오른쪽 렉, 렉2 = 왼쪽 렉** (cmd.c PO 에서 rack 1 → right_x/right_y). 만재 핀 FULL_1_x · 경광등 _1 핀 · FDCAN1 = 렉1 (핀 이름 그대로 맞음). HMI 화면은 왼쪽 칸 = Sensor[0] 이라 `hmi.c` 에서 왼쪽 칸에 렉2, 오른쪽 칸에 렉1 을 넣는다. 같은 렉 안 R/Y 만 배선이 반대라 `lamp_rfid`/`lamp_manjae` 에서 바꿔 씀.
- 긴급정지: 인터럽트는 깨우기만, `estop_event` 가 핀을 직접 읽음. 모든 대기(`wait_xy`, `wait_tilt`, 원점, CLI 이동)에 `if (estop == 1) return 0;`. 흐름: 긴급정지 → 알람 → 리셋(처음 상태 INITIAL) → 시작(원점).
- HMI: ESP32 프로토콜(`STX '1' 'C' 작업 일시정지 알람 센서20 ETX`)에 맞춰 `hmi.c` 가 상태를 채워 보냄. 수동 버튼은 설정→수동(33)으로 수동 모드일 때만.
- PO: 틸트 전·복귀 전에 **그 단** 만재(`floor_full(rack, y_no)`) / **그 렉** 카드 없음이면 `PPAUSE:F` / `PPAUSE:S` 한 번 보내고 대기, 풀리면 자동 진행. 같은 렉이라도 다른 단은 진행. MI/MO 는 그대로 이동. `PO(rack, x, y, y_no, …)` 로 단 번호를 받음.
- TCP: 완료응답 AI/AO/EO 를 PC 승인(`번호-AI_…`)까지 1초마다, 승인 전 MI/MO/PO 는 NAK. 원점 완료 `번호<ACK>homing done` 한 번. 램프 명령 `LL_번호_색_모드_시간`(번호 홀수 렉1 / 짝수 렉2, 색 1 적 2 녹 3 황). PC 연결 중 램프는 PC 가, 안 연결이면 STM 이.
- RFID: 0.5초마다 리더에 'C' 질문, 2초 동안 못 읽으면 카드 없음. 리더 ID = 200 + DIP (두 렉 다 201, CAN 선이 따로라 같아도 됨).
- 램프: 타이머 안 씀. `blink()` 가 시간으로 켬/끔 → 렉1·2 같은 박자. 긴급정지·알람 빠르게, 원점·일시정지 느리게.
- 넷: 마지막 명령 후 20초 조용하면 끊김 (전에는 접속 후 20초에 무조건 끊김).
- 가감속(사용자 값): X `IM_ACC/DEC_MS 900`, Y `AM_ACC_DEC_MS 100`, TILT `TM_ACC/DEC_MS 200`.
- X 토크 원점(사용자 값): `IM_HOME_HI_RPM 180`, `IM_HOME_LO_RPM 80`, `IM_HOME_ACC/DEC_MS 1000`, `IM_HOME_TIME_MS 20`, `IM_HOME_FORCE 35`. 목표 = "쾅 부딪히지 않고 살짝 닿으면 원점". 원칙: **토크(FORCE)만 낮추면 마찰·가속 토크에 오인식**(X 는 모터 2대라 한쪽만 먼저 잡히면 비틀림) → 먼저 **속도를 낮추고**, 판단 시간(TIME)은 너무 짧으면 순간 튀는 토크에 오인식. 지금 TIME 20ms 는 짧은 편이라 테스트로 확인 필요.

## 10/7 회사에서 바뀐 것 (event · hmi · run · status 정리, 일시정지 T/F/S/M)

- **안 쓰는 함수 삭제**: `AM_Speed/AM_Torque/AM_Read`, `IM_AlarmRead/IM_DIPRead/IM_Read`, `TM_Read`, `Home_lamp/Alarm_lamp`, `NET_Link`, run.c `Motor_Check`·`CHECK_COUNT`, event.h `Alarm_Set/Clear` 선언(본문은 event.c 에 주석으로 남김). 헤더도 같이 지움.
- **상태 이름**: 초기상태 = `RESTART`(사용자가 INITIAL → RESTART 로 바꿈, CRC 레지스터 이름과 겹쳐서). 전원 켜지면 `status_1/2 = RESTART`. `Status_INIT()` 이 RESTART 로 만듦.
- **S() 를 status.c 로**: S 는 "멈춰라 표시"만 (`Status_PAUSE` 두 렉 + `pause_why='M'` + PC 알림). 모터를 바로 세우지 않음. MI/MO 는 도착 후, PO 는 쏟기 전·복귀 전에 멈춤.
- **hmi.c**: 보낼 27바이트 틀 `tx` 를 static 으로 미리 채우고 바뀌는 11칸만 씀. 버튼 = `rx[2] - '0'`. `hmi_pause` 가 `pause_why` 대로 1 시딩월 / 2 만재 / 3 틸트 / 4 사용자.
- **event.c hmi_event**: 수동 버튼(21~26)은 `manual == 1` 일 때만. RUN = 초기상태면 `Status_HOME`(→ Motor_Task 가 원점), 일시정지 M·T 면 풀기. STOP → `S()`. RESET → 알람 리셋 + 초기상태. 설정→수동(33) 토글.
- **긴급정지 EXTI 콜백**: 주석 처리돼 있어서 긴급정지가 이동 끝난 뒤에야 먹던 것 → 사용자가 주석 풂 (테스트 필요).
- **manjae.c**: `floor_full(rack, dan)` 공개(manjae.h). `Full_Read` 는 `GPIOE->IDR & 0x7F80` 으로 만재 8칸 한 번에 읽고 바뀔 때만 나눔.
- **PO 일시정지 (T/F/S/M)**: run.c PO = 이동 → `PO_event(rack, y_no, 1)` → Y 올리며 틸트 → `touch = PO_pour(…)` → `PO_event(rack, y_no, touch)` → 복귀. `PO_pour`/`PO_event` 는 event.c 에 있지만 **Motor_Task 가 실행**.

  | 이유 | 언제 | 풀기 | HMI |
  |---|---|---|---|
  | T 틸트 | 쏟는 동안 그 단 센서에 한 번도 안 닿음 | HMI RUN | 3 |
  | F 만재 | 그 단 만재 | 센서 풀리면 자동 / PC `01D` | 2 |
  | S 시딩월 | 그 렉 카드 없음 | 카드 들어오면 자동 | 1 |
  | M 사용자 | STOP 버튼 / S 명령 | HMI RUN | 4 |
- **cmd.c**: `CMD_D ('D' << 8)` 추가 — 만재(F)로 멈춘 것만 억지로 풀기.
- 이름 정리: `full_new/pause_new` → `full_event_on/pause_event_on`.
- 미룸: TM `GO_ABS 0x03 → 0x07`(움직이는 중 새 목표 받기, 틸트 즉시 멈춤·재개용) — 사용자가 직접 고쳐 볼 예정. TCP W6100 인터럽트 · CAN 은 다음 주.

### 10/7 결론 : 구조가 마음에 안 듦 → 집에서 "왜 이렇게 짰나" 공부부터

사용자 생각: **10/2 코드는 Claude 가 거의 다 짠 것**이라 내 코드가 아니다. 앞으로는 **내가 구조를 짜고 싶고, 어떻게 짜야 하는지 알면서** 하고 싶다.
특히 고쳐야 할 것 같은 곳: **status.c, run.c (Motor_Run), cmd.c**.

- Motor_Run 이 지저분한 이유(10/7 에 본 것): 켜질 때 원점 / HMI RUN 원점 / 큐 명령 3가지 일을 한 함수에서 함. 원점 길이 3개(켜질 때·HMI = `motor_basic_init`, PC·CLI 의 I = `I()` 만)로 다름. 시작·끝 상태를 `cmd.command == CMD_…` 로 명령마다 따로 정함 (MI→W, MO→R 유지, PO→W).
- Claude 가 낸 안 2개는 **사용자가 거절**: ① `home_run`/`cmd_run` 새 함수 ("I 함수가 있는데 왜 또 함수를"), ② `cmd.command ==` 줄을 옮기는 안 ("커멘드들이 보기 싫다"). → 다음엔 사용자가 구조를 먼저 짜고 Claude 는 설명·리뷰만.
- 상태(status)를 바꾸는 곳이 흩어져 있음: run.c(Motor_Run), event.c(hmi_event, PO_event, estop_event), status.c(S), cmd.c(D). status.c 를 어떻게 할지 사용자가 정할 것.

**집에서 할 수업 (다음 세션 첫 일)**: 파일마다, **함수마다** 아래를 자세히 설명.
1. 이 함수가 하는 일 (한 줄) + 누가 부르나 / 어느 태스크가 실행하나
2. **왜 이렇게 짰나** (다른 방법과 비교, 장단점)
3. 그림 (호출 흐름, 시간 흐름, 메모리 · 버스)
4. 나오는 **C 개념** (static, volatile, extern, 구조체 값 전달, 포인터, `'0'` 빼기, 비트 마스크 `& 0x7F80`, `<< 8` 등)
5. 나오는 **펌웨어 개념** (태스크 · 우선순위, 큐, 스레드 플래그, 뮤텍스, DMA, IDLE 인터럽트, EXTI, Modbus RTU, 폴링 vs 인터럽트, 시간 제한)
6. CPU / 버스 / RAM 을 얼마나 쓰나

순서 추천: status → run → event → cmd → hmi → net → 모터(i/a/t_motor) → manjae · rfid · lamp → usart.c USER CODE (UART2_Xfer, UART4_Xfer).

### 남은 일
0. **빌드 전 꼭**: `app_freertos.c` Start_Motor_Task 168줄 `PO_event(rack, dan, touch);` 삭제 (모르는 변수라 빌드 에러, `Motor_Run()` 은 안 끝나서 실행도 안 됨). Motor_Task 는 `Motor_Run();` 하나면 됨.
1. 위 수업 → 사용자가 status / run / cmd 구조 직접 설계 → Claude 리뷰 (작업 방식 7~8)
2. 정해야 할 것: CMD_S `S_1` 만 받음(렉 번호인데 S 는 두 렉 다 멈춤 — `S_1`/`S_2` 다 받기?), hmi.h 버튼 이름을 ESP 와 맞추기(ESP 12 = 일시정지인데 이름 `BTN_STOP`), 수동 버튼이 `AM_Pos` 등 실패를 안 봄(실패면 now = 0 → 0 근처로 이동 위험), 램프 규칙을 lamp.c 로, `Status_RUN` 이 PAUSE 를 덮는 경우, MO 도착 후 R 유지 이유
3. 테스트: 긴급정지 즉시 멈춤(EXTI 주석 푼 뒤), T 오인식(쏟는 시간 `return_delay*10` 이 짧으면 정상인데 T), F 풀린 뒤 카드 다시 안 봄, PC `01D`
4. GO_ABS 0x07 (틸트 즉시 멈춤 · 재개)
5. **알람 상황 정하기** (데이터시트): 드라이브 알람 X/Y/TILT, 통신 끊김 몇 번이면 알람, 이동 30초 초과 → 위치 이탈(5/6/7)?, 원점 실패(10)? / 알람 후 램프·PC 이벤트·복구 방법
6. **X 원점 살짝 닿기**: 원점 5~10번 반복해서 중간에 멈추는지(오인식) 확인 → 속도 먼저, 토크는 5씩, TIME 20ms 적당한지
7. 긴급정지가 드라이브 / HMI 전원을 끊는지 확인 (끊으면 통신 시간 초과로 몇 초씩 버스가 막힘)
8. 확인: HMI 센서 칸 위/아래 순서, PC 단 번호 1 = 만재 센서 1단(FULL_x_1)인지
9. 다음 주: TCP W6100 인터럽트, CAN

## 현재 상태 (2026-10-07 회사 버전 기준, 사용자가 git 에 올릴 예정)

- 완료: UART2 DMA 통신, 시간 제한, 이동 중 명령 거부, UART4_Xfer, event.c(긴급정지, 램프, 만재 EXTI, HMI), S 일시정지(표시만), CLI INIT, 긴급정지 중 즉시 중단(estop 확인), HMI 상태 표시(ESP32 프로토콜, 일시정지 이유 포함), TCP 이벤트 · 완료응답 · LL 램프 · D, RFID CAN(0.5초 질문 / 2초 판정), PO 일시정지 T/F/S/M, 안 쓰는 함수 삭제.
- 문법 검사 통과 (run / event / cmd / hmi / manjae / status). 보드 테스트는 아직. app_freertos.c 168줄은 위 남은 일 0번.
- 9/28 리뷰 남음: UART4 플래그 비교 `& XFER_DONE`, 버퍼 길이 검사.
- 터치패드: JC3248W535C (ESP32-S3, 3.5" 320x480). 소스는 하이웍스 HMI.zip (HMI_JC3248W535_v1.2, `serial1.cpp` HMI_Parser 가 프로토콜). Sensor[0] = 화면 왼쪽. ESP 작업: 0 초기 1 대기 2 운전 3 일시정지 4 정지 5 원점 6 알람 / 일시정지: 0 없음 1 시딩월 2 만재 3 틸트 4 사용자.
- 아직: 알람(ALARM_* 번호 / Alarm_Set), UART1 CLI 폴링, W6100 1ms 폴링.
- 노션 정리: 코텍전자 → [10/02](https://app.notion.com/p/3ed10d3ee54481fb9d0fc4639a4e600b). 10/7 은 아직.

## 일정 (마감 2026-10-14, 10/5·10/6(예비군)·10/9 휴무, 매일 19시까지)

- 9/28 HMI / 9/29 Event + HMI 버튼 / 9/30 일시정지 S / 10/1 이벤트 → TCP 프로토콜 + ACK
- 10/2 RFID (CAN) / 10/7 효율화 / 10/8 센서 EXTI / 10/12 통합 테스트 / 10/13 유지보수 정리 / 10/14 여유

## 참고

- GitHub: https://github.com/ParkSeong0425/ThreeD_roder
- 노션: 코텍전자 → [09/27](https://app.notion.com/p/3e710d3ee544816aadc3d257fcc232da), [09/27 (2) 개념 정리 · 이해도 테스트](https://app.notion.com/p/3e810d3ee544810186fbec938c853ed1)
- CLI: Tera Term, STLINK-V3 VCP (집 PC COM6), 115200. Windows 11 25H2는 wmic가 없어 CubeIDE 시리얼 콘솔이 안 됨.
- 명령: CLI `X/Y pulse percent`, `C percent`, `R/L deg percent`, `S`, `I`, `INIT`, `IP`, `FRAM`. TCP `STX 01MI_... ETX` 형식.
- 문법 검사: `arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard -std=gnu11 -DUSE_HAL_DRIVER -DSTM32G473xx -fsyntax-only -Wall` + 모든 헤더 폴더 `-I`.
