# MariaDB 10.6 Replication 구성

Master-Slave Replication을 단계별로 구성한다.

---

## 단계별 수행 노드 요약

| 단계 | 작업 | 수행 노드 |
|------|------|----------|
| 1-1 | VM 복제 | VirtualBox (호스트) |
| 1-2 | 호스트명/IP 변경 | Master, Slave 각각 |
| 1-3 | /etc/hosts 등록 | Master, Slave 모두 |
| 1-4 | 데이터 초기화 | Slave |
| 2-1 | server.cnf 설정 | Master |
| 2-2 | 복제 유저 생성 | Master |
| 2-3 | Master 상태 확인 | Master |
| 3-1 | server.cnf 설정 | Slave |
| 3-2 | Master 연결 설정 | Slave |
| 3-3 | 복제 상태 확인 | Slave |
| 4 | 복제 테스트 | Master → Slave |

> 시스템 작업(vi, systemctl, rm 등)은 OS root에서, DB 작업(mysql 접속 후 SQL)은 MariaDB root 계정으로 수행한다.

---

## 1. VM 복제 및 사전 준비

### 1-1. VM 복제 (VirtualBox)

MariaDB Single 설치가 완료된 VM을 VirtualBox에서 복제한다.

> 동일한 환경 2대가 필요하므로, 이미 구성된 VM을 복제하면 설치 과정을 반복하지 않아도 된다.

1. VirtualBox에서 기존 MariaDB VM 우클릭 → 복제
2. 복제 방식: 완전한 복제 (Full Clone)
3. MAC 주소 정책: 모든 네트워크 어댑터의 새 MAC 주소 생성
4. 위 과정을 반복하여 총 2대 구성 (Master, Slave)

### 1-2. 호스트명 및 IP 변경 (Master, Slave 각각)

복제한 VM은 호스트명과 IP가 동일하므로 각각 변경해야 한다.

> 복제본은 원본과 호스트명/IP가 같아 충돌이 발생한다. 반드시 서로 다르게 설정한다.

**Master 노드에서:**

```bash
hostnamectl set-hostname maria-master
nmcli con mod eth0 ipv4.addresses 172.31.0.251/16
nmcli con mod eth0 ipv4.gateway 172.31.0.1
nmcli con down eth0 && nmcli con up eth0
```

<img width="976" height="686" alt="image" src="https://github.com/user-attachments/assets/6bca7d57-1c2f-41e4-8998-f756a41554b6" />

**Slave 노드에서:**

```bash
hostnamectl set-hostname maria-slave
nmcli con mod eth0 ipv4.addresses 172.31.0.252/16
nmcli con mod eth0 ipv4.gateway 172.31.0.1
nmcli con down eth0 && nmcli con up eth0
```

<img width="976" height="685" alt="image" src="https://github.com/user-attachments/assets/d79fd6ba-695e-4cab-beff-1f883049be11" />

> GUI(설정 → 네트워크)로 설정해도 결과는 동일하다.

### 1-3. /etc/hosts 등록 (Master, Slave 모두)

> 호스트명으로 서로 통신할 수 있도록 양쪽 노드에 동일하게 등록한다.

```bash
vi /etc/hosts
```

```
172.31.0.251  maria-master
172.31.0.252  maria-slave
```

### 1-4. 복제본 데이터 초기화 (Slave에서만)

> VM을 복제하면 Master의 데이터가 그대로 들어있다. Slave는 복제 시작 시점부터 동기화하므로 기존 데이터를 비우고 시작하는 것이 깔끔하다.

```bash
systemctl stop mariadb
rm -rf /var/lib/mysql/*
mysql_install_db --user=mysql --datadir=/var/lib/mysql
systemctl start mariadb
mariadb-secure-installation
```

mariadb-secure-installation 진행 시 아래와 같이 답변한다.

| 질문 | 답변 | 이유 |
|------|------|------|
| Enter current password for root | 그냥 Enter | 초기 비밀번호 없음 |
| Switch to unix_socket authentication? | n | 비밀번호 방식 유지 |
| Change the root password? | Y | root 비밀번호 설정 |
| New password / Re-enter | oracle | 실습 환경 편의상 |
| Remove anonymous users? | Y | 익명 접속 차단 |
| Disallow root login remotely? | Y | root 원격 접속 차단 (실습이면 N 가능) |
| Remove test database? | Y | 불필요한 test DB 제거 |
| Reload privilege tables? | Y | 변경 사항 즉시 적용 |

<img width="908" height="635" alt="image" src="https://github.com/user-attachments/assets/8795abb2-5f76-4134-b8f2-e1bebbb57255" />

---

## 2. Master 설정

