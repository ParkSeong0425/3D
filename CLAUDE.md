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
- 상태(status)는 W 대기 / R 운전 / I 원점 / P 일시정지 / A 알람 / N 초기상태(`INITIAL`)만. 만재·카드는 상태가 아니라 `full_1/2`, `card_ok_1/2` 변수로만 본다.
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

### 남은 일 (집에서)
1. 오늘 코드 읽고, 원하는 방식으로 직접 다시 쓰기 → Claude 리뷰 (작업 방식 7~8)
2. **알람 상황 정하기** (데이터시트): 드라이브 알람 X/Y/TILT, 통신 끊김 몇 번이면 알람, 이동 30초 초과 → 위치 이탈(5/6/7)?, 원점 실패(10)? / 알람 후 램프·PC 이벤트·복구 방법
3. **X 원점 살짝 닿기**: 원점 5~10번 반복해서 중간에 멈추는지(오인식) 확인 → 속도 먼저, 토크는 5씩, TIME 20ms 적당한지
4. 긴급정지가 드라이브 / HMI 전원을 끊는지 확인 (끊으면 통신 시간 초과로 몇 초씩 버스가 막힘)
5. 확인: HMI 센서 칸 위/아래 순서, PC 단 번호 1 = 만재 센서 1단(FULL_x_1)인지
6. 덧붙인 것 정리 후보: `manual`, `done_text`, `home_text`, `tcp_mode/tcp_end`, `floor_full`, `Status_INIT`, 램프 R/Y 교체(배선 고치면)

## 현재 상태 (2026-10-02 회사 버전 기준)

- 완료: UART2 DMA 통신, 시간 제한, 이동 중 명령 거부, UART4_Xfer, event.c(긴급정지, 램프, 만재 EXTI, HMI), S 일시정지, CLI INIT, **긴급정지 중 즉시 중단(estop 확인)**, HMI 상태 표시(ESP32 프로토콜), TCP 이벤트 · 완료응답 · LL 램프, RFID CAN(0.5초 질문 / 2초 판정).
- 9/28 리뷰 할 일 중 해결: 긴급정지 후 Motor_Task 중단, HMI 버튼 번호(한 바이트라 문제 없었음). 남음: UART4 플래그 비교 `& XFER_DONE`, 버퍼 길이 검사.
- 터치패드: JC3248W535C (ESP32-S3, 3.5" 320x480). 소스는 하이웍스 HMI.zip (HMI_JC3248W535_v1.2, `serial1.cpp` HMI_Parser 가 프로토콜). Sensor[0] = 화면 왼쪽.
- 아직: 알람(ALARM_* 번호 / Alarm_Set), UART1 CLI 폴링, W6100 1ms 폴링.
- 노션 정리: 코텍전자 → [10/02](https://app.notion.com/p/3ed10d3ee54481fb9d0fc4639a4e600b).

## 일정 (마감 2026-10-14, 10/5·10/6(예비군)·10/9 휴무, 매일 19시까지)

- 9/28 HMI / 9/29 Event + HMI 버튼 / 9/30 일시정지 S / 10/1 이벤트 → TCP 프로토콜 + ACK
- 10/2 RFID (CAN) / 10/7 효율화 / 10/8 센서 EXTI / 10/12 통합 테스트 / 10/13 유지보수 정리 / 10/14 여유

## 참고

- GitHub: https://github.com/ParkSeong0425/ThreeD_roder
- 노션: 코텍전자 → [09/27](https://app.notion.com/p/3e710d3ee544816aadc3d257fcc232da), [09/27 (2) 개념 정리 · 이해도 테스트](https://app.notion.com/p/3e810d3ee544810186fbec938c853ed1)
- CLI: Tera Term, STLINK-V3 VCP (집 PC COM6), 115200. Windows 11 25H2는 wmic가 없어 CubeIDE 시리얼 콘솔이 안 됨.
- 명령: CLI `X/Y pulse percent`, `C percent`, `R/L deg percent`, `S`, `I`, `INIT`, `IP`, `FRAM`. TCP `STX 01MI_... ETX` 형식.
- 문법 검사: `arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard -std=gnu11 -DUSE_HAL_DRIVER -DSTM32G473xx -fsyntax-only -Wall` + 모든 헤더 폴더 `-I`.
