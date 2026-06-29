# MuJoCo RL 환경 설치 가이드

> 본 가이드는 AALAB 워크스테이션 환경을 기준으로 작성되었습니다.
> MuJoCo + Gymnasium + Stable-Baselines3 + PyTorch 설치 및 검증 과정을 담고 있습니다.

---

## 목차

1. [테스트 환경](#테스트-환경)
2. [사전 확인](#사전-확인)
3. [MuJoCo 설치](#mujoco-설치)
4. [RL 학습 환경 설치](#rl-학습-환경-설치)
5. [전체 파이프라인 검증](#전체-파이프라인-검증)
6. [참고사항](#참고사항)
7. [참고 링크](#참고-링크)

---

## 테스트 환경

| 항목 | 스펙 |
|------|------|
| OS | Ubuntu 22.04.5 LTS |
| GPU | NVIDIA GeForce RTX 4090 × 2 |
| NVIDIA Driver | 580.159.03 |
| CUDA (드라이버 지원) | 13.0 |
| RAM | 124GB |
| Storage | 480GB NVMe |
| Python | 3.10.12 |

스펙 확인 명령어:

```bash
lsb_release -a
nvidia-smi
python3 --version
free -h && df -h ~
```

---

## 사전 확인

### Miniconda 설치 여부 확인

```bash
conda --version
```

설치되어 있지 않다면 아래 명령어로 설치:

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
source ~/.bashrc
```

---

## MuJoCo 설치

### 1. conda 환경 생성

```bash
conda create -n mujoco python=3.10 -y
conda activate mujoco
```

> Python 3.10 이상 필수

### 2. MuJoCo 설치

```bash
pip install mujoco
```

> MuJoCo 3.x부터는 pip 한 줄로 설치 완료.
> 라이브러리가 패키지에 내장되어 있어 별도 바이너리 다운로드 및 환경변수 설정 불필요.

### 3. 설치 확인

```bash
python -c "import mujoco; print(mujoco.__version__)"
```

정상 출력 예시:
```
3.10.0
```

```bash
python -c "import mujoco; m = mujoco.MjModel.from_xml_string('<mujoco/>'); print('OK')"
```

정상 출력 예시:
```
OK
```

---

## RL 학습 환경 설치

> `conda activate mujoco` 상태에서 진행

### 1. Gymnasium + Stable-Baselines3 설치

```bash
pip install gymnasium[mujoco] stable-baselines3[extra]
```

- `gymnasium[mujoco]`: MuJoCo 기반 환경 포함 (Ant, HalfCheetah, Hopper 등)
- `stable-baselines3[extra]`: TensorBoard, OpenCV, pandas, matplotlib 포함

### 2. PyTorch 설치

```bash
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu128
```

> NVIDIA Driver 580 기준 CUDA 13.0까지 지원하지만,
> PyTorch 공식 wheel은 현재 **CUDA 12.8(cu128)까지** 제공.
> Stable-Baselines3 2.9.0은 **PyTorch 2.8 이상** 필요.

### 3. 설치 확인

```bash
python -c "
import gymnasium as gym
import stable_baselines3 as sb3
import torch
print('gymnasium:', gym.__version__)
print('stable-baselines3:', sb3.__version__)
print('torch:', torch.__version__)
print('CUDA available:', torch.cuda.is_available())
print('GPU count:', torch.cuda.device_count())
"
```

정상 출력 예시:
```
gymnasium: 1.3.0
stable-baselines3: 2.9.0
torch: 2.10.0+cu128
CUDA available: True
GPU count: 2
```

---

## 전체 파이프라인 검증

아래 스크립트로 MuJoCo 환경 로드 → PPO 학습 → 완료까지 한번에 검증:

```bash
python -c "
import gymnasium as gym
from stable_baselines3 import PPO

env = gym.make('Ant-v5')
model = PPO('MlpPolicy', env, verbose=1, device='cpu')
model.learn(total_timesteps=5000)
print('Training complete!')
env.close()
"
```

정상 출력 예시:
```
---------------------------------
| rollout/           |          |
|    ep_len_mean     | ...      |
|    ep_rew_mean     | ...      |
...
Training complete!
```

---

## 참고사항

### MlpPolicy vs CnnPolicy 디바이스 선택

| Policy 종류 | 입력 | 권장 device |
|------------|------|------------|
| `MlpPolicy` | 상태벡터 (저차원) | `device='cpu'` |
| `CnnPolicy` | 이미지 (고차원) | `device='cuda'` |

PPO + MlpPolicy 조합은 신경망 크기가 작아 GPU 전송 오버헤드가 오히려 커서 **CPU가 더 빠릅니다.**
이미지 기반 환경(픽셀 입력)에서는 `CnnPolicy + cuda` 사용.

### 워크스테이션 직접 접속 vs Remote-SSH

| 상황 | GUI 뷰어 | `MUJOCO_GL` 설정 |
|------|---------|-----------------|
| 워크스테이션 직접 앉아서 | 창 정상 표시 ✅ | 불필요 |
| Remote-SSH (원격 접속) | 창 안 뜸 | `egl` 설정 필요 |

워크스테이션 직접 접속 시 아래처럼 viewer 창을 바로 띄울 수 있음:

```python
import mujoco
import mujoco.viewer

m = mujoco.MjModel.from_xml_path('model.xml')
d = mujoco.MjData(m)

with mujoco.viewer.launch_passive(m, d) as viewer:
    while viewer.is_running():
        mujoco.mj_step(m, d)
        viewer.sync()
```

> 권장 워크플로우: 학습은 Remote-SSH로 돌리고, 환경 디버깅 및 시각화는 워크스테이션에 직접 앉아서 진행.

### Headless 서버 환경 (Remote-SSH)

디스플레이 없이 학습 돌릴 때 EGL 백엔드 설정 권장:

```bash
export MUJOCO_GL=egl
```

`~/.bashrc`에 추가하면 매번 설정 불필요:

```bash
echo 'export MUJOCO_GL=egl' >> ~/.bashrc
source ~/.bashrc
```

### conda 환경 관리

```bash
conda activate mujoco    # 환경 진입
conda deactivate         # 환경 나가기
conda env list           # 전체 환경 목록
```

---

## 참고 링크

- [MuJoCo 공식 GitHub](https://github.com/google-deepmind/mujoco)
- [Gymnasium 공식 문서](https://gymnasium.farama.org)
- [Stable-Baselines3 공식 GitHub](https://github.com/DLR-RM/stable-baselines3)
- [PyTorch 설치 페이지](https://pytorch.org/get-started/locally/)