> 이 섹션은 모두 Master 노드(172.31.0.251)에서 수행한다.

### 2-1. 설정 파일 수정 (Master)

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
<img width="921" height="335" alt="스크린샷 2026-05-29 164547" src="https://github.com/user-attachments/assets/1efdb9cc-a577-4ca0-bfea-5eedf356d6e7" />

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

### 2-2. 복제 유저 생성 (Master)

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

### 2-3. Master 상태 확인 (Master)

> Slave 설정 시 필요한 현재 Binary Log 위치를 확인한다.

```sql
SHOW MASTER STATUS;
```
<img width="925" height="175" alt="image" src="https://github.com/user-attachments/assets/fc2c9f9c-64ef-46da-9152-ec2bd5d000d1" />

GTID 현재 위치도 확인한다.

```sql
SELECT @@gtid_current_pos;
EXIT;
```
<img width="922" height="169" alt="image" src="https://github.com/user-attachments/assets/8adbe4fc-ba38-4a62-b98f-e77c8bce450e" />

---

## 3. Slave 설정

> 이 섹션은 모두 Slave 노드(172.31.0.252)에서 수행한다.

### 3-1. 설정 파일 수정 (Slave)

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

<img width="910" height="303" alt="image" src="https://github.com/user-attachments/assets/78a457cf-c8d6-420d-877d-171dfcda3b67" />

```bash
systemctl restart mariadb
```

### 3-2. Master 연결 설정 (Slave)

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

### 3-3. 복제 상태 확인 (Slave)

```sql
SHOW SLAVE STATUS\G
```

아래 두 항목이 모두 `Yes`이면 정상이다.

```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```
<img width="914" height="325" alt="image" src="https://github.com/user-attachments/assets/bb8e77a9-d999-48da-8b4f-bea69f7a2633" />

| 항목 | 의미 |
|------|------|
| Slave_IO_Running | Master에서 Binary Log를 받아오는 스레드 정상 동작 |
| Slave_SQL_Running | 받아온 로그를 Slave에 적용하는 스레드 정상 동작 |

> 둘 중 하나라도 No이면 5번 트러블슈팅을 참고한다.

---

## 4. 복제 테스트

### 4-1. Master에서 데이터 생성 (Master)

```bash
mysql -u root -p
```

```sql
CREATE DATABASE repltest;
USE repltest;

CREATE TABLE t1 (id INT PRIMARY KEY, name VARCHAR(20));
INSERT INTO t1 VALUES (1, 'replication');

EXIT;
```

### 4-2. Slave에서 복제 확인 (Slave)

```bash
mysql -u root -p
```

```sql
USE repltest;
SELECT * FROM t1;
-- (1, 'replication') 이 조회되면 복제 성공
EXIT;
```
<img width="1863" height="432" alt="image" src="https://github.com/user-attachments/assets/47d6b59c-ed72-4eab-948c-51f6f9fe56ba" />

### 4-3. Slave 읽기 전용 확인 (Slave)

> read_only는 SUPER 권한이 없는 일반 유저만 차단한다.
> root는 SUPER 권한이 있어 read_only 상태에서도 쓰기가 가능하므로, 반드시 일반 유저로 테스트한다.

```sql
-- 일반 유저 생성
CREATE USER test@'localhost' IDENTIFIED BY 'test';
GRANT SELECT, INSERT ON repltest.* TO test@'localhost';
EXIT;
```

```bash
mysql -u test -p repltest
```

```sql
INSERT INTO t1 VALUES (99, 'readonly-test');
-- ERROR 1290: --read-only 오류가 발생하면 정상
```
<img width="905" height="372" alt="image" src="https://github.com/user-attachments/assets/31e473ce-29ce-450a-8e12-192a056023d8" />

---

## 5. 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| Slave_IO_Running: No | Master 접속 실패 | 복제 유저 권한/IP, 방화벽, 네트워크 확인 |
| Slave_IO_Running: Connecting | 인증 실패 | 복제 유저 비밀번호, Master IP 재확인 |
| Slave_SQL_Running: No | 데이터 충돌 | 양 노드 데이터 불일치. Slave 초기화 후 재구성 |
| `server-id` 충돌 | 두 노드 server-id 동일 | server.cnf에서 서로 다르게 설정 |
| 복제 지연 발생 | 네트워크/부하 | `SHOW SLAVE STATUS`의 Seconds_Behind_Master 확인 |

### 복제 재구성 방법 (Slave)

복제가 꼬였을 때 Slave를 초기화하고 다시 구성한다.

```sql
STOP SLAVE;
RESET SLAVE ALL;
SET GLOBAL gtid_slave_pos = "";
-- 이후 3-2 단계부터 재수행
```
