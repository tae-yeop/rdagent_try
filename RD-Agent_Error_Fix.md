# RD-Agent 실행 오류 해결 가이드

## 문제: numpy.typing AttributeError

### 오류 메시지
```
AttributeError: module 'numpy' has no attribute 'typing'. Did you mean: '_typing'?
```

### 원인
RD-Agent의 qlib Docker 이미지가:
1. 오래된 qlib 커밋에 고정 (`3e72593b8c985f01979bebcf646658002ac43b00`)
2. numpy 버전이 명시되지 않아 **numpy < 1.20** 설치됨
3. pandas가 의존하는 `numexpr`이 `numpy.typing` 필요 (numpy 1.20+)

---

## 해결 방법

### 방법 1: Docker 이미지 재빌드 (권장)

Docker 이미지의 numpy를 업그레이드합니다.

#### Step 1: Dockerfile 수정

```bash
# 1. Dockerfile 복사
mkdir -p ~/fin_agents/RD-Agent/custom_docker
cp /home/tyk/.local/lib/python3.10/site-packages/rdagent/scenarios/qlib/docker/Dockerfile \
   ~/fin_agents/RD-Agent/custom_docker/Dockerfile

# 2. Dockerfile 수정
nano ~/fin_agents/RD-Agent/custom_docker/Dockerfile
```

**수정할 내용:**

```dockerfile
FROM pytorch/pytorch:2.2.1-cuda12.1-cudnn8-runtime

RUN apt-get clean && apt-get update && apt-get install -y \  
    curl \  
    vim \  
    git \  
    build-essential \
    && rm -rf /var/lib/apt/lists/* 

RUN git clone https://github.com/microsoft/qlib.git

WORKDIR /workspace/qlib

RUN git fetch && git reset 3e72593b8c985f01979bebcf646658002ac43b00 --hard

# ========== 수정: numpy 버전 명시 ==========
RUN python -m pip install --upgrade cython
RUN python -m pip install "numpy>=1.23.0,<2.0.0"  # ← 추가
RUN python -m pip install "numexpr>=2.8.5"        # ← 추가
RUN python -m pip install -e .

RUN pip install catboost
RUN pip install xgboost
RUN pip install scipy==1.11.4
RUN pip install tables
# ==========================================
```

#### Step 2: 이미지 재빌드

```bash
cd ~/fin_agents/RD-Agent/custom_docker

# 기존 이미지 삭제
docker rmi local_qlib:latest

# 새 이미지 빌드
docker build -t local_qlib:latest .
```

#### Step 3: 실행

```bash
cd ~/fin_agents/RD-Agent
rdagent fin_quant
```

---

### 방법 2: 이미 빌드된 이미지 수정 (빠른 임시 해결)

기존 Docker 이미지를 직접 수정합니다.

```bash
# 1. 컨테이너 실행
docker run -it local_qlib:latest /bin/bash

# 2. 컨테이너 내부에서 numpy 업그레이드
pip install --upgrade "numpy>=1.23.0,<2.0.0"
pip install --upgrade "numexpr>=2.8.5"

# 3. 컨테이너 ID 확인
exit
docker ps -a | head -2

# 4. 이미지로 커밋 (CONTAINER_ID를 실제 ID로 변경)
docker commit <CONTAINER_ID> local_qlib:latest

# 5. 실행
cd ~/fin_agents/RD-Agent
rdagent fin_quant
```

---

### 방법 3: .env 파일 확인 (API Key 설정)

RD-Agent는 **반드시 LLM API가 필요**합니다.

```bash
# 1. .env 파일 생성
cd ~/fin_agents/RD-Agent
cp .env.example .env

# 2. .env 파일 편집
nano .env
```

**최소 설정 (OpenAI 사용):**

```bash
# Chat Model (필수)
OPENAI_API_KEY="sk-your-openai-api-key-here"
CHAT_MODEL="gpt-4o-mini"  # 또는 gpt-4o

# Embedding Model (필수, RAG용)
EMBEDDING_MODEL="text-embedding-3-small"
```

**DeepSeek 사용 (저렴):**

```bash
# Chat Model
OPENAI_API_KEY="sk-your-deepseek-key"
OPENAI_API_BASE="https://api.deepseek.com/v1"
CHAT_MODEL="deepseek-chat"

# Embedding Model (별도 서비스)
LITELLM_PROXY_API_KEY="sk-your-siliconflow-key"
LITELLM_PROXY_API_BASE="https://api.siliconflow.cn/v1"
EMBEDDING_MODEL="litellm_proxy/BAAI/bge-m3"
```

---

### 방법 4: qlib 최신 버전 사용 (고급)

Dockerfile의 qlib 커밋을 최신으로 변경합니다.

```dockerfile
# 기존 (오래된 커밋)
RUN git fetch && git reset 3e72593b8c985f01979bebcf646658002ac43b00 --hard

# 변경 (최신 버전)
RUN git fetch && git checkout main  # 또는 특정 태그
```

