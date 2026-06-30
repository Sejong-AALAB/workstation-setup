# Isaac Sim 설치 가이드

> 본 가이드는 AALAB 워크스테이션 환경을 기준으로 작성되었습니다.
> Isaac Sim 6.0.1 (pip 설치) 설치 및 headless 실행 검증 과정을 담고 있습니다.

---

## 목차

1. [테스트 환경](#테스트-환경)
2. [사전 확인 (기존 설치 여부)](#사전-확인-기존-설치-여부)
3. [NVIDIA 드라이버 업데이트](#nvidia-드라이버-업데이트)
4. [기존 설치 제거 (재설치 시)](#기존-설치-제거-재설치-시)
5. [Isaac Sim 설치](#isaac-sim-설치)
6. [Headless 실행 검증](#headless-실행-검증)
7. [참고사항](#참고사항)
8. [참고 링크](#참고-링크)

---

## 테스트 환경

| 항목 | 요구사항 (Isaac Sim 6.0.1) | 현재 스펙 |
|------|---------------------------|----------|
| OS | Ubuntu 22.04/24.04 | Ubuntu 22.04.5 LTS |
| GPU | RTX 4080 이상 (16GB+) | RTX 4090 × 2 (24GB) |
| **NVIDIA Driver** | **595.58.03 이상** | 595.x (업데이트 완료) |
| RAM | 32GB 이상 | 124GB |
| Storage | 50GB 이상 | 330GB 여유 |
| **Python** | **3.12** | conda 환경으로 분리 |

> GPU 없이 RT Core 미지원 카드(A100, H100 등)는 지원되지 않음

---

## 사전 확인 (기존 설치 여부)

설치 전 기존 설치 흔적이 있는지 반드시 확인합니다.

```bash
find ~ -name "isaac-sim.sh" 2>/dev/null
python3 -c "import isaacsim; print('Isaac Sim installed')" 2>/dev/null || echo "not installed"
conda env list | grep isaac
```

캐시/설정 잔여 파일도 확인:

```bash
ls ~/.local/share/ov 2>/dev/null || echo "없음"
ls ~/.cache/ov 2>/dev/null || echo "없음"
ls ~/.nvidia-omniverse 2>/dev/null || echo "없음"
```

> 본 워크스테이션에서는 과거 Omniverse Launcher 기반 Isaac Sim **5.1.0-rc.19**가 `~/isaac-sim/`에 설치되어 있었고, `isaaclab_env` conda 환경에 Python 바인딩이 구성되어 있었음. 최신 버전(6.0.1)으로 재설치하기 위해 삭제 후 진행함.

---

## NVIDIA 드라이버 업데이트

Isaac Sim 6.0.1은 **드라이버 595.58.03 이상**을 요구합니다. 기존 드라이버(580.159.03)는 요구사항 미달이므로 업데이트가 필요합니다.

> ⚠️ 드라이버 업데이트는 디스플레이 서버와 충돌할 수 있어 **워크스테이션에 직접 앉아서 진행**할 것을 권장합니다. Remote-SSH 중 진행 시 연결이 끊기거나 화면이 깨질 수 있습니다.

```bash
sudo apt-get install -y ubuntu-drivers-common
sudo ubuntu-drivers autoinstall
sudo reboot
```

재부팅 후 확인:

```bash
nvidia-smi
```

Driver 버전이 595 이상으로 표시되면 정상입니다.

---

## 기존 설치 제거 (재설치 시)

```bash
conda deactivate
rm -rf ~/isaac-sim
conda remove -n isaaclab_env --all -y
```

캐시/설정 파일도 깔끔하게 제거:

```bash
rm -rf ~/.local/share/ov
rm -rf ~/.cache/ov
rm -rf ~/.nvidia-omniverse
```

제거 확인:

```bash
ls ~/isaac-sim 2>/dev/null || echo "isaac-sim 삭제 완료"
conda env list | grep isaac
```

---

## Isaac Sim 설치

### 1. conda 환경 생성 (Python 3.12 필수)

```bash
conda create -n isaacsim python=3.12 -y
conda activate isaacsim
```

### 2. tmux 세션에서 설치 (연결 끊겨도 설치 유지)

설치 용량이 크고(~50GB) 시간이 오래 걸리므로(30~60분) tmux 세션 안에서 진행합니다.

```bash
tmux new -s isaacsim
conda activate isaacsim
```

### 3. Isaac Sim 6.0.1 설치

```bash
pip install isaacsim[all,extscache]==6.0.1.0 --extra-index-url https://pypi.nvidia.com
```

설치 중 EULA 동의 프롬프트가 표시되면 `Yes` 입력:

```
Do you accept the EULA? (Yes/No): Yes
The EULA was accepted.
```

### 4. tmux 분리 (설치는 백그라운드에서 계속 진행)

```
Ctrl+B 누른 후 D
```

다시 접속해서 진행 상황 확인:

```bash
tmux attach -t isaacsim
```

### 5. 설치 확인

```bash
conda activate isaacsim
python -c "import isaacsim; print('Isaac Sim OK')"
```

---

## Headless 실행 검증

### 시행착오: 기본 `isaacsim` 명령어는 GUI 앱을 실행함

```bash
isaacsim
```

Remote-SSH 환경(디스플레이 없음)에서 위 명령어를 그냥 실행하면 아래와 같은 에러가 발생합니다.

```
Error: Cannot setup ExternalDragDrop without a default window
Error: Cannot set window title without a default window
ImportError: cannot import name 'get_physxui_interface' from 'omni.physxui'
```

> `omni.physx.supportui` 등 GUI 전용 확장이 디스플레이 없는 환경에서 로드를 시도하다 실패하는 것. headless 모드로 실행하면 해당 확장은 자동으로 제외됨.

### 사용 가능한 experience(.kit) 파일 확인

```bash
find ~/miniconda3/envs/isaacsim/lib/python3.12/site-packages/isaacsim/apps -name "*.kit"
```

확인된 주요 파일:
- `isaacsim.exp.full.kit` — 전체 GUI 앱 (기본값)
- `isaacsim.exp.base.kit` — 경량 버전, **RL 학습에 권장**
- `isaacsim.exp.full.streaming.kit` — 헤드리스 스트리밍용
- `isaacsim.exp.compatibility_check.kit` — 호환성 체크용

### 올바른 headless 실행 명령어

```bash
isaacsim isaacsim.exp.base.kit --no-window
```

### 정상 실행 확인 로그

```
Warp 1.13.0 initialized:
   CUDA Toolkit 12.9, Driver 13.2
   Devices:
     "cuda:0"   : "NVIDIA GeForce RTX 4090" (24 GiB, sm_89, mempool enabled)
     "cuda:1"   : "NVIDIA GeForce RTX 4090" (24 GiB, sm_89, mempool enabled)
...
[ext: isaacsim.exp.base-6.0.1] startup
app ready
```

`app ready`가 출력되고 에러 없이 모든 extension이 startup되면 정상입니다. 종료는 `Ctrl+C`.

---

## 참고사항

### ROS2 연동 — CARLA와 다르게 별도 설치 불필요

Isaac Sim은 ROS2 브리지가 **기본 내장**되어 있습니다. CARLA처럼 `carla-ros-bridge` 같은 별도 패키지 설치가 필요 없고, extension만 활성화하면 ROS2 Humble과 바로 연동됩니다.

### 실행 모드 선택 가이드

| 명령어 | 용도 |
|--------|------|
| `isaacsim` (= `isaacsim isaacsim.exp.full.kit`) | 워크스테이션 직접 접속 + GUI 필요 시 |
| `isaacsim isaacsim.exp.base.kit --no-window` | Remote-SSH + RL 학습 등 headless 작업 |
| `isaacsim isaacsim.exp.full.streaming --no-window` | Headless + 원격 스트리밍 뷰어 필요 시 |

### conda 환경 관리

```bash
conda activate isaacsim    # 환경 진입
conda deactivate           # 환경 나가기
```

### 버전 관련 주의사항

- Isaac Sim 6.0.1은 **release candidate(rc) 단계** 컴포넌트가 일부 포함되어 있어 GUI 전용 확장(`omni.physx.supportui` 등)에서 import 에러가 발생할 수 있음 — headless 모드 사용 시 영향 없음
- pip 패키지 설치 시 Python **3.12 고정** 필요 (3.10, 3.11 환경에서는 wheel 미지원)

---

## 참고 링크

- [Isaac Sim 공식 페이지 (NVIDIA Developer)](https://developer.nvidia.com/isaac/sim)
- [Isaac Sim GitHub](https://github.com/isaac-sim/IsaacSim)
- [Isaac Sim 공식 문서](https://docs.isaacsim.omniverse.nvidia.com/latest/index.html)
- [Isaac Sim 시스템 요구사항](https://docs.isaacsim.omniverse.nvidia.com/latest/installation/requirements.html)
