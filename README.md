# Melting Tank KPI MLOps Serving (v3)

v2 학습 파이프라인이 만든 모델 번들로 **현재 1분의 센서값 10개를 입력받아 다음 1분의 NG 발생 여부**를 예측하는 별도 서빙 저장소입니다.

## 프로젝트 범위

- FastAPI 추론 API와 OpenAPI 문서
- v2 모델·스케일러·메타데이터 계약 검증
- KPI 승인 모델만 허용하는 readiness/serving gate
- 예측 결과 SQLite 저장 및 간단한 운영 대시보드
- API Docker 이미지와 Compose 실행

학습, 임계값 재탐색, MLflow Tracking Server는 modeling 저장소(v2)의 책임이며 이 저장소에는 포함하지 않습니다.

## 구조

```text
app/             FastAPI, 모델 로더, SQLite, 대시보드
artifacts/       v2 MLflow Run에서 받은 모델 번들(커밋 제외)
runtime-data/    예측 SQLite 파일(커밋 제외)
scripts/         API 통합 확인 요청
tests/           API·게이트·저장소 단위 테스트
docs/            v2-v3 연결과 실행 절차
```

## 빠른 실행

1. v2의 승인된 `model_bundle` 파일 4개를 `artifacts/`에 둡니다.
2. v2에서 사용한 Python 3.11/3.12 가상환경을 활성화합니다.
3. 의존성과 프로젝트를 설치하고 테스트합니다.

```bash
python -m pip install -e ".[dev]"
python -m pytest
python -m uvicorn app.main:app --reload
```

브라우저에서 다음을 확인합니다.

- `http://127.0.0.1:8000/healthz`: 프로세스 생존 여부
- `http://127.0.0.1:8000/readyz`: 모델 로드·배포 승인 여부
- `http://127.0.0.1:8000/status`: 차단 사유와 모델 계약
- `http://127.0.0.1:8000/docs`: API 문서 및 직접 요청
- `http://127.0.0.1:8000/dashboard`: 최근 예측 대시보드

## 미승인 v2 baseline 확인

현재 baseline은 KPI 미달이므로 기본 설정에서 `/readyz`와 `/predict`가 `503`을 반환하는 것이 정상입니다. 수업용 통합 검증에 한해 `ALLOW_UNAPPROVED_MODEL=true`로 실행할 수 있으며, 응답에는 미승인 및 개발 예외 상태가 함께 표시됩니다. 실제 배포에는 이 옵션을 사용하지 않습니다.

```bash
python scripts/smoke_request.py
```

## 주요 변경사항
- S3 승인 모델 번들 동기화 및 게시 로직 추가
- ECR, ECS Fargate, ALB, IAM 구성을 위한 CloudFormation 템플릿 추가
- GitHub Actions OIDC 기반 CI/CD 워크플로 구성
- Ruff, pytest, Docker 이미지 빌드 자동 검증 추가
- ECS 배포 후 `/healthz`, `/readyz` 상태 확인 구성
- v4 브랜치 전략과 AWS 배포 절차 문서화
## 검증 결과
- `python -m ruff check .` 통과
- `python -m pytest` 통과
- GitHub Actions에서 Docker 이미지 빌드 검증 구성
- CloudFormation 템플릿 구문 검증
## 참고사항
- 운영 AWS 배포에서는 `ALLOW_UNAPPROVED_MODEL=false`를 유지합니다.
- KPI 승인 모델 번들이 S3에 등록되어야 `/readyz`가 정상 응답합니다.
- AWS Access Key를 저장하지 않고 GitHub Actions OIDC 인증을 사용합니다.


세부 내용은 [v2-v3 연결](docs/01_v2_v3_연결.md)과 [실행 절차](docs/02_실행_절차.md)를 참고합니다.