**주의:** qlib API가 변경되었을 수 있어 RD-Agent와 호환성 문제 발생 가능

---

## 전체 해결 스크립트

한 번에 실행할 수 있는 스크립트:

```bash
#!/bin/bash
# fix_rdagent.sh

set -e

echo "=== RD-Agent qlib Docker 이미지 수정 ==="

# 1. custom_docker 폴더 생성
mkdir -p ~/fin_agents/RD-Agent/custom_docker
cd ~/fin_agents/RD-Agent/custom_docker

# 2. Dockerfile 생성
cat > Dockerfile << 'EOF'
FROM pytorch/pytorch:2.2.1-cuda12.1-cudnn8-runtime

RUN apt-get clean && apt-get update && apt-get install -y \  
    curl \  
    vim \  
    git \  
    build-essential \
    && rm -rf /var/lib/apt/lists/* 

RUN git clone https://github.com/microsoft/qlib.git

WORKDIR /workspace/qlib

RUN git fetch && git reset 3e72593b8c985f01979bebcf646658002ac43b00 --hard

# Fix numpy version conflict
RUN python -m pip install --upgrade cython
RUN python -m pip install "numpy>=1.23.0,<2.0.0"
RUN python -m pip install "numexpr>=2.8.5"
RUN python -m pip install -e .

RUN pip install catboost
RUN pip install xgboost
RUN pip install scipy==1.11.4
RUN pip install tables
EOF

# 3. 기존 이미지 삭제
echo "기존 이미지 삭제..."
docker rmi local_qlib:latest 2>/dev/null || true

# 4. 새 이미지 빌드
echo "새 이미지 빌드 중... (5-10분 소요)"
docker build -t local_qlib:latest .

echo "=== 완료! ==="
echo "이제 실행하세요: rdagent fin_quant"
```

**실행:**

```bash
chmod +x fix_rdagent.sh
./fix_rdagent.sh
```

---

## .env 파일 템플릿

RD-Agent가 정상 작동하려면 **반드시** .env 파일이 필요합니다.

```bash
# ~/fin_agents/RD-Agent/.env

# ==========================================
# Global configs
# ==========================================
MAX_RETRY=10
RETRY_WAIT_SECONDS=20

# ==========================================
# Chat Model (필수)
# ==========================================
OPENAI_API_KEY="sk-your-api-key-here"
OPENAI_API_BASE="https://api.openai.com/v1"  # 또는 다른 엔드포인트
CHAT_MODEL="gpt-4o-mini"

# ==========================================
# Embedding Model (필수, RAG용)
# ==========================================
EMBEDDING_MODEL="text-embedding-3-small"

# ==========================================
# Cache Settings (선택)
# ==========================================
USE_CHAT_CACHE=True
USE_EMBEDDING_CACHE=True
```

---

## 실행 확인

```bash
# 1. Docker 이미지 확인
docker images | grep local_qlib

# 출력 예시:
# local_qlib    latest    abc123def456    5 minutes ago    10.5GB

# 2. Health check
cd ~/fin_agents/RD-Agent
rdagent health_check

# 3. 실행
rdagent fin_quant
```

---

## 예상 비용

RD-Agent는 **코드 생성**을 하므로 토큰 소비가 많습니다:

### GPT-4o 사용 시:
- Chat Model: ~$2-5 per factor (1000-2500 tokens/call × 10-20 calls)
- Embedding Model: ~$0.01 per factor
- **총 비용**: ~$20-50 per 10 factors

### GPT-4o-mini 사용 시 (권장):
- Chat Model: ~$0.50-1 per factor
- Embedding Model: ~$0.01 per factor
- **총 비용**: ~$5-10 per 10 factors

### DeepSeek Chat 사용 시 (최저가):
- Chat Model: ~$0.05-0.10 per factor
- Embedding Model (BAAI/bge-m3): ~$0.01 per factor
- **총 비용**: ~$0.50-1 per 10 factors

---

## 문제가 계속되면

### 로그 확인:
```bash
# RD-Agent 로그
cd ~/fin_agents/RD-Agent
ls -la log/

# Docker 로그
docker logs <CONTAINER_ID>
```

### GitHub Issue 확인:
- https://github.com/microsoft/RD-Agent/issues
- "numpy.typing" 또는 "AttributeError" 검색

### 대안:
RD-Agent는 복잡한 설정이 필요하므로, **ValueInvestmentCrew (Tool-based)**를 먼저 구현하는 것을 권장합니다:

- ✅ Docker 불필요
- ✅ RAG 불필요 (Embedding Model 불필요)
- ✅ Chat Model만 필요
- ✅ 비용 저렴 ($0/month with DeepSeek R1)
- ✅ 3-4주면 프로토타입 완성

```bash
cd ~/fin_agents
mkdir ValueInvestmentCrew
cd ValueInvestmentCrew
# ValueInvestmentCrew_Design.md 참고하여 개발 시작
```
