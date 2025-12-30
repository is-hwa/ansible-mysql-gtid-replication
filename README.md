Ansible Role: MySQL Primary-Replica Setup
이 프로젝트는 Ansible을 이용하여 MySQL 8.0 기반의 Primary-Replica(Master-Slave) 환경을 자동 구축하는 Role입니다. GTID 기반의 복제 방식을 채택하였으며, 초기 보안 설정 및 운영 정책(Read-only)이 포함되어 있습니다.

주요 기능
MySQL 8.0 설치 및 my.cnf 최적화 (GTID 복제 설정 포함)

초기 보안 설정 (Root 비밀번호 변경, 원격 Root 접속 차단, 익명 사용자 및 테스트 DB 제거)

Primary-Replica 복제 자동 구성

Replica 서버의 read_only 및 super_read_only 강제 적용

애플리케이션 전용 데이터베이스 및 계정 생성

역할 구조 (Role Structure)
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
    └── my.cnf.j2            # MySQL 설정 템플릿
설정 및 실행
인벤토리 구성
db_primary와 db_replica 그룹을 통해 서버 역할을 정의합니다.

Ini, TOML

[db_primary]
storage1

[db_replica]
storage2
변수 관리
민감한 정보(비밀번호 등)는 group_vars/db/vault.yml에 정의하고 Ansible Vault를 통해 암호화하여 관리할 것을 권장합니다.

mysql_root_password: Root 계정 비밀번호

mysql_repl_password: 복제 계정 비밀번호

mysql_app_password: 앱 계정 비밀번호

실행 명령
Bash

ansible-playbook -i inventory.ini mysql_playbook.yml --ask-vault-pass
트러블슈팅 및 설계 결정 사항
1. SQL Multi-statement 실행 오류 해결
community.mysql.mysql_query 모듈을 사용하여 복제 설정(STOP, RESET, CHANGE SOURCE 등)을 한 번에 실행할 때, 환경에 따라 1064 문법 에러가 발생하는 이슈가 있었습니다.

해결: 복제 설정 과정을 각각의 단일 SQL 태스크로 분리하여 실행함으로써 모듈의 Multi-statement 처리 제약 문제를 해결하고 안정성을 높였습니다.

2. 복제 중단(Replication Break) 방지를 위한 유저 생성 순서 조정
Primary와 Replica에서 동시에 동일한 앱 계정을 생성할 경우, 복제가 시작될 때 Replica에서 '이미 존재하는 계정' 에러로 인해 SQL Thread가 중단되는 현상이 발생했습니다.

해결:

복제 연결에 필수적인 repl_user만 먼저 생성.

Primary-Replica 복제 연결 완료.

이후 모든 앱 계정(app_user)과 DB는 Primary에서만 생성.

결과적으로 모든 변경 사항이 바이너리 로그를 통해 Replica로 전달되도록 하여 충돌을 방지했습니다.

3. 멱등성 유지 (Replication Re-configuration)
이미 복제가 진행 중인 서버에서 플레이북을 재실행할 때 불필요한 복제 초기화를 방지하도록 설계되었습니다. force_repl_reset 변수가 true이거나 복제 상태가 비정상일 때만 RESET REPLICA를 수행합니다.

상태 검증
설치 완료 후 tasks/verify.yml을 통해 다음 사항을 자동 검증합니다.

MySQL 서비스 활성화 여부

GTID 모드 활성화 및 server_id 일치 여부

Replica 서버의 Slave_IO_Running 및 Slave_SQL_Running 상태 (Yes/Yes)

Replica 서버의 읽기 전용 모드 적용 여부
