# 1. 주제

**스마트 세이프티 시스템 (Smart Safety System)**
건설 현장 작업자 IoT 안전 모니터링 플랫폼
STM32 / Raspberry Pi 5 / Qt Dashboard / Bluetooth SPP / TCP / ML

---

## 2. 주제 선정 관련 기사

## 기사 1

**2024년 산업재해 사망자 589명… 건설업 46.9% 최다**
출처 : 세계일보 / 고용노동부 보도자료 (2025.03.12)

- 2024년 산업현장 사고사망자 589명 (553건)
- 건설업 276명 → 전체의 46.9% 차지
- 떨어짐 사고가 전체 사망의 38.5% → 안전모 미착용이 주요 원인

## 기사 2

**안전모 미착용 시 30cm 추락에도 사망 — 개인보호구 착용이 핵심**
출처 : 안전보건공단 / 세이프티퍼스트뉴스 (2024.02.07)

- 30~40cm 낮은 높이 추락에도 안전모 미착용으로 사망 사례 빈번
- AB형·ABE형 안전모는 추락 위험 방지 용도로 국가 공인 기준에 명시
- 착용 기준이 있어도 실제 미착용 상태 작업 다수

## 기사 3

**건설현장 사고 원인 1위는 관리적 요인 — 안전 규정 미준수 32%**
출처 : SK AX 분석 리포트 / 한국건설산업연구원 (2025.09)

- 사고 원인 1위 : 관리적 요인 32% (안전규정 미준수 / 부적절한 지시 / 관리 부재)
- 2024년 상반기 부상 근로자 15,959명 / 경제적 손실 6조 원
- 인력 감시·수기 보고 방식은 한계 → 스마트 안전관리 솔루션 도입 필수

---

# 2. 목표

현장에서 안전모 미착용·근무 중 음주·과중 작업 지시 등이 실제로 빈번하게 발생한다. 이를 방지하기 위해 스마트 세이프티 시스템을 개발했다.

| 감지 항목 | 방법 |
| --- | --- |
| 음주 여부 | MQ-3 알코올 센서 → gas_do 플래그 → Qt 경보 |
| 심박수 이상 | HW-827 맥박 → ADC → BPM 피크감지 알고리즘 |
| 헬멧 착용 여부 | IR Tracker → 착용 시각 기록 → 근로시간 자동 누적 |
| 행동 / 낙상 | MPU-6050 6축 IMU → ML 행동인식 + 낙상 감지 |
| 영상 증거 | GStreamer → 사고 전후 10초 MP4 자동 저장 |

기업 ↔ 작업자 양방향 대응력 강화 : Qt 대시보드로 관리자가 실시간 모니터링, STM32 LCD로 작업자에게 상태 역전송

---

# 3. 시스템 구성도

![image.png](attachment:cd90fd84-d150-4639-920e-4cd1026b0104:image.png)

```markdown
[STM32 BAND]
  MPU-6050  가속도+자이로 6축
  BMP-280   기압 (층간 이동 감지)
  MQ-3      알코올
  HW-827    맥박 ADC
  LCD 1602  BPM / 착용 / 행동 표시
        |
        | Bluetooth SPP (HC-06 / rfcomm0 / 9600bps)
        |
[Helmet RPi5]  ← 클라이언트
  BH-1750   조도
  IR Tracker  헬멧 착용 감지
  웹캠      GStreamer JPEG 스트림
  ML 모델   imu_motion_model2.pkl
  SQLite DB helmet_sensor_data.db
        |
        | WiFi TCP  9090 (센서) / 9091 (영상)
        |
[Server RPi5]  ← 서버
  C++17 멀티스레드 서버
  DeviceState  헬멧별 독립 상태 관리
  GStreamer    MP4 저장
  JSONL 로그  헬멧별 파일 분리
        |
        | WiFi TCP  9092 (영상) / 9093 (센서) / 9094 (SWITCH 명령)
        |
[Qt Dashboard]  ← 관리자 UI
  실시간 센서 수신
  영상 스트리밍 표시
  헬멧 전환 (SWITCH:helmet-02)
  사고 / 낙상 / 음주 알림 카드
```

---

# 4. 기술 정리

## 하드웨어

| 부품 | 역할 |
| --- | --- |
| STM32 | 센서 허브 (MPU-6050, BMP-280, MQ-3, HW-827, LCD) |
| MPU-6050 | 가속도+자이로 6축 → ML 행동인식 / accel_mag / accel_delta 계산 |
| BMP-280 | 기압 → 층간 이동 감지 보조 |
| MQ-3 | 알코올 디지털 출력 (gas_do = 0 / 1) |
| HW-827 | 맥박 ADC 원시값 → BPM 피크감지 변환 |
| IR Tracker | 헬멧 착용 유무 판단 → 근로시간 누적 |
| BH-1750 | 조도(lux) 측정 → 환경 분류 (야간 / 실내 / 야외) |
| 웹캠 | GStreamer 영상 스트리밍 → 사고 영상 저장 |
| LCD 1602 | BPM / 착용상태 / 행동 역전송 표시 |
| Raspberry Pi 5 | 클라이언트(헬멧) 1대 + 서버 1대 |

## 소프트웨어

| 항목 | 내용 |
| --- | --- |
| 클라이언트 | Python 3 (test_helmet22.py) |
| 서버 | C++17 (server.cpp) + nlohmann/json + GStreamer |
| 데이터베이스 | SQLite3 (WAL 모드 / 10패킷 배치 commit) |
| ML 모델 | scikit-learn Random Forest (joblib) — 정확도 97.5% |
| 영상 스트리밍 | GStreamer + libcamerasrc → JPEG → TCP |
| ML 학습 방식 | UCI WISDM 공개 데이터 선학습 → 실측 데이터 재학습 |
| Qt Dashboard | 실시간 센서 / 영상 / 경보 표시 |

