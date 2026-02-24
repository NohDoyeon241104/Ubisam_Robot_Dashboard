# ROS 2 Jazzy + Gazebo Harmonic
## WSL2 환경 설치 트러블슈팅 가이드

> 작성일: 2026년 2월 24일 | 환경: Ubuntu 24.04 (WSL2)

---

## 1. 개요

ROS 2 Jazzy 환경에서 Gazebo 시뮬레이터를 연동할 때 발생한 오류를 해결한 과정을 기록합니다.

`ros-jazzy-ros-gz` 패키지 설치 후 `gz sim` 명령어를 실행했을 때 `command not found` 오류가 발생하였으며, Gazebo 저장소를 별도로 추가하고 `gz-harmonic`을 설치하여 해결하였습니다.

---

## 2. 환경 정보

| 항목 | 내용 |
|------|------|
| OS | Ubuntu 24.04 LTS (Noble) |
| 실행 환경 | WSL2 (Windows Subsystem for Linux 2) |
| ROS 버전 | ROS 2 Jazzy Jalisco |
| Gazebo 버전 | Gazebo Harmonic |
| 디스플레이 | WSLg (`DISPLAY=:0`, `WAYLAND_DISPLAY=wayland-0`) |

---

## 3. 문제 발생 과정

### 3.1 ros-jazzy-ros-gz 패키지 설치

```bash
sudo apt install ros-jazzy-ros-gz -y
```

✅ 설치는 정상적으로 완료 — `Setting up ros-jazzy-ros-gz` 메시지 출력 확인

### 3.2 설치 확인

```bash
ros2 pkg list | grep gz
```

```
gz_cmake_vendor
gz_common_vendor
gz_dartsim_vendor
gz_fuel_tools_vendor
gz_gui_vendor
gz_math_vendor
gz_msgs_vendor
gz_ogre_next_vendor
gz_physics_vendor
gz_plugin_vendor
gz_rendering_vendor
gz_sensors_vendor
gz_sim_vendor
gz_tools_vendor
gz_transport_vendor
gz_utils_vendor
ros_gz
ros_gz_bridge
ros_gz_image
ros_gz_interfaces
ros_gz_sim
ros_gz_sim_demos
```

### 3.3 launch 실행 시 오류 발생

```bash
ros2 launch ros_gz_sim gz_sim.launch.py
```

```
[ERROR] [launch]: Caught exception in launch (see debug for traceback):
        'NoneType' object has no attribute 'lower'
```

❌ launch 파일 내부에서 환경변수 또는 인자가 `None`으로 반환되어 발생하는 오류

### 3.4 원인 진단

환경변수 및 `gz` 명령어 직접 실행 테스트:

```bash
printenv | grep -E 'DISPLAY|WAYLAND|XDG|GZ|IGN'
```

```
WAYLAND_DISPLAY=wayland-0
DISPLAY=:0
XDG_RUNTIME_DIR=/run/user/1000/
XDG_DATA_DIRS=/usr/local/share:/usr/share:/var/lib/snapd/desktop
```

```bash
gz sim           # → gz: command not found
gz sim --version # → gz: command not found
dpkg -l | grep gazebo  # → (결과 없음)
```

> ⚠️ **원인**: `ros-jazzy-ros-gz`는 ROS-Gazebo 브릿지 패키지이며, Gazebo 시뮬레이터 본체(`gz-harmonic`)는 **별도의 저장소에서 설치**해야 합니다.

---

## 4. 해결 방법

### 4.1 Gazebo 공식 저장소 등록

```bash
# GPG 키 추가
sudo curl https://packages.osrfoundation.org/gazebo.gpg \
    --output /usr/share/keyrings/pkgs-osrf-archive-keyring.gpg

# APT 저장소 등록
echo "deb [arch=$(dpkg --print-architecture) \
    signed-by=/usr/share/keyrings/pkgs-osrf-archive-keyring.gpg] \
    http://packages.osrfoundation.org/gazebo/ubuntu-stable \
    $(lsb_release -cs) main" \
    | sudo tee /etc/apt/sources.list.d/gazebo-stable.list > /dev/null

# 패키지 목록 업데이트
sudo apt update
```

### 4.2 Gazebo Harmonic 설치

```bash
sudo apt install gz-harmonic -y
```

### 4.3 설치 확인

```bash
gz sim --version
# → Gazebo Sim, version 8.x.x

source /opt/ros/jazzy/setup.bash
ros2 launch ros_gz_sim gz_sim.launch.py
# → Gazebo 시뮬레이터 정상 실행 확인 ✅
```

---

## 5. 실행 화면

`ros2 launch ros_gz_sim gz_sim.launch.py` 실행 시 나타나는 Quick Start 화면:

![Gazebo Quick Start](./gazebo_quickstart.png)

*Gazebo Harmonic v8.10.0 — NAO, Panda, Prius 등 예제 월드 선택 가능*

---

## 6. 핵심 요약

- `ros-jazzy-ros-gz`는 ROS-Gazebo **브릿지만** 제공하며, Gazebo 본체는 포함되지 않습니다.
- Gazebo Harmonic은 **OSRF 저장소를 별도로 등록**한 후 `gz-harmonic` 패키지를 설치해야 합니다.
- ROS 2 Jazzy의 공식 Gazebo 조합은 **Gazebo Harmonic**입니다.
- WSL2 + WSLg 환경에서 `DISPLAY=:0`, `WAYLAND_DISPLAY=wayland-0`이면 GUI 실행이 가능합니다.
