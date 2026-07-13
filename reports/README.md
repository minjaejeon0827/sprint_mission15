# 스프린트 미션 15 — 두 연구자 협업 도커 워크플로우

학생 성적 데이터로 회귀 모델을 학습(연구자 1)하고, 그 결과물을 도커로 전달받아
추론(연구자 2)하는 전체 파이프라인을 도커 이미지 · docker-compose 로 구성한 프로젝트.

## 폴더 구조

```
V2. 최종 버전

mission-result/
├── reports/
|   └── 보고서.pdf     # 보고서·발표자료 PDF
|
|   # 제출물 1: 코드 폴더
├── researcher1/  # 연구자 1
|   ├── src/
|   |   ├── data/     # mission15_train.csv, mission15_test.csv
|   |   ├── eda_modeling.ipynb  # 전처리·EDA·모델링 노트북
|   |   └── train.py
|   |
│   └── .dockerignore · Dockerfile · requirements.txt
│
├── researcher2/  # 연구자 2
|   ├── src/
|   |   └── work/inference.ipynb  # 추론 노트북
│   └── .dockerignore · Dockerfile · requirements.txt  # Jupyter 이미지
│
└── .dockerignore · docker-compose.yml · .env · README.md
```

```
V1. 초기 버전

mission-result/
├── researcher1/                 # 연구자 1 : 전처리·EDA·모델링·모델 저장
│   ├── Dockerfile               #   전처리→모델링→model.pkl 자동화 이미지
│   ├── requirements.txt         #   고정 버전 (연구자 2와 동일)
│   ├── train.py                 #   .ipynb 를 옮긴 자동화 스크립트 (ENTRYPOINT)
│   ├── eda_modeling.ipynb       #   전처리·EDA·모델링 노트북
│   └── data/
│       ├── mission15_train.csv
│       └── mission15_test.csv
├── researcher2/                 # 연구자 2 : 추론 Jupyter 환경
│   ├── Dockerfile               #   Jupyter Notebook 이미지 (버전 통일)
│   ├── requirements.txt         #   연구자 1과 동일 버전 + jupyterlab
│   └── work/
│       └── inference.ipynb      #   추론 노트북 (result.csv 생성)
├── docker-compose.yml           # 두 이미지 오케스트레이션
├── .env                         # Docker Hub 사용자명 · 태그 · 토큰
└── README.md
```

## 사전 준비

`.env` 파일의 `DOCKERHUB_USERNAME` 을 본인 Docker Hub 아이디로 수정한다.

---

## STEP 1 — 연구자 1 : 이미지 빌드 & Docker Hub 업로드

```bash
cd researcher1

# 이미지 빌드
docker build -t <DOCKERHUB_ID>/mission15-trainer:v1.0.0 .

# (선택) 로컬에서 동작 확인 — 공유 볼륨에 model.pkl / test.csv 생성 확인
docker run --rm -v shared-data:/app/output <DOCKERHUB_ID>/mission15-trainer:v1.0.0

# Docker Hub 로그인 후 push
docker login
docker push <DOCKERHUB_ID>/mission15-trainer:v1.0.0
```

실행하면 `train.py` 가 학습 후 검증 RMSE 를 출력하고,
`model.pkl` 과 `mission15_test.csv` 를 컨테이너의 `/app/output` (공유 볼륨)에 저장한다.

---

## STEP 2 — 연구자 2 : docker-compose 로 추론

연구자 2는 데이터·모델을 갖고 있지 않다. Docker Hub 이미지만 받아서 실행한다.

```bash
cd ..                       # mission-result/ 로 이동

# 연구자 1 이미지를 Docker Hub 에서 받아오기 (없으면 build 로 대체됨)
docker compose pull trainer

# 전체 스택 실행 (trainer 실행→종료 후 jupyter 기동)
docker compose up --build
```

동작 순서
1. **trainer** 컨테이너가 실행되어 `model.pkl` · `test.csv` 를 공유 볼륨 `shared-data` 에 기록하고 정상 종료한다.
2. `depends_on: condition: service_completed_successfully` 조건에 따라 **jupyter** 컨테이너가 기동한다.
3. jupyter 컨테이너는 같은 볼륨을 `/home/jovyan/shared` 로 마운트하여 두 파일을 전달받는다.

브라우저에서 접속:

```
http://localhost:8888/?token=mission15
```

`work/inference.ipynb` 를 열고 위에서부터 실행하면
`/home/jovyan/shared` 의 `model.pkl` · `test.csv` 로 추론하여
`work/result.csv` (호스트의 `researcher2/src/work/result.csv`)로 결과가 저장된다.

---

## 파일 전달 전략 (참고사항 2)

연구자 1 컨테이너의 `model.pkl` · `test.csv` 를 연구자 2 컨테이너로 넘기는 방법은 두 가지다.

### 전략 A (기본·자동) — 공유 Named Volume
`docker-compose.yml` 이 사용하는 방식. `shared-data` 볼륨을 두 컨테이너가 함께 마운트한다.
- trainer: `shared-data → /app/output` (파일 쓰기)
- jupyter: `shared-data → /home/jovyan/shared` (파일 읽기)

컨테이너가 삭제돼도 볼륨은 남아 재현성이 높고, 사람이 개입할 필요가 없다.

### 전략 B (수동·보완) — 호스트를 매개로 `docker cp`
공유 볼륨을 쓰지 않고, 컨테이너 → 호스트 → 컨테이너 순으로 직접 복사한다.

```bash
# 1) trainer 를 1회 실행 (종료된 컨테이너로 남겨 docker cp 대상이 되게 함)
docker run --name mission15-trainer <DOCKERHUB_ID>/mission15-trainer:v1.0.0

# 2) 컨테이너 → 호스트 로 파일 꺼내기
docker cp mission15-trainer:/app/output/model.pkl            ./researcher2/src/work/model.pkl
docker cp mission15-trainer:/app/output/mission15_test.csv   ./researcher2/src/work/mission15_test.csv

# 3) 호스트 → jupyter 컨테이너 로 넣기 (jupyter 가 실행 중일 때)
docker cp ./researcher2/src/work/model.pkl          mission15-jupyter:/home/jovyan/shared/model.pkl
docker cp ./researcher2/src/work/mission15_test.csv mission15-jupyter:/home/jovyan/shared/mission15_test.csv
```

`docker cp` 는 정지된 컨테이너에도 동작하므로 배치성 trainer 와 잘 맞는다.

---

## 버전 통일 전략 (참고사항 1)

- 두 이미지 모두 **`python:3.11-slim`** 베이스로 파이썬 버전을 맞춘다.
- 두 `requirements.txt` 의 `scikit-learn / pandas / numpy / joblib` **버전 문자열을 완전히 동일**하게 고정한다.
- 특히 `scikit-learn` 버전이 다르면 `model.pkl` 로드 시
  `InconsistentVersionWarning` 또는 로드 실패가 발생하므로, 동일 버전 고정이 핵심이다.

---

## 정리

```bash
docker compose down          # 컨테이너 정지·삭제 (볼륨 유지)
docker compose down -v       # 공유 볼륨까지 삭제
```
