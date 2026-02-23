# 🤖 ROS2 실시간 대시보드 아키텍처 정리
> 작성일: 2026-02-23 | 환경: WSL2 + Ubuntu 24.04 + ROS2 Jazzy + Vue 3

---

## 📁 프로젝트 구조

```
ubisam_robot_ui/
├── src/
│   ├── App.vue                  ← 메인 진입점, Rosbridge 연결 상태 표시
│   └── components/
│       ├── SensorChart.vue      ← 실시간 메시지 수신 카운트 그래프
│       └── TurtleMap.vue        ← 거북이 위치 궤적 + 방향 Canvas
│       └── HelloWorld.vue        ← 기존 컴포넌트 Vue       
```

---

## 🖥️ 실행 터미널 구성

> 총 4개의 터미널이 동시에 실행되어야 합니다.

| 터미널 | 역할 | 실행 명령어 |
|--------|------|------------|
| **A** | turtlesim 시뮬레이터 | `ros2 run turtlesim turtlesim_node` |
| **B** | Rosbridge WebSocket 서버 | `ros2 launch rosbridge_server rosbridge_websocket_launch.xml` |
| **C** | Vue 개발 서버 | `npm run dev` |
| **D** | 거북이 키보드 조종 | `ros2 run turtlesim turtle_teleop_key` |

### 터미널별 실행 전 공통 선행 명령어
```bash
source /opt/ros/jazzy/setup.bash
```
> 터미널 A, B, D 에서 ros2 명령어 실행 전 반드시 입력 (또는 ~/.bashrc에 등록된 경우 생략 가능)

---

## 🔁 전체 데이터 흐름

```
[터미널 D] turtle_teleop_key
       ↓  화살표 키 입력
       ↓  /turtle1/cmd_vel 토픽 (geometry_msgs/msg/Twist)
       ↓
[터미널 A] turtlesim_node
       ↓  /turtle1/pose 토픽 발행 (turtlesim/msg/Pose)
       ↓  x, y, theta, linear_velocity, angular_velocity
       ↓
[터미널 B] rosbridge_server (ws://localhost:9090)
       ↓  WebSocket 프로토콜로 변환
       ↓
[터미널 C] Vue 개발 서버 (http://localhost:5173)
       ├── App.vue          → 연결 상태, /chatter 메시지 수신
       ├── SensorChart.vue  → /chatter 메시지 수신 카운트 그래프
       └── TurtleMap.vue    → /turtle1/pose 수신 → Canvas 렌더링
```

---

## 📡 토픽별 상세 정보

### 1. `/chatter` — Hello World 메시지
| 항목 | 내용 |
|------|------|
| 방향 | turtlesim → Rosbridge → Vue |
| 메시지 타입 | `std_msgs/msg/String` |
| 발행 주체 | `ros2 run demo_nodes_cpp talker` |
| 구독 컴포넌트 | `App.vue`, `SensorChart.vue` |
| 용도 | 연결 상태 확인, 수신 카운트 그래프 |

### 2. `/turtle1/pose` — 거북이 위치 및 방향
| 항목 | 내용 |
|------|------|
| 방향 | turtlesim → Rosbridge → Vue |
| 메시지 타입 | `turtlesim/msg/Pose` |
| 발행 주체 | `turtlesim_node` (자동 발행) |
| 구독 컴포넌트 | `TurtleMap.vue` |
| 주요 필드 | `x`, `y` (0~11), `theta` (라디안) |

### 3. `/turtle1/cmd_vel` — 거북이 이동 명령
| 항목 | 내용 |
|------|------|
| 방향 | turtle_teleop_key → turtlesim |
| 메시지 타입 | `geometry_msgs/msg/Twist` |
| 발행 주체 | `turtle_teleop_key` (터미널 D) |
| 주요 필드 | `linear.x` (전진/후진), `angular.z` (회전) |

---

## 🧩 컴포넌트별 상세 구조

---

### App.vue — 메인 연결 관리

**역할:** Rosbridge 연결 상태 표시 + `/chatter` 토픽 구독

