# Ansible Role: MySQL Primary-Replica Setup

이 프로젝트는 Ansible을 이용하여 MySQL 8.0 기반의 Primary-Replica(Master-Slave) 환경을 자동 구축하는 Role입니다. GTID 기반의 복제 방식을 채택하였으며, 초기 보안 설정 및 운영 정책(Read-only)이 포함되어 있습니다.

## 1. 주요 기능
- MySQL 8.0 설치 및 my.cnf 최적화 (GTID 복제 설정 포함)
- 초기 보안 설정 (Root 비밀번호 변경, 원격 Root 접속 차단, 익명 사용자 및 테스트 DB 제거)
- Primary-Replica 복제 자동 구성
- Replica 서버의 read_only 및 super_read_only 강제 적용
- 애플리케이션 전용 데이터베이스 및 계정 생성

## 2. 아키텍처 구조


- **Primary**: 읽기/쓰기 허용, 바이너리 로그 생성 주체.
- **Replica**: 읽기 전용 설정, Primary의 데이터를 실시간 동기화.

## 3. 역할 구조 (Role Structure)
```text
roles/mysql_setup/
├── defaults/main.yml        # 기본 변수 정의
├── tasks/
│   ├── main.yml             # 전체 작업 흐름 제어
│   ├── install.yml          # 패키지 설치
│   ├── config.yml           # 설정 파일 배포 및 서비스 관리
│   ├── secure.yml           # 보안 강화 설정
│   ├── repl_user.yml        # 복제용 계정 생성 (Primary)
│   ├── replication_replica.yml # 복제 연결 및 상태 설정 (Replica)
│   ├── app_user.yml         # 앱 전용 DB/계정 생성
│   └── verify.yml           # 최종 상태 검증
└── templates/
    └── my.cnf.j2            # MySQL 설정 템플릿 (GTID 기반)
```

## 4. 설정 및 실행

### 인벤토리 구성 예시
`db_primary`와 `db_replica` 그룹을 통해 서버 역할을 정의합니다.

```ini
[db_primary]
storage1

[db_replica]
storage2

[db:children]
db_primary
db_replica
```

### 변수 관리
민감한 정보는 `group_vars/db/vault.yml`에 정의하고 Ansible Vault를 통해 암호화하여 관리합니다.

- **mysql_root_password**: Root 계정 비밀번호
- **mysql_repl_password**: 복제 계정 비밀번호
- **mysql_app_password**: 애플리케이션 계정 비밀번호

### 실행 명령
```bash
ansible-playbook -i inventory.ini mysql_playbook.yml --ask-vault-pass
```

## 5. 트러블슈팅 및 설계 결정 사항

### SQL Multi-statement 실행 오류 해결
`community.mysql.mysql_query` 모듈을 사용하여 복제 설정(STOP, RESET, CHANGE SOURCE 등)을 단일 쿼리로 실행할 때, 특정 환경에서 1064 문법 에러가 발생하는 이슈를 확인했습니다.
- **해결**: 각 복제 설정 단계를 개별 태스크로 분리하여 실행함으로써 모듈의 Multi-statement 처리 제약 문제를 해결하고 안정성을 확보했습니다.

### 복제 중단(Replication Break) 방지를 위한 계정 생성 순서 조정
Replica 서버에서 계정을 수동으로 먼저 생성한 후 Primary에서 복제를 시작할 경우, Primary의 계정 생성 로그가 Replica에 적용될 때 '중복 계정 에러'로 인해 SQL Thread가 중단되는 현상이 발생했습니다.
- **해결**: 
    1. 복제 연결에 필수적인 `repl_user`만 Primary에서 선제 생성.
    2. Primary-Replica 복제 연결 완료.
    3. 이후 모든 앱 전용 계정(`app_user`) 및 DB는 Primary에서만 생성하여 바이너리 로그를 통해 Replica로 자동 전파되도록 구성했습니다.

### 멱등성 유지 (Replication Re-configuration)
이미 복제가 진행 중인 서버에서 플레이북을 재실행할 때 불필요한 복제 초기화를 방지하도록 설계되었습니다. `force_repl_reset` 변수가 `true`이거나 복제 상태가 비정상일 때만 `RESET REPLICA`를 수행합니다.

## 6. 상태 검증
설치 완료 후 `tasks/verify.yml`을 통해 다음 사항을 자동 검증합니다.
- MySQL 서비스 활성화 여부
- GTID 모드 활성화 및 server_id 일치 여부
- Replica 서버의 Replica_IO_Running 및 Replica_SQL_Running 상태 (Yes/Yes)
- Replica 서버의 read_only 적용 여부
