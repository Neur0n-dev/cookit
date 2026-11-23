# 05. 개발 환경

### 1. 저장소

| 구분 | 내용 |
|---|---|
| 방식 | 새 저장소 1개에 백엔드·프론트·인프라·문서를 모두 관리 (모노레포) |
| 이름 | `cookit` (`github.com/Neur0n-dev/cookit`, 이미지 `ghcr.io/neur0n-dev/cookit/backend`) |
| 공개 여부 | 공개 (포트폴리오용), 비밀 정보는 GitHub Secrets·Ansible Vault로 분리 |

### 2. 저장소 구조

```
cookit/
├── backend/                  Spring Boot (Gradle)
│   ├── src/test/             단위·통합 테스트 (JUnit, CI에서 실행)
│   └── Dockerfile            백엔드 이미지 빌드
├── frontend/                 React + Vite (정적 파일로 빌드, LB Nginx에 배포)
├── infra/
│   ├── ansible/
│   │   ├── inventory/        서버 목록 (1단계·3단계)
│   │   ├── group_vars/       서버 그룹별 설정, Vault 암호화 파일
│   │   ├── roles/            common, nginx, certbot, keepalived, docker,
│   │   │                     mariadb, galera(garbd 포함), haproxy, redis,
│   │   │                     monitoring(exporter 포함), alloy, backup, runner
│   │   └── playbooks/        서버 구성·배포 실행 파일
│   └── docker/               WAS·모니터링 서버용 compose 파일
├── load-test/                k6 부하 테스트 스크립트
├── docs/                     00~10 단계 문서, images/
├── .github/workflows/        CI·배포
├── docker-compose.local.yml  로컬 개발용 MariaDB·Redis
├── .env.example              필요한 설정 항목 안내 (값은 비움)
├── .gitignore                .env 등 비밀 정보 파일 커밋 방지
└── README.md
```

※ Nginx·HAProxy·Prometheus 등 설정 파일은 Ansible role 안의 템플릿으로 관리

### 3. 로컬 개발 도구

| 구분 | 도구 | 용도 |
|---|---|---|
| 백엔드 | JDK 21, Gradle Wrapper | Spring Boot 빌드·실행 (저장소의 `gradlew` 사용, Gradle 별도 설치 없음) |
| 프론트 | Node.js LTS, npm | React 빌드·개발 서버 |
| IDE | IntelliJ IDEA | 백엔드·프론트 개발, DB 접속(Database 기능) |
| 형상 관리 | Git | 소스 관리, GitHub 연동 |
| 컨테이너 | Colima + Docker CLI | 로컬 MariaDB·Redis 실행, 이미지 빌드 확인 (GUI 없이 명령어로만 사용, `colima start` 후 `docker` 명령 사용) |
| 테스트 | k6 | 부하 테스트 |

**비고**

1. 세부 버전은 구축 시점의 최신 안정 버전으로 확정해 README에 기록
2. Ansible은 개발 PC가 아닌 모니터링 서버에서 실행 (6. 참고)

### 4. 로컬 실행 방식

| 구분 | 내용 |
|---|---|
| DB·Redis | `docker compose -f docker-compose.local.yml up`으로 MariaDB 11.4·Redis 실행 (운영과 같은 버전) |
| 백엔드 | IntelliJ에서 `dev` 프로필로 실행 (`localhost:8080`) |
| 프론트 | Vite 개발 서버 실행, `/api` 요청은 `localhost:8080`으로 전달 (운영의 Nginx 역할) |
| 설정값 | 프로필별 분리 (`dev` 개발 PC, `prod` 운영 서버), 비밀 정보는 환경 변수로만 입력, `.env.example`로 필요한 항목 안내 |
| Gemini | `dev` 프로필은 가짜(mock) 응답 사용 → API 키 없이 개발, 사용량 절약. 실제 호출은 설정으로 전환 |

※ 부하 테스트(k6) 때도 Gemini mock 사용 (실제 호출 시 사용량 한도 초과·비용 발생)

### 5. 브랜치·커밋 규칙

| 구분 | 내용 |
|---|---|
| 브랜치 | `main`(배포 기준) + 작업별 `feature/*`, `fix/*`, `infra/*`, `docs/*` |
| 병합 | Pull Request로만 `main`에 병합, `main` 직접 push 금지 |
| 브랜치 보호 | GitHub 브랜치 보호 규칙: PR 필수, CI(테스트·빌드) 통과 필수, 리뷰 승인 0명 (1인 프로젝트라 본인 PR 승인 불가) |
| 병합 방식 | Squash merge (PR 하나를 커밋 1개로 합쳐 병합), 병합 후 작업 브랜치 자동 삭제 → `main` 커밋 1개 = 배포 1번 = 이미지 태그 1개 |
| 배포 | `main` 병합 시 자동 배포 (04 상세 설계 > 12. 배포 기준) |
| 커밋 메시지 | 앞에 종류 표시: `feat` 기능, `fix` 수정, `refactor` 기능 변화 없는 코드 정리, `test` 테스트 코드, `infra` 인프라, `docs` 문서, `chore` 기타 |

### 6. 관리자 PC·Ansible 실행 환경

| 구분 | 설정 |
|---|---|
| 관리자 PC | SSH 키 생성 후 전체 서버에 공개 키 등록, `~/.ssh/config`에 서버별 별칭 등록 |
| 관리자 PC | 내부망에서 Grafana(3000) 접속 확인 |
| 모니터링 서버 | Ansible 설치, 저장소 내려받기(clone) |
| 모니터링 서버 | Ansible용 SSH 키 생성 후 전체 서버에 공개 키 등록 |
| 모니터링 서버 | Ansible Vault 비밀번호 파일은 저장소 밖에 보관 (권한 600, 본인만 읽기) |
| 관리자 | Vault 비밀번호는 개인 비밀번호 관리 앱(예: 1Password, Bitwarden)에 별도 보관 (서버 장애 대비) |


---

이전 문서: [04. 상세 설계](04-detailed-design.md) · 다음 문서: [06. 인프라 구축](06-infra-build.md)
