# HELIOS-BMS

태양광 친환경 충전 시스템과 배터리 안전 관리 기능을 통합한 임베디드 시스템 프로젝트. STM32 기반 멀티 보드 아키텍처와 CAN 통신으로 구성되며, CC/CV/MPPT 충전 알고리즘, 실시간 BMS 안전 제어, 자율 주행 기능을 하나의 플랫폼으로 통합한다.

---

## 개요

자동차 산업의 전동화 및 SDV(Software Defined Vehicle) 전환과 함께 배터리 안전 기술의 중요성이 높아지고 있다. 기존 고정 전원 기반 충전 시스템은 태양광과 같이 입력이 지속적으로 변동하는 환경에 효과적으로 대응하기 어렵다는 한계가 존재한다.

HELIOS-BMS는 이에 대응하여 다음 세 기능을 하나의 플랫폼으로 통합한 시스템이다.

1. **태양광 추적 및 충전효율 최적화** — 조도 센서 기반 광원 추적 및 MPPT 알고리즘으로 발전 효율 최적화
2. **배터리 안전 관리(BMS)** — 온도·가스·전류·전압 실시간 감지 및 위험 수준별 속도 제한
3. **지능형 주행** — 조이스틱 수동 주행 모드 및 초음파 센서 자율 주행 모드

---

## 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                      HELIOS-BMS 시스템                       │
│                                                             │
│   [Remote]               [Solar_master_R01]                 │
│  STM32 BlackPill  ──▶   STM32 Master Board                  │
│  조이스틱 보드    UART   차량 주행 제어 / BMS 안전 관리       │
│  (블루투스 송신)         온도·가스·전류·전압 감지             │
│                          FSM 기반 상태 제어                  │
│                          (INIT → IDLE → Force Stop)         │
│                                     │ CAN 통신              │
│                                     ▼                       │
│                          [Solar_slave_R01]                  │
│                          STM32 Slave Board                  │
│                          태양광 추적 / 충전 제어             │
│                          CC · CV · MPPT 알고리즘            │
│                          PI 제어기                          │
│                          FSM 기반 상태 제어                  │
│                          (INIT → CAN_IDLE → IDLE)           │
└─────────────────────────────────────────────────────────────┘
```

---

## 레포지토리 구조

```
HELIOS-BMS/
├── Remote/                 # STM32 BlackPill — 조이스틱 원격 제어 보드
├── Solar_master_R01/       # STM32 Master Board — 차량 주행 제어 및 BMS 안전 관리
├── Solar_slave_R01/        # STM32 Slave Board  — 태양광 추적 및 충전 제어
└── .gitignore
```

---

## 보드별 역할

### Remote — 조이스틱 원격 제어 보드
STM32 BlackPill 기반 입력 보드. 조이스틱 값을 읽어 UART 무선 통신으로 주행 명령을 전송한다.

- 조이스틱 ADC 읽기 (X/Y 축)
- 방향 명령 패킷 생성 및 UART 송신
- 수동 주행 모드 전환 명령 처리

### Solar_master_R01 — 차량 주행 제어 및 BMS 보드 (Master Board)
차량 주행 제어, 배터리 안전 관리, 통신을 FSM(Finite State Machine) 기반으로 통합 담당하는 주 제어 보드.

**FSM 상태 흐름**: `INIT → IDLE → (Force Stop)`

- **INIT**: 주변장치 초기화 수행 후 IDLE로 전이
- **IDLE**: 블루투스 조이스틱 명령 수신 및 CAN 통신으로 Slave Board에 제어 신호 전달
- **Force Stop**: 배터리 위험 상태 또는 조이스틱 정지 명령 발생 시 전환, 차량 구동 중단

주요 기능:
- **BMS 안전 관리**:
  - 온도 감지: NTC 센서, 과열 경고 및 위험 단계별 속도 제한
  - 가스 감지: MQ135, ppm 기반 SAFE / WARNING / DANGER 판정
  - 전류 감지: INA219 (I2C), 과전류 시 즉각 감속
  - 전압 감지: 배터리 전압 실시간 모니터링
- **위험 수준별 속도 제한**: 감지된 위험 요소 수에 따라 속도를 단계적으로 감소
- **주행 제어**:
  - 수동 모드: Remote 조이스틱 명령 수신 (UART/블루투스) → 모터 구동
  - 자율 모드: 초음파 센서 3방향(좌/중/우) 기반 장애물 회피
- **CAN 통신**: Slave Board로 주행 제어 신호 및 동작 모드 전달

### Solar_slave_R01 — 태양광 추적 및 충전 제어 보드 (Slave Board)
Master Board로부터 전달되는 CAN 메시지를 기반으로 태양광 추적과 충전 기능을 제어하는 보드.

**FSM 상태 흐름**: `INIT → CAN_IDLE → IDLE`

- **INIT**: 초기화 수행 후 CAN_IDLE로 전이
- **CAN_IDLE**: Master Board의 CAN 신호 대기
- **IDLE**: 수신된 조이스틱 명령에 따라 동작 모드 결정 (광원 추적 / 충전 알고리즘 실행)

주요 기능:
- **광원 추적 시스템**: 조도 센서 기반 패널 각도 최적화, 서보 모터 제어
- **충전 알고리즘**:
  - CC (Constant Current): 정전류 충전 단계
  - CV (Constant Voltage): 정전압 충전 단계
  - MPPT (Maximum Power Point Tracking): 태양광 최대 전력점 추종
- **PI 제어기**: 4차 소신호 모델 기반 설계, 비선형 동특성 보상
- **CAN 통신**: Master Board로부터 제어 신호 수신

---

## 주요 기술

### 충전 알고리즘

| 단계 | 알고리즘 | 조건 |
|---|---|---|
| 1단계 | MPPT | 태양광 입력 → 최대 전력점 추종 |
| 2단계 | CC (정전류) | 배터리 전압 < 충전 목표 전압 |
| 3단계 | CV (정전압) | 배터리 전압 ≈ 충전 목표 전압 |

MPPT는 P&O(Perturbation & Observe) 방식을 기반으로 하며, 4차 소신호 모델에서 설계된 PI 제어기가 전력 변환부를 안정적으로 제어한다.

### BMS 위험 수준 제어

```
위험 요소 감지 수   감속 단계(reduction_step)
     1개           1 step 감속
     2개           5 step 감속
     3개 이상       Stop
