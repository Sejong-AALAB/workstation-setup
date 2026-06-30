# CARLA + ROS2 브리지 설치 가이드

> 본 가이드는 AALAB 워크스테이션 환경을 기준으로 작성되었습니다.
> CARLA 0.9.16 + ROS2 Humble 연동(`carla-ros-bridge`) 설치 및 검증 과정을 담고 있습니다.
> [CARLA_Setup.md](./CARLA_Setup.md) 설치가 선행되어 있어야 합니다.

---

## 목차

1. [테스트 환경](#테스트-환경)
2. [버전 호환성 주의사항](#버전-호환성-주의사항)
3. [워크스페이스 생성 및 빌드](#워크스페이스-생성-및-빌드)
4. [빌드 중 발생한 문제와 해결](#빌드-중-발생한-문제와-해결)
5. [실행 및 연동 테스트](#실행-및-연동-테스트)
6. [참고사항](#참고사항)
7. [참고 링크](#참고-링크)

---

## 테스트 환경

| 항목 | 스펙 |
|------|------|
| OS | Ubuntu 22.04.5 LTS |
| ROS2 | Humble |
| CARLA | 0.9.16 (UE4) |
| 시스템 Python | 3.10 (ROS2 Humble 기준) |

> 기존 `~/ros2_ws`(OrbbecSDK_ROS2, usb_cam 등 하드웨어 드라이버용)와 분리하여 `~/carla_ros2_ws`로 별도 워크스페이스 구성.

---

## 버전 호환성 주의사항

`carla-ros-bridge` 공식 레포에는 명확한 CARLA-ROS2 호환성 매트릭스가 공개되어 있지 않습니다. 확인된 사항:

| 항목 | 내용 |
|------|------|
| 최신 **태그된 릴리즈** | 0.9.12 (2022년, 오래된 CARLA 버전 대상 — 사용하지 말 것) |
| **master 브랜치** (태그 없음) | "CARLA 0.9.13 이상" 명시 — 0.9.16 포함 |
| ROS2 distro 공식 매트릭스 | 없음. 공식 문서는 Foxy 기준이지만 Humble도 동일한 colcon/ament 구조라 빌드 가능 |

**따라서 CARLA 0.9.16 + ROS2 Humble 조합에는 `master` 브랜치를 사용합니다.** (특정 release 태그 X)

---

## 워크스페이스 생성 및 빌드

> ⚠️ ROS2는 **시스템 Python(3.10)**을 사용합니다. conda 환경이 켜져 있으면 `python3` 경로가 충돌하여 `rosdep`, `colcon build`가 실패할 수 있습니다. **반드시 `conda deactivate` 후 진행하세요.**

### 1. conda 비활성화

```bash
conda deactivate
```

### 2. 워크스페이스 생성 및 레포 클론

```bash
mkdir -p ~/carla_ros2_ws/src && cd ~/carla_ros2_ws
git clone --recurse-submodules https://github.com/carla-simulator/ros-bridge.git src/ros-bridge
```

### 3. rosdep 의존성 설치

```bash
source /opt/ros/humble/setup.bash
rosdep update
rosdep install --from-paths src --ignore-src --rosdistro humble -y
```

> `rosdep update` 시 `/usr/share/python3-rosdep2/debian.yaml` 관련 에러가 날 수 있는데, 다른 소스(humble 등)는 정상 업데이트되므로 무시 가능. 해결하려면:
> ```bash
> sudo apt install --reinstall python3-rosdep2
> ```

### 4. 빌드 (워크스페이스 루트에서 실행)

```bash
cd ~/carla_ros2_ws    # src의 상위 폴더. src 안에서 실행하면 안 됨
colcon build --symlink-install
```

> `--symlink-install`: Python 패키지를 심볼릭 링크로 설치하여, 나중에 launch 파일 등 Python 스크립트 수정 시 재빌드 없이 바로 반영됨.
> 기존에 일반 `colcon build`로 빌드된 적이 있다면 `rm -rf build install log` 후 클린 빌드해야 충돌 없음.

---

## 빌드 중 발생한 문제와 해결

### 문제 1 — `pcl_recorder` 빌드 실패: `tf2_eigen/tf2_eigen.h` 헤더 없음

```
fatal error: tf2_eigen/tf2_eigen.h: 그런 파일이나 디렉터리가 없습니다
```

**원인:** `ros-bridge` 레포 자체의 CMakeLists.txt 버그. `find_package(tf2_eigen REQUIRED)`로 패키지는 찾지만, 실제 빌드 타겟(`ament_target_dependencies`)에는 연결되지 않아 include 경로가 잡히지 않음.

```cmake
# pcl_recorder/CMakeLists.txt (수정 전)
ament_target_dependencies(${PROJECT_NAME}_node rclcpp sensor_msgs
                          pcl_conversions tf2 tf2_ros)   # tf2_eigen 빠짐
```

**해결:** `tf2_eigen`을 `ament_target_dependencies`에 직접 추가.

```bash
nano ~/carla_ros2_ws/src/ros-bridge/pcl_recorder/CMakeLists.txt
```

```cmake
# 수정 후
ament_target_dependencies(${PROJECT_NAME}_node rclcpp sensor_msgs
                          pcl_conversions tf2 tf2_ros tf2_eigen)
```

수정 후 재빌드:
```bash
rm -rf build/pcl_recorder install/pcl_recorder
colcon build --packages-select pcl_recorder
```

> 빌드 성공 후 `#warning This header is obsolete, please include tf2_eigen/tf2_eigen.hpp instead` 경고가 뜨는데, 이는 ros-bridge 코드가 구버전 헤더(`.h`)를 참조해서 나는 경고일 뿐 빌드는 정상 완료됨. 무시 가능.

### 문제 2 — 브리지 실행 시 `ModuleNotFoundError: No module named 'carla'`

**원인:** `carla` Python 패키지가 conda 환경(`carla` env)에만 설치되어 있고, ROS2는 시스템 Python(3.10)으로 동작하기 때문에 못 찾음.

**해결:** 동일한 wheel을 시스템 Python에도 설치.

```bash
conda deactivate
pip3 install ~/CARLA_0.9.16/PythonAPI/carla/dist/carla-0.9.16-cp310-cp310-manylinux_2_31_x86_64.whl
```

### 문제 3 — `CARLA python module version 0.9.13 required. Found: 0.9.16`

**원인:** `bridge.py`가 같은 디렉토리의 `CARLA_VERSION` 파일 내용과 설치된 carla 모듈 버전을 **정확히 일치**하는지(`!=` 비교) 검사함. master 브랜치 기준 파일 내용이 `0.9.13`으로 고정되어 있어 0.9.16을 거부함.

```python
# bridge.py
if LooseVersion(dist.version) != LooseVersion(CarlaRosBridge.CARLA_VERSION):
    carla_bridge.logfatal("CARLA python module version {} required. Found: {}".format(...))
    sys.exit(1)
```

**해결:** `CARLA_VERSION` 파일 내용을 실제 설치 버전으로 수정.

```bash
echo "0.9.16" > ~/carla_ros2_ws/src/ros-bridge/carla_ros_bridge/src/carla_ros_bridge/CARLA_VERSION
```

`--symlink-install`로 빌드했다면 재빌드 없이 즉시 반영됨.

---

## 실행 및 연동 테스트

총 3개 터미널이 필요합니다.

### 터미널 1 — CARLA 서버 (conda `carla` 환경)

```bash
conda activate carla
cd ~/CARLA_0.9.16
./CarlaUE4.sh -RenderOffScreen
```

### 터미널 2 — ROS2 브리지 (시스템 Python, conda 비활성화)

```bash
conda deactivate
source /opt/ros/humble/setup.bash
source ~/carla_ros2_ws/install/setup.bash
ros2 launch carla_ros_bridge carla_ros_bridge.launch.py
```

정상 실행 시 `Created TrafficLight(...)`, `Created Actor(...)` 등 맵의 모든 액터가 ROS2로 등록되는 로그가 출력됩니다.

### 터미널 3 — 토픽 확인

```bash
source /opt/ros/humble/setup.bash
ros2 topic list
```

정상 출력 예시:
```
/carla/control
/carla/debug_marker
/carla/status
/carla/weather_control
/carla/world_info
/clock
/parameter_events
/rosout
```

`/carla/...` 토픽이 보이면 연동 성공입니다.

---

## 참고사항

### conda / 시스템 Python 역할 분리

| 작업 | Python 환경 |
|------|------------|
| CARLA 서버 실행 | conda `carla` env |
| CARLA Python 클라이언트 단독 스크립트 | conda `carla` env |
| ROS2 워크스페이스 빌드 (`rosdep`, `colcon build`) | 시스템 Python (conda 비활성화) |
| ROS2 브리지 실행 (`ros2 launch`) | 시스템 Python (conda 비활성화) |

### 기존 ROS2 워크스페이스와 분리

연구실 기존 `~/ros2_ws`(하드웨어 드라이버용 OrbbecSDK_ROS2, usb_cam)와 의존성 충돌을 막기 위해 `~/carla_ros2_ws`를 완전히 별도로 구성했습니다. 두 워크스페이스를 동시에 source할 경우 패키지명 충돌 여부를 확인하세요.

### Isaac Sim과의 차이

Isaac Sim은 ROS2 브리지가 **기본 내장**되어 있어 별도 설치가 필요 없습니다. CARLA만 이런 별도 빌드 과정이 필요합니다 ([IsaacSim_Setup.md](./IsaacSim_Setup.md) 참고).

---

## 참고 링크

- [carla-ros-bridge GitHub](https://github.com/carla-simulator/ros-bridge)
- [carla-ros-bridge 공식 문서](https://carla.readthedocs.io/projects/ros-bridge/en/latest/)
- [ROS2 Humble 공식 문서](https://docs.ros.org/en/humble/)
