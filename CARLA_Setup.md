# CARLA 설치 가이드

> 본 가이드는 AALAB 워크스테이션 환경을 기준으로 작성되었습니다.
> CARLA 0.9.16 (UE4 기반) 설치 및 검증 과정을 담고 있습니다.

---

## 목차

1. [테스트 환경](#테스트-환경)
2. [버전 선택 가이드](#버전-선택-가이드)
3. [사전 확인](#사전-확인)
4. [CARLA 설치](#carla-설치)
5. [서버 실행 및 연결 테스트](#서버-실행-및-연결-테스트)
6. [참고사항](#참고사항)
7. [참고 링크](#참고-링크)

---

## 테스트 환경

| 항목 | 스펙 |
|------|------|
| OS | Ubuntu 22.04.5 LTS |
| GPU | NVIDIA GeForce RTX 4090 × 2 |
| NVIDIA Driver | 580.159.03 |
| RAM | 124GB |
| Storage | 480GB NVMe (330GB 여유) |
| Python | 3.10 (conda 환경) |
| ROS2 | Humble |

---

## 버전 선택 가이드

CARLA는 현재 두 가지 버전이 존재합니다.

| 버전 | 엔진 | 설치 방식 | Python 지원 | 권장 여부 |
|------|------|----------|------------|---------|
| **0.9.16** | **UE4** | **프리빌드 바이너리** | **3.8–3.12** | **✅ 권장** |
| 0.10.0 | UE5.5 | 소스 빌드 (복잡) | 미확인 | ❌ |

**RL 실험 및 군집주행 연구에는 0.9.16(UE4)을 권장합니다.**

- Python API, Traffic Manager, 센서 구성이 성숙하고 안정적
- 관련 논문 코드 대부분이 0.9.x 기준
- ROS2 Humble 공식 연동 지원
- 0.10.0(UE5)은 그래픽 품질 향상이 목적이며 RL 실험에서 이점 없음

---

## 사전 확인

### CARLA 설치 여부 확인

```bash
find ~ -name "CarlaUE4.sh" 2>/dev/null
python3 -c "import carla; print(carla.__version__)" 2>/dev/null || echo "carla not installed"
```

### ROS2 설치 확인

```bash
printenv ROS_DISTRO
python3 -c "import rclpy; print('rclpy OK')"
```

> Ubuntu 22.04 + ROS2 Humble 조합이 CARLA와 공식 호환됩니다.

---

## CARLA 설치

### 1. conda 환경 생성

```bash
conda create -n carla python=3.10 -y
conda activate carla
```

### 2. CARLA 0.9.16 다운로드

```bash
cd ~
wget -L -O CARLA_0.9.16.tar.gz https://tiny.carla.org/carla-0-9-16-linux
```

> 약 8GB, 네트워크 환경에 따라 시간이 걸릴 수 있습니다.
> `-L` 옵션으로 리다이렉트를 자동으로 따라갑니다.

### 3. 압축 해제

```bash
mkdir ~/CARLA_0.9.16
tar -xzf CARLA_0.9.16.tar.gz -C ~/CARLA_0.9.16
```

### 4. Python 클라이언트 wheel 확인

```bash
ls ~/CARLA_0.9.16/PythonAPI/carla/dist/
```

Python 3.10 환경 기준 아래 파일이 있어야 합니다:
```
carla-0.9.16-cp310-cp310-manylinux_2_31_x86_64.whl
```

### 5. Python 클라이언트 설치

```bash
pip install ~/CARLA_0.9.16/PythonAPI/carla/dist/carla-0.9.16-cp310-cp310-manylinux_2_31_x86_64.whl
```

### 6. 설치 확인

```bash
python -c "import carla; print(dir(carla))"
```

`Client`, `World`, `Vehicle`, `TrafficManager` 등이 출력되면 정상입니다.

---

## 서버 실행 및 연결 테스트

### 서버 실행

**워크스테이션 직접 접속 시 (GUI 모드):**
```bash
cd ~/CARLA_0.9.16
./CarlaUE4.sh
```

**Remote-SSH 환경 (Headless 모드):**
```bash
cd ~/CARLA_0.9.16
./CarlaUE4.sh -RenderOffScreen
```

> Remote-SSH 환경에서는 반드시 `-RenderOffScreen` 옵션을 사용해야 합니다.

### 연결 테스트 (새 터미널에서)

```bash
conda activate carla
python -c "
import carla
client = carla.Client('localhost', 2000)
client.set_timeout(10.0)
world = client.get_world()
print('Connected! Map:', world.get_map().name)
"
```

정상 출력 예시:
```
Connected! Map: Carla/Maps/Town10HD_Opt
```

### 서버 종료

```bash
Ctrl+C
```

---

## 참고사항

### 포트 사용

CARLA 서버는 기본적으로 아래 포트를 사용합니다:

| 포트 | 용도 |
|------|------|
| 2000 | 클라이언트 연결 |
| 2001 | 스트림 |

방화벽에서 두 포트가 열려 있어야 합니다.

### conda 환경 관리

```bash
conda activate carla    # 환경 진입
conda deactivate        # 환경 나가기
```

### ROS2 브리지 연동

CARLA + ROS2 Humble 연동 시 `carla-ros-bridge`를 별도 워크스페이스에 설치합니다.
기존 ROS2 워크스페이스(`~/ros2_ws`)와 분리하여 의존성 충돌을 방지하세요.

```bash
mkdir -p ~/carla_ros2_ws/src
```

> carla-ros-bridge 설치는 별도 가이드를 참고하세요.

### 시뮬레이터별 conda 환경 분리

세 시뮬레이터의 Python 의존성이 다르므로 반드시 환경을 분리하여 사용하세요.

```bash
conda activate mujoco   # MuJoCo 실험 시
conda activate carla    # CARLA 실험 시
```

---

## 참고 링크

- [CARLA 공식 홈페이지](https://carla.org/)
- [CARLA GitHub](https://github.com/carla-simulator/carla)
- [CARLA 공식 문서](https://carla.readthedocs.io/en/latest/)
- [CARLA ROS2 Bridge](https://github.com/carla-simulator/ros-bridge)