```

온도·가스·전류 각각이 임계값을 초과할 경우 위험 요소로 카운트되며, 누적 수에 따라 속도가 단계적으로 제한된다.

### CAN 통신 구성

Master Board와 Slave Board는 CAN Bus로 연결된다.

| 방향 | 내용 |
|---|---|
| Master → Slave | 주행 제어 신호, 동작 모드, 조이스틱 명령 |
| Slave → Master | 충전 상태, MPPT 동작 여부, 태양광 전력 데이터 |

---

## 하드웨어 구성

| 구성 요소 | 모델 | 용도 |
|---|---|---|
| MCU (Master 1/2) | STM32F411RETx | 충전 제어 / BMS 및 주행 |
| MCU (Remote) | STM32 BlackPill | 조이스틱 입력 송신 |
| 조이스틱 | 아날로그 2축 | 수동 주행 방향 입력 |
| 조도 센서 | LDR / 포토다이오드 | 광원 방향 추적 |
| 온도 센서 | NTC 서미스터 | 배터리 온도 감지 |
| 가스 센서 | MQ135 | 가스 누출 감지 |
| 전류·전압 센서 | INA219 (I2C) | 배터리 전류·전압 측정 |
| 초음파 센서 | HC-SR04 × 3 | 좌·중·우 장애물 감지 |
| 모터 드라이버 | L298N | DC 모터 PWM 제어 |
| 태양전지 패널 | - | 태양광 에너지 수집 |
| DC-DC 컨버터 | Buck/Boost | 전력 변환 |

---

## 개발 환경

- **IDE**: STM32CubeIDE
- **언어**: C (STM32 HAL 기반)
- **통신**: CAN Bus (Master ↔ Slave), UART/블루투스 (Remote → Master)
- **MCU**: STM32F411RETx (Master 1/2), STM32 BlackPill (Remote)
