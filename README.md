# MariaDB 10.6 Replication 구성 가이드

Oracle Linux 7.7 환경에서 MariaDB 10.6 Master-Slave Replication을 구성하는 가이드입니다.
VirtualBox 기반의 실습 환경에서 구성했습니다.

---

## Replication이란

Master(Primary) DB의 변경사항을 Slave(Replica) DB에 자동으로 복제하는 기술이다.
Master에서 발생한 모든 DML/DDL이 Binary Log에 기록되고, Slave가 이를 읽어와 동일하게 적용한다.

```
[Master]  ──→  Binary Log  ──→  [Slave]
 쓰기/읽기                        읽기 전용
```

Oracle의 Data Guard와 유사한 개념이며, 읽기 부하 분산과 백업 용도로 사용한다.

---

## 환경 구성

| 항목 | 내용 |
|------|------|
| OS | Oracle Linux 7.7 |
| DB 버전 | MariaDB 10.6 (LTS) |
| 복제 방식 | GTID 기반 비동기 복제 |
| 가상화 | VirtualBox 7.0.18 |

---

## 서버 구성

| 역할 | 호스트명 | IP | server-id |
|------|---------|----|-----------|
| Master | maria-master | 172.xx.xx.xx | 1 |
| Slave | maria-slave | 172.xx.xx.xx | 2 |

> MariaDB Single 설치가 완료된 VM을 복제하여 2대를 구성한다.

---

## 구성 순서

| 순서 | 내용 |
|------|------|
| 1 | [VM 복제 및 사전 준비](./mariadb-replication.md#1-vm-복제-및-사전-준비) |
| 2 | [Master 설정](./mariadb-replication.md#2-master-설정) |
| 3 | [Slave 설정](./mariadb-replication.md#3-slave-설정) |
| 4 | [복제 테스트](./mariadb-replication.md#4-복제-테스트) |
| 5 | [트러블슈팅](./mariadb-replication.md#5-트러블슈팅) |

---

## 주요 특징

- GTID 기반 복제 구성
- Master-Slave 단방향 복제
- 복제 유저 생성 및 권한 부여
- 실시간 DML 복제 테스트
- 실제 구성 중 발생한 오류 및 해결 방법 포함
