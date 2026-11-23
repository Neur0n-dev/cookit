# 06. 인프라 구축

## 구축 준비

### 1. 시작 전 확인 사항

| 구분 | 확인 내용 | 필요 이유 | 적용 |
|---|---|---|---|
| 공인 IP·NAT | 공인 IP 할당, 80 포트 포워딩(NAT) 허용 (2단계부터 443 추가) | 외부에서 서비스 접속 | 1단계 |
| 원래 IP 유지 | NAT를 거쳐도 사용자 원래 IP가 LB까지 전달되는지 | IP별 추천 횟수 제한 | 1단계 |
| NAT 헤어핀 | 내부망에서 도메인으로 접속 가능한지 (내부 → 공인 IP → 다시 내부) | 내부망에서 서비스 확인 | 1단계 |
| 나가는 연결 | 서버 → 외부 443·53·123 허용, 프록시 여부 | 패키지·이미지 설치, Gemini, 알림, 시간 동기화 | 1단계 |
| 호스트 자원 | 여유 용량(최대 1.4TB), 씬 프로비저닝(실제 사용한 만큼만 디스크 차지) 가능 여부, 하이퍼바이저 종류 | VM 생성 | 1단계 |
| 내부 IP | IP 8개 (서버 7개 + VIP 1개), 모두 같은 네트워크 대역 | VIP는 같은 대역 안에서만 이동 가능 | 1단계 |
| VIP 허용 | 가상 스위치가 VM의 추가 IP(VIP) 통신을 막지 않는지 | 일부 가상화 환경은 등록되지 않은 IP 차단 | 3단계 |
| DNS | 가비아에 `cookit.neurondev.co.kr` A 레코드 → 공인 IP 등록 | 도메인 접속, 2단계 인증서 발급 | 1단계 |

※ 1단계 시작 전에 모두 확인 (3단계용 IP·용량도 미리 확보)

### 2. 구축 원칙

1. VM 템플릿 생성·복제, IP·호스트명 설정까지만 수동, 이후 설정은 모두 Ansible로 실행
2. 모니터링 서버를 가장 먼저 구성 (Ansible 실행 서버, 이후 작업 결과를 바로 Grafana로 확인)
3. playbook은 여러 번 실행해도 결과가 같게 작성 (멱등성), 서버에서 직접 수정 금지
4. 단계 시작 전 VM 스냅샷 생성 → 문제 시 이전 상태로 복구
5. 작업마다 완료 확인 후 다음 작업 진행
6. SELinux(리눅스 보안 기능)는 끄지 않고 필요한 허용만 추가 (예: Nginx → WAS 전달 허용)

## 1단계 · 단일 구성 (VM 4대, HTTP)

### 3. 1단계 구축 순서

| 순서 | 작업 | 대상 (role) | 완료 확인 |
|---|---|---|---|
| 1 | VM 템플릿 생성: CentOS Stream 9 최소 설치, 관리자 계정, 관리자 PC 공개 키 등록 | 템플릿 (수동) | 템플릿 VM SSH 접속 |
| 2 | VM 4대 복제: IP·호스트명 설정 (`cookit-lb01`, `cookit-was01`, `cookit-db01`, `cookit-mon01`), 모니터링 서버만 메모리 8GB | 전체 (수동) | 관리자 PC에서 4대 SSH 접속 |
| 3 | Ansible 준비: Ansible 설치, 저장소 내려받기, SSH 키 생성·배포, inventory·Vault 파일 작성 | `mon01` (수동) | `ansible all -m ping` 성공 |
| 4 | 공통 설정: 시간대·시간 동기화(chrony), SSH 보안(비밀번호·root 로그인 차단), firewalld, 로그 정리(logrotate), 기본 패키지. WAS·LB에 배포 전용 계정(`deploy`) 생성 | 전체 (`common`) | 허용 포트 외 접속 차단 확인 |
| 5 | 모니터링 서버: Docker, Prometheus·Grafana·Alertmanager·Loki·blackbox_exporter, Redis·redis_exporter | `mon01` (`docker`, `monitoring`, `redis`) | Grafana 접속, Redis 응답 |
| 6 | 지표·로그 수집: node_exporter, Alloy | 전체 (`monitoring`, `alloy`) | Grafana에서 4대 지표·로그 확인 |
| 7 | DB: MariaDB 11.4 설치, 문자셋(utf8mb4)·메모리 설정, DB·용도별 계정 4개, mysqld_exporter | `db01` (`mariadb`) | 앱 계정 접속, DB 지표 수집 |
| 8 | WAS: Docker, HAProxy(DB 연결 전달), 앱 실행용 compose 파일 | `was01` (`docker`, `haproxy`) | `localhost:3306`으로 DB 접속, HAProxy 통계(8404) 확인 |
| 9 | LB: Nginx(프론트 폴더 구조, `/api` 전달, 보안 헤더, 요청 제한), nginx_exporter | `lb01` (`nginx`) | 도메인으로 기본 페이지 접속 |
| 10 | 배포 준비: Runner 전용 계정(sudo 없음) 생성, Self-hosted Runner 설치·저장소 연결(라벨 `cookit-deploy`, 서비스 등록), 배포용 SSH 키를 `deploy` 계정에 등록 | `mon01` (`runner`) | GitHub에서 Runner 연결 상태 확인, Runner 계정으로 WAS·LB SSH 접속 |
| 11 | 백업: mariabackup 매일 03:00, 모니터링 서버로 전송, 완료 시각 지표 기록 | `db01` (`backup`) | 백업 파일 생성, 임시 컨테이너로 복구 테스트 |
| 12 | 알림 확인: 테스트 알림 발송, healthchecks.io 생존 신호 연결 | `mon01` | Slack 수신, healthchecks.io에 신호 정상 표시 |

