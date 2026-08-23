# infra-engineering-lab

SM, IT 인프라, DB 시스템 관리 직무를 준비하며 학습한 내용을 실습 중심으로 기록하는 저장소입니다.

Linux, Network, Database, Cloud, Monitoring, Troubleshooting, Automation을 공부하면서 사용한 명령어와 설정, 문제 해결 과정, 프로젝트 결과를 정리합니다.

## 학습 목표

- 운영체제와 네트워크의 기본 원리를 이해하고 직접 검증합니다.
- 서버와 데이터베이스를 구축·운영하며 관리 역량을 기릅니다.
- 장애의 증상, 원인, 해결 과정과 재발 방지 방법을 기록합니다.
- 반복 작업을 스크립트와 도구로 자동화합니다.
- 작은 실습을 실제 운영 환경을 가정한 프로젝트로 발전시킵니다.

## 현재 구조

```text
infra-engineering-lab/
├── README.md
├── 01-linux/
│   └── README.md
├── 10-troubleshooting/
│   └── README.md
└── 11-projects/
    └── README.md
```

| 경로 | 기록 내용 |
| --- | --- |
| `01-linux/` | Linux 기본 개념, 명령어, 권한, 프로세스, 서비스, 디스크, 로그, SSH 실습 |
| `10-troubleshooting/` | 장애 증상, 점검 과정, 원인, 해결, 검증, 재발 방지 기록 |
| `11-projects/` | 여러 학습 주제를 결합한 구축·운영 프로젝트 |

## 기록 원칙

단순한 개념 요약보다 직접 실행하고 확인한 내용을 중심으로 작성합니다.

1. 학습 또는 실습 목표
2. 환경과 사전 조건
3. 사용한 명령어와 설정
4. 실행 결과와 검증 방법
5. 문제, 원인, 해결 과정
6. 배운 점과 다음 과제

민감한 정보는 저장하지 않습니다. 비밀번호, API 키, 실제 IP 주소 등은 예시 값이나 환경 변수로 대체합니다.

## 기록 방법
- 공부 시작 시: git pull
git pull origin main

- 공부 종료 시: git add → git commit → git push
git add .
git commit -m "docs: add Linux permission practice"
git push origin main


## 향후 확장 계획

학습 진도에 따라 필요한 영역만 단계적으로 추가합니다.

```text
infra-engineering-lab/
├── 01-linux/
├── 02-network/
├── 03-database/
├── 04-web-infrastructure/
├── 05-container/
├── 06-monitoring/
├── 07-cloud/
├── 08-automation/
├── 09-security/
├── 10-troubleshooting/
└── 11-projects/
```

- **Network**: TCP/IP, 서브넷, 라우팅, DNS, HTTP/HTTPS, 패킷 분석
- **Database**: MySQL·Oracle 구조, 권한, 인덱스, 트랜잭션, 백업과 복구, 모니터링
- **Web Infrastructure**: Nginx, 리버스 프록시, TLS, 서비스 운영
- **Container**: Docker와 Docker Compose 기반 실행 환경
- **Monitoring**: 시스템·DB 지표, 로그, Prometheus, Grafana
- **Cloud**: AWS IAM, VPC, EC2, S3, RDS, CloudWatch
- **Automation**: Bash, Python, Ansible을 이용한 반복 작업 자동화
- **Security**: Linux, SSH, 방화벽, 데이터베이스 보안 설정

## 진행 방식

학습 기록 → 실습 기록 → 장애 대응 기록 → 작은 프로젝트 → 통합 인프라 프로젝트 순서로 발전시킵니다.
