# 03. 기술 스택

> 02 아키텍처 구성도 순서(서버 기반 → WEB → APP → DB → 운영)로 정리

## 서버 기반

| 구분 | 선택 | 역할 | 선정 이유 |
|---|---|---|---|
| OS | CentOS Stream 9 | 전체 서버 운영체제 | RHEL 계열 무료 OS로 실무 RHEL 환경과 동일한 운영 경험 |
| 실행 방식 | VM + Docker | WAS(Spring Boot·HAProxy)와 모니터링 서버(모니터링 도구·Redis)는 컨테이너로 실행, LB·DB는 VM에 직접 설치 | 앱은 컨테이너로 손쉽게 교체, LB·DB는 직접 설치로 실무와 동일하게 운영 |
| 방화벽 | firewalld | 서버별 허용 포트 관리 | CentOS 기본 방화벽, Ansible로 일괄 설정 |
| 시간 동기화 | chrony | 전체 서버 시간을 표준 시간(NTP)에 맞춤 | CentOS 기본 포함, 로그·지표 시간 일치에 필요 |

## 웹·로드밸런서

| 구분 | 선택 | 역할 | 선정 이유 |
|---|---|---|---|
| 도메인 | cookit.neurondev.co.kr (가비아) | 서비스 접속 주소 | 보유한 도메인 사용 |
| LB | Nginx | 화면 파일 제공, /api 요청을 WAS로 분배, HTTPS 처리 | 정적 파일 서빙과 API 분배를 한 곳에서 처리 |
| 인증서 (2단계) | Let's Encrypt + certbot | 무료 인증서 발급·자동 갱신 | 무료 정식 인증서, 90일마다 자동 갱신 |
| LB 이중화 (3단계) | keepalived (VRRP) | LB 2대가 서로 생존 확인, 장애 시 VIP 자동 이동 | 리눅스 표준 이중화 도구, 설정 단순 |

## 애플리케이션

| 구분 | 선택 | 역할 | 선정 이유 |
|---|---|---|---|
| 프론트 | React + Vite (SPA) | 사용자 화면 (브라우저에서 실행) | 빌드 결과를 Nginx가 직접 서빙, 서버 추가 불필요 |
| 백엔드 | Java 21 + Spring Boot 4.2 | API 서버 | 국내 실무 표준, Actuator로 헬스체크·Prometheus 지표 기본 제공 |
| 인증 | JWT | 로그인 정보를 토큰으로 발급, 서버에 세션 저장 안 함 | 세션 저장소 불필요, WAS 여러 대에서 그대로 동작 |
| 외부 API | Google Gemini | 재료 기반 레시피 생성 (LLM) | 무료 사용량 제공, JSON 형식 응답 지원으로 레시피 데이터 저장 용이 |

## 데이터

| 구분 | 선택 | 역할 | 선정 이유 |
|---|---|---|---|
| 캐시 | Redis | 추천 결과 캐시, 추천 요청 횟수 제한, 인기 레시피 랭킹 | 메모리 기반 빠른 응답, WAS 여러 대가 공유 |
| DB 연결 | HAProxy | WAS와 DB 사이 중계, 정상 DB로만 연결 (장애 시 자동 전환) | DB 헬스체크 기능 강력, 앱 설정 변경 없이 DB 추가·전환 |
| DB | MariaDB 11.4 LTS | 회원·재료·즐겨찾기·검색 기록 저장 | 무료, MySQL 호환, Galera 내장 |
| DB 이중화 (3단계) | Galera Cluster | DB 2대 간 실시간 동기 복제 | 데이터 유실 없는 이중화, 쓰기는 1대로 운영(A-S)해 충돌 방지 |
| DB 투표 (3단계) | garbd | 데이터 없이 투표만 하는 3번째 노드, DB 1대 장애 시 과반 유지 | DB 3대 대신 2대 + garbd로 서버 절약 |
| 백업 | mariabackup | DB 전체 백업 (서비스 중단 없이) | MariaDB 공식 도구, 운영 중 백업 가능 |

## 모니터링·알림

| 구분 | 선택 | 역할 | 선정 이유 |
|---|---|---|---|
| 지표 수집 | Prometheus | 서버·앱 지표(CPU, 메모리, 응답시간 등) 주기적 수집·저장 | 무료, 실무 표준 |
| 지표 제공 | exporter (node·nginx·mysqld·redis) | 각 서버·프로그램의 지표를 Prometheus가 읽을 수 있게 제공 | Prometheus 공식·표준 exporter (mysqld_exporter는 MariaDB 지표 수집에도 사용) |
| 접속 확인 | blackbox_exporter | 외부 사용자처럼 서비스에 접속해 응답·인증서 만료일 확인 | 가용성(%) 측정 기준 |
| 로그 수집 | Grafana Alloy | 각 서버 로그를 Loki로 전송 (로그 수집만 사용) | Grafana 공식 수집 도구, 지원 종료된 Promtail의 후속 |
| 로그 저장 | Loki | 전체 서버 로그를 한곳에 모아 검색 | Grafana와 연동, 가벼움 |
| 대시보드 | Grafana | 수집한 지표·로그를 그래프로 표시 | 무료, 대시보드 구성 쉬움 |
| 알림 | Alertmanager | 장애 조건 시 알림 발송 (중복 묶음·등급 분리) | Prometheus 공식 알림 도구 |
| 알림 채널 | Slack | critical·warning 모두 Slack으로 수신 | 무료, Alertmanager 기본 지원, 실무에서 널리 사용 |
| 생존 신호 확인 | healthchecks.io | 모니터링 서버가 보내는 신호가 끊기면 Slack 알림 | 무료, 모니터링 서버 자체 장애 감지 |

## 자동화·배포

| 구분 | 선택 | 역할 | 선정 이유 |
|---|---|---|---|
| 서버 자동화 | Ansible | 서버 설치·설정을 코드로 관리 | 설정을 코드로 관리, 재구축 시 동일 상태 복구 |
| CI/CD | GitHub Actions | 코드 반영 시 빌드·테스트·배포 자동 실행 | 코드와 같은 곳에서 관리, 무료 |
| 이미지 저장소 | GHCR | 빌드된 Docker 이미지 보관 | GitHub 통합, 공개 이미지 무료, WAS에서 로그인 없이 내려받기 |
| 배포 실행 | Self-hosted Runner | 내부 서버에서 GitHub Actions 작업 실행 (내부 VM에 배포) | GitHub 서버는 내부망 접근 불가, 나가는 연결(443)만으로 동작 |

## 테스트

| 구분 | 선택 | 역할 | 선정 이유 |
|---|---|---|---|
| 단위·통합 테스트 | JUnit 5, Testcontainers | 코드 단위 검사, 실제 MariaDB·Redis 컨테이너로 통합 검사 (CI에서 자동 실행) | Spring Boot 기본 지원, 운영과 같은 DB로 검사 |
| 부하 테스트 | k6 | 가상 사용자로 요청을 보내 성능·무중단 배포 측정 | 무료, 스크립트 작성 쉬움 |

---

이전 문서: [02. 아키텍처 설계](02-architecture.md) · 다음 문서: [04. 상세 설계](04-detailed-design.md)