```
Rosbridge (ws://localhost:9090)
       ↓  연결 이벤트 (connection / error / close)
App.vue
  ├── status: '✅ 연결됨' / '❌ 오류' / '🔌 연결 끊김'
  └── message: 수신된 마지막 Hello World 문자열
```

**핵심 코드 구조:**
```js
const ros = new Ros({ url: 'ws://localhost:9090' })
ros.on('connection', () => { status.value = '✅ 연결됨' })

const listener = new Topic({
  ros,
  name: '/chatter',
  messageType: 'std_msgs/msg/String'
})
listener.subscribe((msg) => { message.value = msg.data })
```

---

### SensorChart.vue — 실시간 수신 카운트 그래프

**역할:** `/chatter` 토픽 수신 횟수를 시간축 기준 Line Chart로 시각화

```
/chatter 토픽 수신
       ↓
count++ (수신 횟수 누적)
       ↓
chartData.labels  ← 현재 시각 (toLocaleTimeString)
chartData.data    ← count 값
       ↓  최대 20개 유지 (shift()로 오래된 데이터 제거)
Chart.js Line Chart 렌더링
```

**사용 라이브러리:**
```bash
npm install chart.js vue-chartjs
```

**성능 최적화 포인트:**
```js
// 최대 20개 초과 시 가장 오래된 데이터 제거
if (chartData.value.labels.length > 20) {
  chartData.value.labels.shift()
  chartData.value.datasets[0].data.shift()
}
```

---

### TurtleMap.vue — 거북이 위치 추적 Canvas

**역할:** `/turtle1/pose` 토픽을 구독하여 Canvas에 이동 궤적과 방향 화살표를 실시간 렌더링

```
/turtle1/pose 수신 (x, y, theta)
       ↓
좌표 변환
  turtlesim x (0~11) → canvas px (0~500) : x * (500/11)
  turtlesim y (0~11) → canvas px 반전    : 500 - y * (500/11)
       ↓
trail 배열에 { x, y } 추가 (최대 500개)
       ↓
Canvas 렌더링
  ├── 궤적: trail 배열 → 노란 점 (오래될수록 투명)
  ├── 현재 위치: 흰색 원
  └── 방향: theta → 빨간 화살표
```

**좌표계 변환 이유:**
```
turtlesim: 좌하단이 원점(0,0), y축이 위로 증가
Canvas:    좌상단이 원점(0,0), y축이 아래로 증가
→ canvas_y = CANVAS_SIZE - (ros_y * SCALE) 로 반전 필요
```

**사용 기술:**
- `<canvas>` + `CanvasRenderingContext2D`
- `roslibjs` Topic.subscribe
- `ref()` 로 반응형 좌표 표시

---

## 🔌 Rosbridge 역할 요약

```
ROS2 세계                    웹 세계
─────────────────────────────────────────────
turtlesim_node               Vue 컴포넌트
  /turtle1/pose  ──────────→  TurtleMap.vue
  /chatter       ──────────→  App.vue
                              SensorChart.vue

turtle_teleop_key
  /turtle1/cmd_vel ←───────  (추후 추가 예정)
                              TurtleMap.vue 키보드 조종

         ↕ 중간에서 변환해주는 것이 Rosbridge
         ROS2 메시지 ↔ JSON over WebSocket
```

---

## ⚡ 전체 실행 순서 (매번 시작할 때)

```bash
# 터미널 A — turtlesim 실행
source /opt/ros/jazzy/setup.bash
ros2 run turtlesim turtlesim_node

# 터미널 B — Rosbridge 서버 실행
source /opt/ros/jazzy/setup.bash
ros2 launch rosbridge_server rosbridge_websocket_launch.xml

# 터미널 C — Vue 개발 서버 실행
cd ~/projects/ubisam_robot_ui
npm run dev

# 터미널 D — 거북이 키보드 조종 (선택)
source /opt/ros/jazzy/setup.bash
ros2 run turtlesim turtle_teleop_key
```

> 브라우저에서 `http://localhost:5173` 접속

---

## 📌 다음 단계 예정

- [ ] `TurtleMap.vue`에 키보드 조종 기능 추가 (브라우저 → `/turtle1/cmd_vel` 발행)
- [ ] Docker화: 현재 환경을 이미지로 저장