**비고**

1. 앱(백엔드·프론트) 첫 배포는 07 개발·08 CI/CD 이후 진행
2. 1단계도 HAProxy 사용 (3단계에서 DB2만 추가하면 되도록)
3. 비밀번호 등은 Vault 파일에서 읽어 설정 (04 상세 설계 > 4. 계정·비밀 정보)

## 2단계 · HTTPS 적용 (VM 4대)

### 4. 2단계 구축 순서

| 순서 | 작업 | 대상 (role) | 완료 확인 |
|---|---|---|---|
| 1 | 443 포트 포워딩(NAT) 추가, LB 방화벽 443 허용 | 방화벽, `lb01` | 외부에서 443 연결 |
| 2 | 인증서 발급: certbot(HTTP 인증), 자동 갱신 타이머, 갱신 후 Nginx 재적용(reload) | `lb01` (`certbot`) | `certbot renew --dry-run` 성공 |
| 3 | Nginx 443 설정(TLS 1.2·1.3, HTTP/2), 80 → 443 자동 이동 (인증서 갱신 확인 경로는 제외) | `lb01` (`nginx`) | 브라우저에서 HTTPS 접속, 자물쇠 표시 |
| 4 | 인증서 만료 알림(14일·3일 전), 접속 확인 주소를 HTTPS로 변경 | `mon01` (`monitoring`) | Grafana에서 인증서 남은 기간 확인 |
| 5 | HSTS(브라우저가 항상 HTTPS로 접속) 적용 | `lb01` (`nginx`) | 응답 헤더에 HSTS 포함 |

※ HSTS는 한 번 적용하면 브라우저가 일정 기간 HTTP 접속을 거부하므로, HTTPS가 며칠간 문제없이 동작한 뒤 적용

## 3단계 · 이중화 구성 (VM 7대, HTTPS)

### 5. 3단계 구축 순서

| 순서 | 작업 | 대상 (role) | 완료 확인 |
|---|---|---|---|
| 1 | VM 3대 복제(`cookit-lb02`, `cookit-was02`, `cookit-db02`), inventory에 추가, 공통 설정·지표·로그 수집 적용 | 신규 3대 (`common`, `monitoring`, `alloy`) | Grafana에서 7대 확인 |
| 2 | Galera: DB1에 Galera 설정 → DB2 설치 후 합류(전체 복사, mariabackup) → garbd(투표 전용) 실행, 방화벽 4567·4568·4444 허용 | `db01`, `db02`, `mon01` (`mariadb`, `galera`) | 클러스터 노드 3개(DB 2 + garbd), 데이터 동기화 |
| 3 | Galera 상태 확인 스크립트(9200), 양쪽 WAS HAProxy에 DB2를 대기(backup)로 추가 | DB, WAS (`galera`, `haproxy`) | HAProxy 통계(8404)에서 DB1 사용·DB2 대기 |
| 4 | WAS2: Docker, HAProxy, 배포 대상에 추가 후 앱 배포 | `was02` (`docker`, `haproxy`) | WAS2 상태 확인(`/actuator/health`) 정상 |
| 5 | LB2: Nginx·인증서 복사, 인증서 갱신 확인 요청은 LB1로 전달, 인증서 매일 동기화 예약 작업 등록(모니터링 서버), LB2 인증서 만료 확인 추가. 양쪽 Nginx에 WAS2 추가 | `lb01`, `lb02` (`nginx`, `certbot`) | LB2로 직접 HTTPS 접속, API가 WAS 2대로 분배, 동기화 playbook 수동 실행 시 LB2 인증서 갱신 확인 |
| 6 | keepalived: LB 2대에 설치, VIP `.100`, Nginx 상태 확인 스크립트, VRRP 방화벽 허용 | `lb01`, `lb02` (`keepalived`) | VIP가 LB1에 있음, LB1 Nginx 중지 시 LB2로 이동 |
| 7 | NAT 대상 변경: LB1 IP → VIP (DNS 변경 없음) | 방화벽 | 도메인 접속 정상 |
| 8 | 운영 변경: 백업을 DB2에서 실행, 모니터링 대상·이중화 알림 추가, WAS 앱 이상 알림을 warning으로 변경 | `db02`, `mon01` (`backup`, `monitoring`) | DB2 백업 파일 생성, 새 지표 수집 |

**비고**

1. DB1 Galera 설정 적용(DB 재시작)과 NAT 대상 변경 때 잠깐 접속 끊김 가능 → 사용자 적은 시간에 작업
2. keepalived 통신은 유니캐스트(상대 LB IP 지정) 방식 (가상화 환경에서 멀티캐스트가 막힌 경우 대비)
3. 장애 테스트(서버 중지, VIP 이동 시간 측정)는 09 테스트에서 진행


---

이전 문서: [05. 개발 환경](05-dev-environment.md) · 다음 문서: [07. 애플리케이션 개발](07-app-dev.md)
