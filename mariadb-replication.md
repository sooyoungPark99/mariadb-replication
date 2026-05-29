# MariaDB 10.6 Replication 구성

Master-Slave Replication을 단계별로 구성한다.
각 단계가 왜 필요한지 이유를 함께 설명한다.

---

## 1. VM 복제 및 사전 준비

### 1-1. VM 복제

MariaDB Single 설치가 완료된 VM을 VirtualBox에서 복제한다.

> 동일한 환경 2대가 필요하므로, 이미 구성된 VM을 복제하면 설치 과정을 반복하지 않아도 된다.

1. VirtualBox에서 기존 MariaDB VM 우클릭 → 복제
2. 복제 방식: 완전한 복제 (Full Clone)
3. MAC 주소 정책: 모든 네트워크 어댑터의 새 MAC 주소 생성
4. 위 과정을 반복하여 총 2대 구성 (Master, Slave)

### 1-2. 호스트명 및 IP 변경 (각 노드에서)

복제한 VM은 호스트명과 IP가 동일하므로 각각 변경해야 한다.

> 복제본은 원본과 호스트명/IP가 같아 충돌이 발생한다. 반드시 서로 다르게 설정한다.

Master 노드:

```bash
hostnamectl set-hostname maria-master
nmcli con mod eth0 ipv4.addresses 172.31.0.251/16
nmcli con mod eth0 ipv4.gateway 172.31.0.1
nmcli con down eth0 && nmcli con up eth0
```

Slave 노드:

```bash
hostnamectl set-hostname maria-slave
nmcli con mod eth0 ipv4.addresses 172.31.0.252/16
nmcli con mod eth0 ipv4.gateway 172.31.0.1
nmcli con down eth0 && nmcli con up eth0
```

### 1-3. /etc/hosts 등록 (양 노드 모두)

> 호스트명으로 서로 통신할 수 있도록 양쪽 노드에 동일하게 등록한다.

```bash
vi /etc/hosts
```

```
172.31.0.251  maria-master
172.31.0.252  maria-slave
```

### 1-4. 복제본 데이터 초기화 (Slave에서)

> VM을 복제하면 Master의 데이터가 그대로 들어있다. Slave는 복제 시작 시점부터 동기화하므로 기존 데이터를 비우고 시작하는 것이 깔끔하다.

```bash
systemctl stop mariadb
rm -rf /var/lib/mysql/*
mysql_install_db --user=mysql --datadir=/var/lib/mysql
systemctl start mariadb
mariadb-secure-installation
```

---

## 2. Master 설정

### 2-1. 설정 파일 수정

```bash
vi /etc/my.cnf.d/server.cnf
```

`[mariadb]` 섹션에 아래 내용을 추가한다.

```
[mariadb]
server-id = 1
log-bin = mariadb-bin
binlog_format = ROW
log-basename = master
gtid_domain_id = 1
```

| 항목 | 설명 |
|------|------|
| server-id | 복제 그룹 내 고유 식별자. 노드마다 달라야 한다 |
| log-bin | Binary Log 활성화. 복제의 핵심이므로 반드시 필요 |
| binlog_format | ROW 방식 권장. 데이터 정합성이 가장 안정적 |
| gtid_domain_id | GTID 기반 복제를 위한 도메인 식별자 |

설정 적용을 위해 재시작한다.

```bash
systemctl restart mariadb
```

### 2-2. 복제 유저 생성

> Slave가 Master에 접속해 Binary Log를 읽어가려면 전용 계정이 필요하다.

```bash
mysql -u root -p
```

```sql
-- 복제 전용 유저 생성
CREATE USER 'repl'@'172.31.0.252' IDENTIFIED BY 'replpass';

-- 복제 권한 부여
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'172.31.0.252';

FLUSH PRIVILEGES;
```

### 2-3. Master 상태 확인

> Slave 설정 시 필요한 현재 Binary Log 위치를 확인한다.

```sql
SHOW MASTER STATUS;
```

```
+--------------------+----------+--------------+------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+--------------------+----------+--------------+------------------+
| mariadb-bin.000001 |      328 |              |                  |
+--------------------+----------+--------------+------------------+
```

GTID 현재 위치도 확인한다.

```sql
SELECT @@gtid_current_pos;
EXIT;
```

---

## 3. Slave 설정

### 3-1. 설정 파일 수정

```bash
vi /etc/my.cnf.d/server.cnf
```

```
[mariadb]
server-id = 2
log-bin = mariadb-bin
binlog_format = ROW
log-basename = slave
gtid_domain_id = 1
read_only = 1
```

> `read_only = 1`: Slave는 읽기 전용이어야 하므로 일반 유저의 쓰기를 차단한다.

```bash
systemctl restart mariadb
```

### 3-2. Master 연결 설정

```bash
mysql -u root -p
```

```sql
-- GTID 기반 복제 시작 위치 설정
SET GLOBAL gtid_slave_pos = "";

-- Master 연결 정보 설정
CHANGE MASTER TO
  MASTER_HOST = '172.31.0.251',
  MASTER_USER = 'repl',
  MASTER_PASSWORD = 'replpass',
  MASTER_PORT = 3306,
  MASTER_USE_GTID = slave_pos;

-- 복제 시작
START SLAVE;
```

### 3-3. 복제 상태 확인

```sql
SHOW SLAVE STATUS\G
```

아래 두 항목이 모두 `Yes`이면 정상이다.

```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

| 항목 | 의미 |
|------|------|
| Slave_IO_Running | Master에서 Binary Log를 받아오는 스레드 정상 동작 |
| Slave_SQL_Running | 받아온 로그를 Slave에 적용하는 스레드 정상 동작 |

> 둘 중 하나라도 No이면 5번 트러블슈팅을 참고한다.

---

## 4. 복제 테스트

### 4-1. Master에서 데이터 생성

```bash
# Master 노드
mysql -u root -p
```

```sql
CREATE DATABASE repltest;
USE repltest;

CREATE TABLE t1 (id INT PRIMARY KEY, name VARCHAR(20));
INSERT INTO t1 VALUES (1, 'replication');

EXIT;
```

### 4-2. Slave에서 복제 확인

```bash
# Slave 노드
mysql -u root -p
```

```sql
USE repltest;
SELECT * FROM t1;
-- (1, 'replication') 이 조회되면 복제 성공
EXIT;
```

### 4-3. Slave 읽기 전용 확인

> Slave에서 쓰기가 차단되는지 확인한다.

```sql
USE repltest;
INSERT INTO t1 VALUES (2, 'test');
-- ERROR 1290: --read-only 오류가 발생하면 정상
```

---

## 5. 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| Slave_IO_Running: No | Master 접속 실패 | 복제 유저 권한/IP, 방화벽, 네트워크 확인 |
| Slave_IO_Running: Connecting | 인증 실패 | 복제 유저 비밀번호, Master IP 재확인 |
| Slave_SQL_Running: No | 데이터 충돌 | 양 노드 데이터 불일치. Slave 초기화 후 재구성 |
| `server-id` 충돌 | 두 노드 server-id 동일 | server.cnf에서 서로 다르게 설정 |
| 복제 지연 발생 | 네트워크/부하 | `SHOW SLAVE STATUS`의 Seconds_Behind_Master 확인 |

### 복제 재구성 방법

복제가 꼬였을 때 Slave를 초기화하고 다시 구성한다.

```sql
STOP SLAVE;
RESET SLAVE ALL;
SET GLOBAL gtid_slave_pos = "";
-- 이후 3-2 단계부터 재수행
```