## 통신

| 구간 | 방식 | 세부 |
| --- | --- | --- |
| STM32 ↔ RPi5 | Bluetooth SPP | HC-06 / rfcomm0 / 9600bps |
| Helmet RPi5 → Server | WiFi TCP | 9090 센서 JSON / 9091 영상 |
| Server → Qt | WiFi TCP | 9092 영상 / 9093 센서 / 9094 SWITCH 명령 |

---

# 5. 주요 기능

## TCP 통신 구조

- 포트 9090 : 헬멧 → 서버, JSON 센서 데이터 1줄씩 전송 (\n 구분)
- 포트 9091 : 헬멧 → 서버, 8byte 길이 헤더 + JPEG 바이너리
- 포트 9092 : 서버 → Qt, 영상 브로드캐스트
- 포트 9093 : 서버 → Qt, 센서 JSON 브로드캐스트
- 포트 9094 : Qt → 서버, SWITCH:helmet-02\n 명령 (영상 전환)
- 연결 직후 DEVID:helmet-01\n 전송 → 다중 헬멧 device_id 매핑

## SPP (Bluetooth Serial Port Profile)

- STM32가 HC-06을 통해 UART로 센서 패킷 전송
- 패킷 포맷 : [DATA] ax,ay,az,gx,gy,gz,pressure,adc,gas_do
- RPi5에서 /dev/rfcomm0 로 수신 → parse_line() 정규식 파싱
- 역방향 : RPi5 → STM32 LCD로 [BPM] / [HLM] / [ACT] 문자열 전송

## 머신러닝 (행동인식)

- 모델 : Random Forest Classifier (scikit-learn)
- 입력 : IMU 6축 데이터 (ax, ay, az, gx, gy, gz) × 50샘플 윈도우
- 피처 : 각 축별 평균 / 표준편차 / 최대 / 최소 / 범위 + 가속도 합성크기 통계 (총 33개)
- 분류 결과 : 걷기 / 뛰기 / 앉음_정지 (확률 < 0.6 → 알수없음)
- 추론 구조 : ml_worker 별도 스레드 + Queue(maxsize=1) → 메인루프 논블로킹
- 낙상 감지 : 기압 감소 0.2hPa 이상 + 가속도 급변 5.0m/s² 이상 동시 발생

## 데이터 저장 처리

SQLite3 배치 commit + WAL 모드로 성능 최적화. 아래 섹션 참고.

---

# 6. 데이터 저장 — 상세

## DB 구조 (helmet_sensor_data.db)

```markdown
CREATE TABLE sensor_log (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp      DATETIME DEFAULT (datetime('now','localtime')),

    -- IMU (STM32 → BT → RPi)
    ax REAL, ay REAL, az REAL,    -- 가속도 m/s²  (원시값 / 10000)
    gx REAL, gy REAL, gz REAL,    -- 자이로 rad/s (원시값 / 10000)

    -- 환경
    pressure       REAL,          -- 기압 hPa (원시값 / 100 or / 10000 자동 판별)
    lux            REAL,          -- 조도 (BH-1750, 0.2초 캐시)

    -- 착용 / 근로
    tracker_state  INTEGER,       -- IR센서 원시값 (0/1)
    helmet_wearing INTEGER,       -- 착용 여부 (0/1)
    work_seconds   INTEGER,       -- 누적 착용 근로시간 (초)

    -- 맥박
    adc_avg        INTEGER,       -- HW-827 ADC 원시값
    bpm            INTEGER,       -- 피크감지 계산 결과

    -- 가속도 파생값
    accel_mag      REAL,          -- 합성크기 sqrt(ax²+ay²+az²)
    accel_delta    REAL,          -- 이전 프레임 대비 변화량

    -- 알코올
    gas_do         INTEGER,       -- MQ-3 출력 (0=정상 / 1=감지)

    -- ML 결과
    activity_ml    TEXT,          -- 걷기 / 뛰기 / 앉음_정지 / 알수없음

    -- 사고
    fall_detected  INTEGER        -- 낙상 감지 (0/1)
);
```

## 성능 최적화

| 항목 | 내용 |
| --- | --- |
| WAL 모드 | PRAGMA journal_mode=WAL → 읽기/쓰기 동시 접근 허용 |
| 동기화 수준 | PRAGMA synchronous=NORMAL → fsync 부하 감소 |
| 배치 commit | COMMIT_EVERY=10 → 10패킷마다 1회 commit |
| 조도 캐시 | BH-1750 0.2초 캐시 → I2C 블로킹 방지 |

## 서버 로그 (JSONL)

- 경로 : /home/pi/smart-band/sensor_logs/<device_id>.jsonl
- 헬멧별 파일 분리 (helmet-01.jsonl / helmet-02.jsonl)
- 각 줄 : { ts_server, client_addr, device_id, activity, environment, alert, payload }

## 사고 영상 저장

- 경로 : /home/pi/smart-band/accidents/accident_<device_id>_<timestamp>.mp4
- 조건 : 가속도 급변 > 15m/s² OR BPM 이상 OR 낙상 감지
- 내용 : 사고 직전 10초 버퍼 + 사고 후 10초 → GStreamer mp4mux 합성
- 저장 완료 후 Qt로 ACCIDENT_VIDEO_SAVED JSON 전송

---
