# CareerPass — Backend AI

> 대학생 취업 준비를 돕는 AI 코칭 플랫폼 **CareerPass**의 AI 마이크로서비스. OpenAI API를 활용해 음성 STT, 면접 질문 생성, 면접 답변 채점, 자기소개서 피드백/재생성을 담당하고 Spring 백엔드에 결과를 반환합니다.

## 기술 스택

![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=for-the-badge&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white)
![OpenAI](https://img.shields.io/badge/OpenAI%20API-412991?style=for-the-badge&logo=openai&logoColor=white)
![Whisper](https://img.shields.io/badge/Whisper-STT-412991?style=for-the-badge&logo=openai&logoColor=white)
![Pydantic](https://img.shields.io/badge/Pydantic-E92063?style=for-the-badge&logo=pydantic&logoColor=white)
![Gunicorn](https://img.shields.io/badge/Gunicorn%20%2B%20Uvicorn-499848?style=for-the-badge&logo=gunicorn&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)

## 주요 기능

- **음성 STT** (`routers/voice_ai.py`, `/voice/analyze`) — 면접 답변 음성 파일(m4a/mp3/wav/webm/ogg)을 받아 OpenAI **Whisper-1**으로 한국어 텍스트 변환
- **면접 질문 생성** (`routers/question_ai.py`, `/question/api/questions`) — 전공·직무·자소서를 프롬프트에 반영해 GPT-3.5-turbo로 맞춤 면접 질문 생성 (자소서 유무에 따라 질문 구성을 동적으로 분기)
- **면접 답변 AI 채점** (`routers/interview_ai.py`, `/interview/analysis/interview/run`) — STT 결과/자소서/질문을 받아 파인튜닝 모델을 호출, 점수·유창성·내용 깊이·구조·필러 단어 수·강점/개선점/리스크를 **Pydantic 스키마로 강제 검증**해 반환
- **자기소개서 AI 피드백 & 재생성** (`routers/resume_edit.py`, `/resume/resume/feedback`) — 약 80줄짜리 시스템 프롬프트로 STAR 기법·정량적 성과·ATS 최적화 관점의 피드백을 생성하고, 원본 재생성본 + 토스 인재상 맞춤 재생성본까지 3종 결과를 동시에 반환
- **헬스체크 & CORS 중앙 설정** (`main_api.py`) — 4개 라우터를 prefix로 통합하는 단일 FastAPI 앱

## 내가 맡은 역할 / 기여한 부분

CareerPass 팀 프로젝트에서 **프론트엔드를 주도하면서, 이 AI 서버를 Spring 백엔드와 연동하는 작업을 팀원들과 함께 진행**했습니다.

- 음성 녹음 업로드(FormData) → Spring 백엔드 → 이 서버의 `/voice/analyze`로 이어지는 멀티파트 전달 흐름을 프론트 쪽 요청 형식 기준으로 점검·조정
- `/interview/analysis/interview/run`, `/resume/resume/feedback` 등 각 엔드포인트의 응답 스키마(`AnswerAnalysisResult`, `FeedbackResponse`)를 프론트에서 그대로 렌더링할 수 있는 형태로 맞추기 위해 필드명/타입을 팀원과 함께 조율
- *(이 레포는 PR을 통해 머지되는 구조라 git 커밋 작성자는 마지막 머지/수정자 기준으로 남아 있으나, 엔드포인트 응답 형식 설계와 프론트–AI 서버 간 연동 검증은 공동 작업)*

## 기술적으로 어려웠던 점과 해결 방법

### 1. LLM이 스키마를 벗어난 JSON을 반환하는 문제
면접 채점(`interview_ai.py`)은 `score`, `fluency`(0~5), `structure` 등 정해진 범위의 숫자와 리스트를 받아야 하는데, LLM이 가끔 스키마를 어기는 형태의 JSON을 반환해 후속 로직이 깨지는 문제가 있었습니다.
→ `response_format={"type": "json_object"}`로 JSON 모드를 강제하고, `AnswerAnalysisResult.model_validate_json()`으로 Pydantic 검증을 거치도록 했습니다. 검증 실패 시(`ValidationError`) 원본 LLM 출력을 로그에 남기고 500 에러로 명확히 알리도록 처리해(`interview_ai.py:139-142`), 잘못된 데이터가 그대로 백엔드로 전달되는 것을 막았습니다.

### 2. OpenAI 클라이언트 초기화 시점 문제로 인한 API Key 누락 오류
초기에는 모듈 로드 시점에 전역으로 OpenAI 클라이언트를 생성했는데, 환경변수가 로드되기 전에 모듈이 import되면서 키가 비어 있는 상태로 클라이언트가 만들어지는 타이밍 이슈가 있었습니다.
→ `voice_ai.py`에서는 요청이 들어온 시점(`analyze` 함수 내부)에 `os.environ.get("QUESTION_VOICE_OPENAI_KEY")`로 키를 다시 읽어 클라이언트를 지연 생성하도록 변경해 문제를 해결했습니다(코드 주석에 `[핵심 해결] 지연 초기화`로 명시).

### 3. 3개의 서로 다른 OpenAI 키를 라우터별로 분리 관리
면접/질문/자소서/음성 4개 기능이 각각 다른 용도의 OpenAI 키(`INTERVIEW_OPENAI_KEY`, `QUESTION_VOICE_OPENAI_KEY`, `RESUME_OPENAI_KEY`)와 파인튜닝 모델 ID(`INTERVIEW_FINEDTUNED_MODEL_ID`)를 쓰도록 설계되어, 키가 누락되면 어느 기능이 죽었는지 추적하기 어려웠습니다.
→ 각 라우터가 자신의 키가 없을 때 즉시 "Mock 모드" 또는 명확한 500 에러로 빠지게 하고(`question_ai.py`, `resume_edit.py`), 로그에 어떤 라우터의 초기화가 실패했는지 출력하도록 해 운영 중 장애 지점을 빠르게 좁힐 수 있게 했습니다.

### 4. 한국어 자소서 텍스트의 특수문자/공백 정규화
사용자가 입력한 자소서에 스마트 쿼트(” ’), zero-width space, 탭/개행이 섞여 들어오면 프롬프트 토큰 낭비 및 LLM 출력 품질 저하로 이어졌습니다.
→ `resume_edit.py`의 Pydantic `validator`에서 스마트 쿼트를 일반 쿼트로 치환하고 `​`, `\xa0`, 개행·탭을 공백으로 정규화한 뒤 다중 공백을 하나로 합치는 전처리를 입력 단계에서 강제했습니다.

## 설치 및 실행 방법

### 사전 요구사항
- Python 3.11
- OpenAI API Key (기능별로 최대 3개: 면접/질문·음성/자소서)

### 환경 변수
프로젝트 루트에 `app_sevice.env` 파일을 생성합니다.

```
SERVICE_HOST=0.0.0.0
SERVICE_PORT=8000

INTERVIEW_OPENAI_KEY=<면접 채점용 OpenAI Key>
INTERVIEW_FINEDTUNED_MODEL_ID=<파인튜닝 모델 ID>
QUESTION_VOICE_OPENAI_KEY=<질문 생성 / STT용 OpenAI Key>
RESUME_OPENAI_KEY=<자소서 피드백용 OpenAI Key>
```

### 로컬 실행

```bash
pip install -r requirements.txt
python main_api.py
```

서버는 기본적으로 `http://0.0.0.0:8000`에서 기동되며, Swagger 문서는 `http://localhost:8000/docs`에서 확인할 수 있습니다.

### Docker 실행

```bash
docker build -t careerpass-ai .
docker run -p 8000:8000 --env-file app_sevice.env careerpass-ai
```

컨테이너는 `gunicorn` + `UvicornWorker` 4개 워커로 구동됩니다.
