# Oracle 19c 백업과 복구 실습 가이드 (RMAN)

> 백업은 만들어 둔 순간이 아니라 복원해 본 순간에 비로소 백업이 된다.
> 이 가이드는 장애를 일부러 만들고, RMAN과 Flashback으로 되살리고, 걸린 시간과 잃은 데이터를 기록한다.
> 모든 명령과 화면은 2026-10-31부터 11-01까지 실제로 실행한 결과를 바탕으로 고쳤다. 화면 예시는 그때 터미널에 나온 그대로다.

| 항목 | 내용 |
|---|---|
| 판 | 2판 (2026-11-01) |
| 1판과 달라진 점 | 실제 실행에서 나온 오류 3건 수정, 실제 화면 예시 추가, 확인 단계 4곳 추가, 문체 정리 |
| 검증 결과 | 1~7장 전 단계 통과. 시나리오 A, B, C는 데이터 손실 0, D는 예상대로 커밋 1건만 잃었다. 실습 뒤 서버 점검 PASS 17, FAIL 0 |

---

## 0. 시작하기 전에

### 0-1. 이 가이드로 하는 것

| 장 | 내용 | 다루는 장애 |
|---|---|---|
| 1장 | 사전 점검, 실습 데이터 만들기 | 없음 |
| 2장 | RMAN 설정, 첫 백업(Level 0), 백업 검증 | 없음 |
| 3장 | 시나리오 A: 사용자 실수 | 잘못된 DELETE, 테이블 DROP |
| 4장 | 시나리오 B: 데이터 파일 1개 유실 | DB를 내리지 않고 테이블스페이스만 복구 |
| 5장 | 시나리오 C: 컨트롤 파일 전부 유실 | 컨트롤 파일 복원 후 완전 복구 |
| 6장 | 시나리오 D: 데이터 디스크 전체 유실 | DB 전체 복원 후 불완전 복구 |
| 7장 | 백업 자동화 | 매일 백업 스크립트와 cron |
| 8장 | 결과 정리 | 실제 측정값 |

실제로 걸린 시간은 1~2장 약 30분, 시나리오마다 10~20분, 7장 15분이었다. 장 단위로 끊어서 해도 앞뒤가 이어진다.

### 0-2. 안전 규칙

1. **실습용 VM에서만 한다.** 운영 중인 DB나 다른 사람이 쓰는 DB에서는 절대 하지 않는다. 4~6장은 데이터 파일을 직접 지운다.
2. **시나리오마다 시작 전에 VM 스냅샷을 찍는다.** VirtualBox에서 VM을 고르고, 스냅샷 탭에서 찍기를 누른다. 복구에 실패해도 그 시점으로 돌아갈 수 있다.
3. 명령은 **상자 위에 적힌 실행 위치**에서만 실행한다. 같은 명령도 위치가 다르면 결과가 다르거나 실패한다.
4. 지우는 명령(`rm`)은 붙여 넣은 뒤 **경로를 한 번 더 읽고** Enter를 누른다.
5. `<…>`로 표시한 부분은 **꺾쇠까지 지우고 실제 값으로 바꿔서** 실행한다. 그대로 실행하면 "그런 이름이 없다"는 오류가 난다.

### 0-3. 실행 위치 표기

모든 명령 상자 위에 아래 표시 중 하나를 붙였다.

| 표시 | 어디에서 | 들어가는 방법 | 프롬프트 모양 |
|---|---|---|---|
| **[root #]** | 리눅스 root 계정 셸 | SSH로 root 접속 | `[root@호스트 ~]#` |
| **[oracle $]** | 리눅스 oracle 계정 셸 | SSH로 oracle 접속 | `$` 또는 `>`로 끝남 |
| **[SQL> 루트]** | SQL*Plus, CDB 루트(CDB$ROOT) | [oracle $]에서 `sqlplus / as sysdba` | `SQL>` |
| **[SQL> PDB]** | SQL*Plus, PDB 안 | [SQL> 루트]에서 `alter session set container = pdb;` | `SQL>` |
| **[RMAN>]** | RMAN | [oracle $]에서 `rman target /` | `RMAN>` |
| **[PC]** | 작업하는 Windows PC | VirtualBox 화면 | 없음 |

- SQL*Plus와 RMAN은 `exit`로 빠져나와 [oracle $]로 돌아간다.
- **[SQL> 루트]와 [SQL> PDB]는 프롬프트가 똑같이 `SQL>`이다.** 헷갈리면 `show con_name`을 실행한다. `CDB$ROOT`면 루트, `PDB`면 PDB다.
- 상자 안의 한 줄이 한 명령이다. 긴 줄은 붙여 넣은 뒤 한 줄로 들어갔는지 보고 Enter를 누른다.
- 여러 줄 블록을 붙여 넣으면 MobaXterm 화면에 입력 줄과 실행 결과가 겹쳐 보일 때가 있다. 표시만 그런 것이니 결과 줄로 판단한다.

### 0-4. 기준 환경과 바꿀 값

명령에는 아래 **기준 값**을 그대로 적었다. 내 환경이 다르면 오른쪽 칸에 적어 두고, 명령을 실행할 때 바꿔 쓴다.

| 항목 | 기준 값 | 확인 명령 (위치) | 내 환경 |
|---|---|---|---|
| DB 이름(SID) | `orcl` | `echo $ORACLE_SID` [oracle $] | |
| PDB 이름 | `pdb` | `show pdbs` [SQL> 루트] | |
| ORACLE_HOME | `/u01/app/oracle/product/19.0.0.0/dbhome_1` | `echo $ORACLE_HOME` [oracle $] | |
| 데이터 파일 위치 | `/oradata/ORCL` (PDB는 `/oradata/ORCL/pdb`) | `select name from v$datafile;` [SQL> 루트] | |
| 컨트롤 파일 | `/oradata/ORCL/control01.ctl`, `control02.ctl` | `show parameter control_files` [SQL> 루트] | |
| 온라인 리두 로그 | `/oradata/ORCL/redo01.log`~`redo03.log` | `select member from v$logfile;` [SQL> 루트] | |
| 아카이브 로그 위치 | `/backup/arch` | `archive log list` [SQL> 루트] | |
| 백업 저장 위치 | `/backup/rman` (이 가이드에서 새로 만든다) | 없음 | |
| 백업 디스크 여유 | 20GB 이상 | `df -h /backup` [oracle $] | |
| 호스트 이름 | `dbts93` | `hostname` [oracle $] | |

**이 가이드가 전제로 하는 구성**
- Oracle 19c, CDB와 PDB 1개
- ARCHIVELOG 모드
- 데이터 파일, 컨트롤 파일, 리두 로그가 있는 디스크(`/oradata`)와 아카이브 로그, 백업이 있는 디스크(`/backup`)가 **서로 다르다**

마지막 조건이 6장을 복구할 수 있는 이유다. 같은 디스크에 두면 그 디스크가 망가질 때 백업도 함께 사라진다.

### 0-5. 용어

| 용어 | 뜻 |
|---|---|
| RMAN(Recovery Manager) | Oracle의 백업과 복구 전용 도구 |
| 컨트롤 파일(control file) | DB의 구조(데이터 파일과 리두 로그 위치, 현재 SCN 등)를 적어 둔 파일. 없으면 DB를 MOUNT할 수 없다 |
| 온라인 리두 로그(online redo log) | 모든 변경을 순서대로 적는 로그. 그룹 몇 개를 돌아가며 덮어쓴다 |
| 아카이브 로그(archived log) | 다 찬 리두 로그를 덮어쓰기 전에 복사해 둔 파일. 백업 이후의 변경을 되살리는 재료다 |
| SCN(System Change Number) | DB 안의 시각 번호. 변경이 일어날 때마다 커진다 |
| 복원(RESTORE) | 백업에서 파일을 원래 자리로 꺼내 놓는 것. 백업 시점의 내용이 된다 |
| 복구(RECOVER) | 복원한 파일에 아카이브 로그와 리두 로그를 적용해 원하는 시점까지 앞으로 감는 것 |
| 완전 복구 | 마지막 커밋까지 모두 되살린다. 데이터 손실이 없다 |
| 불완전 복구 | 어느 시점까지만 되살린다. 그 뒤의 변경은 잃는다. 끝나면 `OPEN RESETLOGS`가 필요하다 |
| RESETLOGS, 인카네이션(incarnation, DB의 세대) | 불완전 복구나 백업 컨트롤 파일을 쓴 복구 뒤에 로그 번호를 1부터 다시 시작하는 것. DB의 새 세대가 시작된다 |
| Level 0, Level 1 | 증분 백업의 기준(전체)과, 그 뒤 바뀐 블록만 담는 백업 |
| RTO(Recovery Time Objective, 복구 목표 시간) | 장애가 난 뒤 서비스가 다시 되기까지 걸린 시간 |
| RPO(Recovery Point Objective, 복구 목표 시점) | 잃은 데이터의 양 또는 시간 범위 |

---

## 1. 사전 점검과 실습 데이터

### 1-1. DB 상태와 DBID 기록

**[oracle $]**
```
sqlplus / as sysdba
```
**[SQL> 루트]**
```
select name, dbid, log_mode, cdb from v$database;
archive log list
show pdbs
show parameter control_files
select member from v$logfile;
exit
```
**[oracle $]**
```
df -h /oradata /backup
```

**실제 화면 (2026-10-31 14:27)**
```
NAME                              DBID LOG_MODE
--------------------------- ---------- ------------------------------------
CDB
---------
ORCL                        1775805046 ARCHIVELOG
YES

Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            /backup/arch
Current log sequence           10

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
         2 PDB$SEED                       READ ONLY  NO
         3 PDB                            READ WRITE NO

/dev/mapper/VGDb-LVDbData    100G  4.7G   96G   5% /oradata
/dev/mapper/VGDb-LVDbBackup  100G  1.1G   99G   2% /backup
```

**확인할 것**
- `LOG_MODE`가 `ARCHIVELOG`인지 본다. 아니면 이 가이드를 진행할 수 없다.
- PDB가 `READ WRITE`인지 본다.
- `/backup`에 20GB 이상 남았는지 본다.
- **DBID를 아래 표에 적는다.** 5장과 6장에서 컨트롤 파일 없이 복원할 때 이 숫자가 없으면 시작할 수 없다. 실무에서도 DBID는 백업 문서에 따로 적어 둔다.

| 기록 | 값 |
|---|---|
| DBID | |
| 기록한 날짜 | |

`select member form v$logfile;`처럼 `from`을 잘못 치면 `ORA-00923: FROM keyword not found`가 난다. 고쳐서 다시 치면 된다.

### 1-2. 백업 폴더 만들기

**[oracle $]**
```
mkdir -p /backup/rman /backup/logs /backup/scripts
ls -ld /backup/rman /backup/logs /backup/scripts
```
**실제 화면**
```
drwxr-xr-x 2 oracle oinstall 6 Oct 31 14:31 /backup/logs
drwxr-xr-x 2 oracle oinstall 6 Oct 31 14:31 /backup/rman
drwxr-xr-x 2 oracle oinstall 6 Oct 31 14:31 /backup/scripts
```
`Permission denied`가 나오면 `/backup`의 소유자가 oracle이 아니다. [root #]에서 `chown oracle:oinstall /backup`을 실행하고 다시 한다.

### 1-3. 실습용 테이블스페이스, 사용자, 데이터

**[oracle $]**
```
sqlplus / as sysdba
```
**[SQL> 루트]**
```
alter session set container = pdb;
show con_name
```
`PDB`가 나오면 여기부터 **[SQL> PDB]**다.

**[SQL> PDB]**
```
create tablespace lab_ts datafile '/oradata/ORCL/pdb/lab_ts01.dbf' size 100m autoextend on next 50m maxsize 2g;
create user lab identified by <비밀번호> default tablespace lab_ts quota unlimited on lab_ts;
grant create session, create table to lab;
create table lab.orders (order_id number primary key, item varchar2(50), qty number, created_at date default sysdate);
insert into lab.orders (order_id, item, qty) select level, 'ITEM-' || level, mod(level, 10) + 1 from dual connect by level <= 10000;
create table lab.event_log (seq number, note varchar2(100), at timestamp default systimestamp);
insert into lab.event_log (seq, note) values (1, 'baseline');
commit;
select count(*) from lab.orders;
select seq, note, to_char(at, 'YYYY-MM-DD HH24:MI:SS') at from lab.event_log order by seq;
exit
```
**실제 화면**
```
Tablespace created.
User created.
Grant succeeded.
Table created.
10000 rows created.
Table created.
1 row created.
Commit complete.

  COUNT(*)
----------
     10000

       SEQ NOTE       AT
         1 baseline   2026-10-31 14:40:13
```

**만든 것과 이유**
- 테이블스페이스 `lab_ts`: 4장에서 이 파일 하나만 지우고 복구한다. 시스템 파일을 건드리지 않으려고 따로 만든다.
- `lab.orders` 1만 행: 복구 뒤 행 수로 데이터가 돌아왔는지 판단한다.
- `lab.event_log`: 시나리오마다 "백업 뒤에 넣은 행"을 하나씩 남긴다. **어느 시점까지 복구됐는지** 보여 주는 표시다.

기본 키는 이름을 정하지 않아 Oracle이 `SYS_C…` 이름을 붙인다. 3장에서 이 이름을 직접 되돌려 본다. 운영에서는 `constraint orders_pk primary key`처럼 이름을 정해 두는 편이 관리하기 좋다.

---

## 2. RMAN 설정과 첫 백업

### 2-1. RMAN 기본 설정

**[oracle $]**
```
rman target /
```
**실제 화면**
```
connected to target database: ORCL (DBID=1775805046)
```
DBID가 1-1에서 적은 값과 같은지 본다.

**[RMAN>]**
```
SHOW ALL;
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/backup/rman/cf_%F';
CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT '/backup/rman/%d_%T_%U.bkp';
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
CONFIGURE BACKUP OPTIMIZATION ON;
CONFIGURE ARCHIVELOG DELETION POLICY TO BACKED UP 1 TIMES TO DEVICE TYPE DISK;
SHOW ALL;
```
**실제 화면 (처음 `SHOW ALL`의 일부)**
```
CONFIGURE RETENTION POLICY TO REDUNDANCY 1; # default
CONFIGURE BACKUP OPTIMIZATION OFF; # default
CONFIGURE CONTROLFILE AUTOBACKUP ON; # default
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '%F'; # default
CONFIGURE ARCHIVELOG DELETION POLICY TO NONE; # default
```
**실제 화면 (설정 뒤 `SHOW ALL`의 일부)**
```
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
CONFIGURE BACKUP OPTIMIZATION ON;
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/backup/rman/cf_%F';
CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT   '/backup/rman/%d_%T_%U.bkp';
CONFIGURE ARCHIVELOG DELETION POLICY TO BACKED UP 1 TIMES TO DISK;
```
바꾼 줄은 끝의 `# default`가 사라진다. 19c는 `CONTROLFILE AUTOBACKUP`의 기본값이 이미 `ON`이다. 그래도 직접 설정해 두면 "의도한 값"이라는 기록이 남는다.

**설정의 뜻과 이유**
- `CONTROLFILE AUTOBACKUP ON`: 백업할 때마다, 그리고 DB 구조가 바뀔 때마다 컨트롤 파일과 spfile을 자동으로 백업한다. **5장과 6장 복구의 출발점**이다.
- `AUTOBACKUP FORMAT … cf_%F`: 자동 백업 파일 이름이다. `%F`에 DBID와 날짜가 들어가서, 컨트롤 파일이 없어도 DBID로 찾을 수 있다.
- `CHANNEL … FORMAT`: 일반 백업 파일 이름이다. `%d`는 DB 이름, `%T`는 날짜, `%U`는 겹치지 않는 고유 번호다.
- `RECOVERY WINDOW OF 7 DAYS`: 최근 7일 안의 어느 시점으로든 복구할 수 있게 백업을 남긴다. 그보다 오래된 백업은 쓸모없음(OBSOLETE)으로 본다.
- `BACKUP OPTIMIZATION ON`: 이미 백업했고 바뀌지 않은 파일은 다시 백업하지 않는다. 6장에서 이 설정이 복원 경로에 영향을 준 것을 직접 본다.
- `ARCHIVELOG DELETION POLICY`: 디스크에 한 번 이상 백업된 아카이브 로그만 지울 수 있게 막는다.

### 2-2. 첫 백업: Level 0

**[RMAN>]**
```
BACKUP AS COMPRESSED BACKUPSET INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG;
```
**실제 화면 (요약)**
```
Starting backup at 31-OCT-26
current log archived
input archived log thread=1 sequence=7 ... sequence=10
piece handle=/backup/rman/ORCL_20261031_025690q0_1_1.bkp tag=TAG20261031T144624
input datafile file number=00001 name=/oradata/ORCL/system01.dbf
...
input datafile file number=00013 name=/oradata/ORCL/pdb/lab_ts01.dbf
...
Starting Control File and SPFILE Autobackup at 31-OCT-26
piece handle=/backup/rman/cf_c-1775805046-20261031-01 comment=NONE
Finished Control File and SPFILE Autobackup at 31-OCT-26
```
실제로는 14:46:24에 시작해 14:48:16에 끝났다. **약 2분**이다. 마지막에 `Finished Control File and SPFILE Autobackup`이 나오면 끝난 것이다.

`lab_ts01.dbf`의 **파일 번호(file number)를 적어 둔다.** 기준 환경에서는 13이었다. 4장에서 쓴다.

**[RMAN>]**
```
LIST BACKUP SUMMARY;
REPORT NEED BACKUP;
RESTORE DATABASE VALIDATE;
exit
```
**[oracle $]**
```
ls -lh /backup/rman/
```
**실제 화면**
```
Key     TY LV S Device Type Completion Time #Pieces #Copies Compressed Tag
1       B  F  A DISK        31-OCT-26       1       1       NO         TAG20261031T144102
2       B  A  A DISK        31-OCT-26       1       1       YES        TAG20261031T144624
3       B  0  A DISK        31-OCT-26       1       1       YES        TAG20261031T144639
4       B  0  A DISK        31-OCT-26       1       1       YES        TAG20261031T144639
5       B  0  A DISK        31-OCT-26       1       1       YES        TAG20261031T144639
6       B  A  A DISK        31-OCT-26       1       1       YES        TAG20261031T144815
7       B  F  A DISK        31-OCT-26       1       1       NO         TAG20261031T144816

Report of files that must be backed up to satisfy 7 days recovery window
File Days  Name
---- ----- -----------------------------------------------------
(빈 표)

channel ORA_DISK_1: validation complete, elapsed time: 00:00:35
channel ORA_DISK_1: validation complete, elapsed time: 00:00:16
channel ORA_DISK_1: validation complete, elapsed time: 00:00:15
Finished restore at 31-OCT-26

total 660M
-rw-r----- 1 oracle oinstall  18M Oct 31 14:48 cf_c-1775805046-20261031-01
-rw-r----- 1 oracle oinstall 130M Oct 31 14:46 ORCL_20261031_025690q0_1_1.bkp
-rw-r----- 1 oracle oinstall 292M Oct 31 14:47 ORCL_20261031_035690qf_1_1.bkp
-rw-r----- 1 oracle oinstall 101M Oct 31 14:47 ORCL_20261031_045690s6_1_1.bkp
-rw-r----- 1 oracle oinstall 120M Oct 31 14:48 ORCL_20261031_055690sv_1_1.bkp
-rw-r----- 1 oracle oinstall 6.5K Oct 31 14:48 ORCL_20261031_065690tf_1_1.bkp
```

**화면 읽는 법**

- `LV`가 `0`인 줄은 Level 0 데이터 파일 백업이다. 루트, PDB, PDB$SEED 3묶음이라 3줄이다.
- `LV`가 `A`인 줄은 아카이브 로그 백업이다. 백업 앞(Key 2)과 뒤(Key 6)에 하나씩 생긴다.
- `LV`가 `F`인 줄은 컨트롤 파일과 spfile 자동 백업이다.
- Key 1(`F`, `Compressed NO`, 14:41)은 1-3에서 테이블스페이스를 만든 **구조 변경 때문에** 자동으로 생긴 백업이다. 2-1에서 형식을 바꾸기 전이라 기본 위치(`$ORACLE_HOME/dbs`)에 저장됐다. 정상이다.

데이터가 들어 있는 `/oradata` 4.7GB가 압축 백업으로는 **660MB**가 됐다.

**명령을 이렇게 쓴 이유**
- `INCREMENTAL LEVEL 0`: 내용은 전체 백업과 같지만 **뒤에 올 Level 1(증분)의 기준**이 된다. 그냥 `BACKUP DATABASE`로 만든 전체 백업은 증분의 기준이 될 수 없다.
- `PLUS ARCHIVELOG`: 백업 앞뒤의 아카이브 로그를 함께 백업한다. 데이터 파일 백업만으로는 일관된 상태로 복구할 수 없다.
- `COMPRESSED`: 기본(BASIC) 압축이다. 추가 라이선스가 필요 없다.
- `RESTORE … VALIDATE`: 파일을 덮어쓰지 않고 **백업을 읽을 수 있는지만** 검사한다. "백업은 있는데 복원이 안 되는" 상황을 미리 막는다.

> **[PC]** 스냅샷: `백업-L0-완료`

---

## 3. 시나리오 A: 사용자 실수 (DELETE, DROP)

**상황**: 개발자가 조건을 잘못 써서 데이터를 지우고 커밋했다. 이어서 테이블을 통째로 DROP했다.
**쓰는 기능**: Flashback Query(언두 데이터로 과거 내용 조회), Flashback Table(휴지통에서 되살리기). **RMAN 백업 없이** 복구한다.

### A-1. 잘못된 DELETE

**[oracle $]**
```
sqlplus / as sysdba
```
**[SQL> 루트]**
```
alter session set container = pdb;
select to_char(systimestamp, 'YYYY-MM-DD HH24:MI:SS') now from dual;
```
출력된 시각을 적는다. 아래에서 이 시각을 **T1**이라고 부른다. (실제 값: `2026-10-31 15:44:54`)

**[SQL> PDB]** 실수 만들기
```
delete from lab.orders where order_id <= 1000;
commit;
select count(*) from lab.orders;
```
**기대 결과**: `1000 rows deleted.` → `9000`

**[SQL> PDB]** T1 시점 확인과 되살리기. `<T1>` 자리에 적어 둔 시각을 넣는다.
```
select count(*) from lab.orders as of timestamp to_timestamp('<T1>', 'YYYY-MM-DD HH24:MI:SS');
insert into lab.orders select * from lab.orders as of timestamp to_timestamp('<T1>', 'YYYY-MM-DD HH24:MI:SS') where order_id <= 1000;
commit;
select count(*) from lab.orders;
```
**실제 화면**
```
SQL> select count(*) from lab.orders as of timestamp to_timestamp('2026-10-31 15:44:54', 'YYYY-MM-DD HH24:MI:SS');
  COUNT(*)
----------
     10000

SQL> insert into lab.orders select * from lab.orders as of timestamp to_timestamp('2026-10-31 15:44:54', 'YYYY-MM-DD HH24:MI:SS') where order_id <= 1000;
1000 rows created.

SQL> select count(*) from lab.orders;
     10000
```
시각을 넣을 때 **작은따옴표가 짝이 맞는지** 본다. `'2026-10-31 15:44:54, 'YYYY…`처럼 닫는 따옴표가 빠지면 `ORA-01756: quoted string not properly terminated`가 난다. 아무것도 바뀌지 않았으니 고쳐서 다시 실행한다.

- `AS OF TIMESTAMP`는 언두(UNDO)에 남은 변경 전 이미지로 그 시각의 테이블 내용을 읽는다.
- 한계: 언두는 일정 시간만 남는다(`undo_retention`, 기본 900초, 공간이 부족하면 더 짧다). 오래 지나면 `ORA-01555`가 나고, 그때는 RMAN 시점 복구로 넘어간다.

### A-2. 테이블 DROP

**[SQL> PDB]**
```
drop table lab.orders;
select count(*) from lab.orders;
```
**기대 결과**: `ORA-00942: table or view does not exist`

**[SQL> PDB]** 휴지통 확인과 되살리기
```
select object_name, original_name, type, droptime from dba_recyclebin where owner = 'LAB';
flashback table lab.orders to before drop;
select count(*) from lab.orders;
select i.index_name, c.constraint_name from dba_indexes i left join dba_constraints c on c.index_name = i.index_name and c.owner = i.owner where i.owner = 'LAB' and i.table_name = 'ORDERS';
```
**실제 화면**
```
OBJECT_NAME                       ORIGINAL_NAME   TYPE    DROPTIME
BIN$Xx5TrBXKC6HgY10KqMBVmA==$0    SYS_C007550     INDEX   2026-10-31:15:50:22
BIN$Xx5TrBXLC6HgY10KqMBVmA==$0    ORDERS          TABLE   2026-10-31:15:50:23

Flashback complete.

  COUNT(*)
----------
     10000

INDEX_NAME                        CONSTRAINT_NAME
BIN$Xx5TrBXKC6HgY10KqMBVmA==$0    BIN$Xx5TrBXJC6HgY10KqMBVmA==$0
```
SQL*Plus는 열이 길어 한 행을 여러 줄로 나눠 보여 준다. 위 화면은 읽기 쉽게 한 줄로 모았다.

**화면 읽는 법**
- 휴지통에는 **TABLE 줄과 INDEX 줄**이 따로 있다. DROP하면 인덱스가 먼저(15:50:22), 테이블이 나중(15:50:23)에 들어간다.
- 테이블은 `ORDERS`로 돌아왔지만, **인덱스와 기본 키 제약 조건은 `BIN$` 이름으로 남았다.**
- 세 `BIN$` 이름은 끝에서 몇 번째 글자 하나만 다르다. 테이블은 `…XL…`, 인덱스는 `…XK…`, 제약 조건은 `…XJ…`였다. 헷갈리기 쉬우니 줄을 확인하고 복사한다.

**[SQL> PDB]** 원래 이름으로 되돌리기
- 인덱스 자리에는 휴지통 조회에서 **TYPE이 INDEX인 줄**의 `BIN$` 이름을 넣는다.
- 새 이름은 같은 줄의 `ORIGINAL_NAME`이다.
- 제약 조건 자리에는 바로 위 조회의 `CONSTRAINT_NAME`(BIN$…)을 넣는다.
```
alter index lab."<INDEX 줄의 BIN$ 이름>" rename to <INDEX 줄의 ORIGINAL_NAME>;
alter table lab.orders rename constraint "<CONSTRAINT_NAME의 BIN$ 이름>" to <INDEX 줄의 ORIGINAL_NAME>;
select i.index_name, c.constraint_name from dba_indexes i left join dba_constraints c on c.index_name = i.index_name and c.owner = i.owner where i.owner = 'LAB' and i.table_name = 'ORDERS';
exit
```
**실제 화면**
```
SQL> alter index lab."BIN$Xx5TrBXKC6HgY10KqMBVmA==$0" rename to SYS_C007550;
Index altered.

SQL> alter table lab.orders rename constraint "BIN$Xx5TrBXJC6HgY10KqMBVmA==$0" to SYS_C007550;
Table altered.

INDEX_NAME     CONSTRAINT_NAME
SYS_C007550    SYS_C007550
```

**자주 나는 오류**

- `ORA-00918: column ambiguously defined`: 두 뷰에 모두 `index_name` 열이 있는데 어느 쪽인지 적지 않았다. `i.index_name`처럼 앞에 별칭을 붙인다.
- `ORA-01418: specified index does not exist`: TABLE 줄의 `BIN$` 이름을 넣었다. 테이블은 이미 `ORDERS`로 돌아와 그 이름이 없다. INDEX 줄의 이름을 넣는다.
- `ORA-23292: The constraint does not exist`: `<…>` 자리 표시를 그대로 실행했다. 조회한 실제 이름으로 바꾼다.

인덱스와 제약 조건에 같은 이름을 붙여도 된다. 처음 기본 키를 만들 때도 Oracle이 둘에 같은 이름을 붙였다. 둘은 이름을 따로 관리한다.

- 휴지통(Recycle Bin): `DROP TABLE`은 바로 지우지 않고 이름을 `BIN$…`으로 바꿔 둔다. 공간이 모자라면 오래된 것부터 실제로 지워진다.
- 되살릴 수 없는 경우: `DROP TABLE … PURGE`, `TRUNCATE`, SYSTEM 테이블스페이스의 테이블이다. 이때는 RMAN 시점 복구가 필요하다.
- 이름 되돌리기: Flashback Table은 테이블 이름만 되돌린다. **인덱스와 제약 조건 이름은 DBA가 직접 되돌려야 한다.** 복구 뒤 확인 항목이다.

**기록 (실측)**

| 항목 | 값 |
|---|---|
| DELETE 복구 | T1(15:44:54)에서 5분 이내, 손실 0행 |
| DROP 복구 | 테이블과 데이터 1분 이내, 이름 정리까지 포함해 손실 0행 |

---

## 4. 시나리오 B: 데이터 파일 1개 유실

**상황**: 운영 중 누군가 실수로 PDB의 데이터 파일 하나를 지웠다. 다른 테이블스페이스는 멀쩡하니 **DB를 열어 둔 채** 그 테이블스페이스만 복구한다.

> **[PC]** 스냅샷: `시나리오B-전`

### B-1. 백업 뒤 변경 남기기

**[oracle $]**
```
sqlplus / as sysdba
```
**[SQL> 루트]**
```
alter session set container = pdb;
insert into lab.orders (order_id, item, qty) select 10000 + level, 'NEW-' || level, 1 from dual connect by level <= 500;
insert into lab.event_log (seq, note) values (2, 'B: after level0 backup');
commit;
select count(*) from lab.orders;
select file_id, file_name from dba_data_files where tablespace_name = 'LAB_TS';
exit
```
**실제 화면**
```
500 rows created.
1 row created.
Commit complete.
     10500
FILE_ID FILE_NAME
     13 /oradata/ORCL/pdb/lab_ts01.dbf
```
추가한 500행과 seq 2는 **Level 0 백업에 없는 데이터**다. 복구 뒤 이것이 살아 있으면 백업에 로그까지 적용해 지금 시점까지 복구된 것이다.

### B-2. 장애 만들기

**[oracle $]**
```
rm /oradata/ORCL/pdb/lab_ts01.dbf
ls -l /oradata/ORCL/pdb/
sqlplus / as sysdba
```
**[SQL> 루트]**
```
alter system flush buffer_cache;
exit
```
**[oracle $]** 새 세션으로 접속해서 읽어 본다.
```
sqlplus lab/Lab_19c@//$(hostname):1521/pdb
```
**[SQL> PDB]** (lab 사용자로 PDB에 접속한 상태)
```
select count(*) from orders;
exit
```
**실제 화면**
```
ERROR at line 1:
ORA-01116: error in opening database file 13
ORA-01110: data file 13: '/oradata/ORCL/pdb/lab_ts01.dbf'
ORA-27041: unable to open file
Linux-x86_64 Error: 2: No such file or directory
```

**[oracle $]** alert log도 본다.
```
tail -30 $ORACLE_BASE/diag/rdbms/orcl/orcl/trace/alert_orcl.log
```
**실제 화면 (끝부분)**
```
2026-10-31T16:33:30.938642+09:00
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_mz00_413721.trc:
ORA-01110: data file 13: '/oradata/ORCL/pdb/lab_ts01.dbf'
ORA-01565: error in identifying file '/oradata/ORCL/pdb/lab_ts01.dbf'
ORA-27037: unable to obtain file status
Checker run found 1 new persistent data failures
```
`Checker run found 1 new persistent data failures`는 Oracle이 장애를 스스로 감지해 기록했다는 뜻이다.

**왜 이렇게 만드나**
- 지워도 바로 오류가 나지 않는 이유: 리눅스는 이미 열린 파일을 지워도, 연 프로세스는 계속 읽고 쓸 수 있다. 그래서 기존 세션은 멀쩡하다.
- `flush buffer_cache`: 메모리에 남은 블록을 비워 다음 조회가 디스크에서 읽게 만든다.
- 새 세션: 새 서버 프로세스가 파일을 **새로 열 때** 오류가 난다. 실제 장애도 대부분 이렇게 발견된다.

### B-3. 복구

**[oracle $]**
```
sqlplus / as sysdba
```
**[SQL> 루트]** 문제가 된 테이블스페이스만 오프라인으로 내린다.
```
alter session set container = pdb;
alter tablespace lab_ts offline immediate;
exit
```
**[oracle $]**
```
rman target /
```
**[RMAN>]**
```
RESTORE TABLESPACE pdb:lab_ts;
RECOVER TABLESPACE pdb:lab_ts;
exit
```
**실제 화면**
```
channel ORA_DISK_1: restoring datafile 00013 to /oradata/ORCL/pdb/lab_ts01.dbf
channel ORA_DISK_1: reading from backup piece /backup/rman/ORCL_20261031_045690s6_1_1.bkp
channel ORA_DISK_1: restore complete, elapsed time: 00:00:03
Finished restore at 31-OCT-26

starting media recovery
media recovery complete, elapsed time: 00:00:00
Finished recover at 31-OCT-26
```
`pdb:lab_ts`는 CDB 루트에 접속한 RMAN에서 PDB의 테이블스페이스를 가리키는 표기다(`PDB이름:테이블스페이스이름`). 실제로 동작하는 것을 확인했다.

**같은 결과를 내는 다른 방법**: 파일 번호로 지정한다. 번호는 B-1의 `FILE_ID`다. 이 번호는 CDB 전체에서 겹치지 않아 루트에서 그대로 쓸 수 있다.

**[RMAN>]**
```
RESTORE DATAFILE 13;
RECOVER DATAFILE 13;
```
실제로 두 방법을 모두 실행해 보았고, 결과가 같았다. 둘 중 하나만 하면 된다.

**[oracle $]**
```
sqlplus / as sysdba
```
**[SQL> 루트]**
```
alter session set container = pdb;
alter tablespace lab_ts online;
select count(*) from lab.orders;
select seq, note from lab.event_log order by seq;
exit
```
**실제 화면**
```
Tablespace altered.
     10500
       SEQ NOTE
         1 baseline
         2 B: after level0 backup
```
10500행과 seq 2가 모두 있다. **완전 복구, 손실 0**이다.

**[oracle $]** 장애 기록이 닫혔는지 본다.
```
rman target /
```
**[RMAN>]**
```
LIST FAILURE;
exit
```
**실제 화면**
```
no failures found that match specification
```

**명령을 이렇게 쓴 이유**
- `OFFLINE IMMEDIATE`: 파일이 없어 정상 체크포인트를 할 수 없으니 즉시 내린다. ARCHIVELOG 모드에서만 된다.
- RESTORE 다음 RECOVER: 꺼낸 파일은 Level 0 시점의 내용이다. 그 뒤의 로그를 적용해 지금 시점까지 앞으로 감는다.
- DB를 내리지 않는다: 다른 테이블스페이스를 쓰는 업무는 복구하는 동안에도 계속된다. ARCHIVELOG 모드의 가장 큰 장점이다.

**기록 (실측)**

| 항목 | 값 |
|---|---|
| 장애 발견 (`ORA-01116`) | 16:33:23 |
| 복원 시간 | 3초 |
| ONLINE과 확인 | 16:39 무렵 |
| RTO | 약 6분 (다른 방법까지 실행한 시간 포함, 순수 복구는 1~2분) |
| RPO | 0 |

---

## 5. 시나리오 C: 컨트롤 파일 전부 유실

**상황**: 서버가 비정상 종료됐고, 다시 켜 보니 컨트롤 파일 두 개가 모두 없다. 데이터 파일과 리두 로그는 남아 있다.

> **[PC]** 스냅샷: `시나리오C-전`

### C-1. 표시 행 남기기와 장애 만들기

**[oracle $]**
```
sqlplus / as sysdba
```
**[SQL> 루트]**
```
alter session set container = pdb;
insert into lab.event_log (seq, note) values (3, 'C: before controlfile loss');
commit;
alter session set container = cdb$root;
shutdown abort
exit
```
**[oracle $]**
```
ls -l /oradata/ORCL/*.ctl
rm /oradata/ORCL/control01.ctl /oradata/ORCL/control02.ctl
ls -l /oradata/ORCL/*.ctl
sqlplus / as sysdba
```
**[SQL> 루트]**
```
startup
exit
```
**실제 화면**
```
ls: cannot access '/oradata/ORCL/*.ctl': No such file or directory

SQL> startup
ORACLE instance started.
Total System Global Area 1207955552 bytes
ORA-00205: error in identifying control file, check alert log for more info
```
인스턴스는 메모리만 올라간 **NOMOUNT** 상태로 남는다. `shutdown abort`는 전원이 갑자기 꺼진 것과 같은 종료다.

### C-2. 복구

**[oracle $]**
```
ls -l /backup/rman/cf_*
rman target /
```
**실제 화면**
```
-rw-r----- 1 oracle oinstall 18841600 Oct 31 14:48 /backup/rman/cf_c-1775805046-20261031-01
-rw-r----- 1 oracle oinstall 18841600 Oct 31 16:46 /backup/rman/cf_c-1775805046-20261031-02

connected to target database: ORCL (not mounted)
```
자동 백업이 두 개다. `-01`은 2장 Level 0 백업 때, `-02`는 시나리오 B에서 테이블스페이스를 오프라인, 온라인으로 바꾼 **구조 변경 때** 생긴 것으로 보인다.

**[RMAN>]** `<DBID>` 자리에 1-1에서 적은 값을 넣는다.
```
SET DBID <DBID>;
RUN { SET CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/backup/rman/cf_%F'; RESTORE CONTROLFILE FROM AUTOBACKUP; }
ALTER DATABASE MOUNT;
CATALOG START WITH '/backup/arch/' NOPROMPT;
RECOVER DATABASE;
ALTER DATABASE OPEN RESETLOGS;
exit
```
**실제 화면**
```
RMAN> SET DBID 1775805046;
executing command: SET DBID

channel ORA_DISK_1: looking for AUTOBACKUP on day: 20261031
channel ORA_DISK_1: AUTOBACKUP found: /backup/rman/cf_c-1775805046-20261031-02
channel ORA_DISK_1: control file restore from AUTOBACKUP complete
output file name=/oradata/ORCL/control01.ctl
output file name=/oradata/ORCL/control02.ctl

RMAN> ALTER DATABASE MOUNT;
Statement processed

RMAN> CATALOG START WITH '/backup/arch/' NOPROMPT;
no files found to be unknown to the database

RMAN> RECOVER DATABASE;
starting media recovery
archived log for thread 1 with sequence 12 is already on disk as file /oradata/ORCL/redo03.log
archived log file name=/oradata/ORCL/redo03.log thread=1 sequence=12
media recovery complete, elapsed time: 00:00:01

RMAN> ALTER DATABASE OPEN RESETLOGS;
Statement processed
```

**화면 읽는 법**
- RMAN은 **가장 최근 자동 백업(`-02`)**을 골랐다. 자동 백업을 켜 둔 덕분에 구조가 바뀐 뒤의 컨트롤 파일로 복원했다.
- `-02`가 이미 아카이브 로그를 모두 알고 있어 `CATALOG`가 새로 등록할 파일이 없었다.
- `already on disk as file /oradata/ORCL/redo03.log`는 **RMAN이 남아 있는 온라인 리두 로그를 스스로 적용했다**는 뜻이다. 이 로그에 seq 3까지의 변경이 들어 있었다.

**명령을 이렇게 쓴 이유**
- `SET DBID`: 컨트롤 파일이 없으면 RMAN은 DBID를 모른다. 자동 백업 파일 이름(`cf_c-DBID-…`)을 찾으려면 필요하다.
- `RESTORE CONTROLFILE FROM AUTOBACKUP`: 최근 자동 백업을 찾아 `control_files`에 적힌 모든 위치에 복원한다.
- `CATALOG START WITH`: 복원한 컨트롤 파일이 모르는 아카이브 로그가 있으면 폴더를 읽어 등록한다. 없으면 "없다"고만 하고 넘어간다.
- `RECOVER DATABASE`: 아카이브 로그와 남은 온라인 리두 로그까지 적용한다. 리두가 살아 있으니 마지막 커밋까지 복구된다.
- `OPEN RESETLOGS`: 백업한 컨트롤 파일을 썼다면 손실이 없어도 반드시 RESETLOGS로 연다. 로그 번호가 1부터 다시 시작한다.

**RMAN이 `RMAN-06054: media recovery requesting unknown archived log`로 멈췄을 때만** 아래 방법을 쓴다. RMAN이 온라인 리두 로그를 스스로 적용하지 못한 경우다.

**[oracle $]**
```
sqlplus / as sysdba
```
**[SQL> 루트]**
```
recover database using backup controlfile until cancel;
```
1. `Specify log:` 질문이 나오면 `/oradata/ORCL/redo01.log`를 입력한다.
2. `Media recovery complete`가 나올 때까지, 다시 물으면 `redo02.log`, `redo03.log` 순서로 입력한다.
3. 그다음 `alter database open resetlogs;`를 실행한다.

RMAN에서 이미 `OPEN RESETLOGS`까지 성공한 뒤에 이 명령을 실행하면 `ORA-01124: cannot recover data file 1 - file is in use or recovery`가 나고 바로 취소된다. DB가 이미 열려 있다는 뜻일 뿐, 아무것도 바뀌지 않는다.

### C-3. 확인과 새 백업

**[oracle $]**
```
sqlplus / as sysdba
```
**[SQL> 루트]**
```
show pdbs
alter pluggable database all open;
alter session set container = pdb;
select count(*) from lab.orders;
select seq, note from lab.event_log order by seq;
select name from v$tempfile;
exit
```
**실제 화면**
```
         3 PDB                            READ WRITE NO
Pluggable database altered.
     10500
         1 baseline
         2 B: after level0 backup
         3 C: before controlfile loss
/oradata/ORCL/pdb/temp01.dbf
```
seq 3까지 있다. **완전 복구, 손실 0**이다. tempfile은 OPEN할 때 Oracle이 자동으로 다시 만든다.

**[oracle $]**
```
rman target /
```
**[RMAN>]**
```
LIST INCARNATION OF DATABASE;
BACKUP AS COMPRESSED BACKUPSET INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG;
exit
```
**실제 화면**
```
DB Key  Inc Key DB Name  DB ID            STATUS  Reset SCN  Reset Time
1       1       ORCL     1775805046       PARENT  1          17-APR-19
2       2       ORCL     1775805046       PARENT  1920977    30-OCT-26
3       3       ORCL     1775805046       CURRENT 2334776    31-OCT-26

skipping archived logs of thread 1 from sequence 7 to 11; already backed up
skipping datafile 5; already backed up 1 time(s)
skipping datafile 6; already backed up 1 time(s)
skipping datafile 8; already backed up 1 time(s)
...
piece handle=/backup/rman/cf_c-1775805046-20261031-04 comment=NONE
```

**화면 읽는 법**
- 인카네이션 1은 설치 이미지, 2는 DB를 만들 때, **3은 방금 한 RESETLOGS**다.
- `skipping datafile 5, 6, 8`은 PDB$SEED 파일이다. 읽기 전용이라 바뀌지 않았고, 2장에서 켠 **BACKUP OPTIMIZATION** 때문에 다시 백업하지 않았다. 6장에서 이 결과를 다시 만난다.

RESETLOGS 뒤에 Level 0을 바로 다시 받는 이유는, 새 인카네이션이 시작되면 이전 백업만으로 새 인카네이션의 시점을 복구하는 일이 복잡해지기 때문이다. 실무에서도 RESETLOGS 직후 새 전체 백업을 받는다.

**기록 (실측)**

| 항목 | 값 |
|---|---|
| 장애 발견 (`ORA-00205`) | 16:48 |
| OPEN RESETLOGS | 16:50 무렵 |
| RTO | 약 2~3분 |
| RPO | 0 |
| RMAN-06054 발생 | 없음 |

---

## 6. 시나리오 D: 데이터 디스크 전체 유실

**상황**: `/oradata` 디스크가 고장 나 데이터 파일, 컨트롤 파일, 온라인 리두 로그가 모두 사라졌다. 새 디스크를 붙였다고 가정한다. `/backup`(백업과 아카이브 로그)과 `/u01`(엔진과 spfile)은 살아 있다.

**예상 결과**: 온라인 리두 로그까지 잃었으니 **마지막 아카이브 로그까지만** 복구된다(불완전 복구). 그 뒤의 커밋은 잃는다. 이 시나리오의 목적은 **얼마나 잃는지 직접 확인하는 것**이다.

> **[PC]** 스냅샷: `시나리오D-전`

### D-1. 남을 데이터와 잃을 데이터 만들기

**[oracle $]**
```
sqlplus / as sysdba
```
**[SQL> 루트]**
```
alter session set container = pdb;
insert into lab.event_log (seq, note) values (4, 'D: archived - should survive');
commit;
alter session set container = cdb$root;
alter system archive log current;
alter session set container = pdb;
insert into lab.event_log (seq, note) values (5, 'D: only in online redo - will be lost');
commit;
alter session set container = cdb$root;
archive log list
shutdown abort
exit
```
**실제 화면**
```
Database log mode              Archive Mode
Next log sequence to archive   4
Current log sequence           4
ORACLE instance shut down.
```
**Current log sequence 값을 적는다.** (실제 값: 4. 5장의 RESETLOGS 뒤라 번호가 작다)

- seq 4: `archive log current`로 아카이브 로그(`/backup/arch`)에 들어갔다. 복구 뒤 **남는다**.
- seq 5: 온라인 리두 로그(`/oradata`)에만 있다. 복구 뒤 **잃는다**.

### D-2. 장애 만들기

**[oracle $]** 경로를 반드시 다시 확인하고 실행한다.
```
ls /oradata/ORCL/
rm -rf /oradata/ORCL
ls -l /oradata/
```
**실제 화면**
```
control01.ctl  pdb      redo01.log  redo03.log    system01.dbf  undotbs01.dbf
control02.ctl  pdbseed  redo02.log  sysaux01.dbf  temp01.dbf    users01.dbf

total 0
```

**[oracle $]** 새 디스크를 붙였다고 보고 빈 폴더 구조를 다시 만든다.
```
mkdir -p /oradata/ORCL/pdb /oradata/ORCL/pdbseed
ls -l /oradata/ORCL/
```
RMAN은 파일은 만들지만 **없는 상위 폴더까지 만들어 주지는 않는다.** 그래서 폴더를 먼저 만든다.

### D-3. 복구

**[oracle $]**
```
sqlplus / as sysdba
```
**[SQL> 루트]**
```
startup nomount
exit
```
**실제 화면**
```
ORACLE instance started.
Total System Global Area 1207955552 bytes
```
spfile은 `/u01`에 있어 살아 있다.

**[oracle $]**
```
rman target /
```
**[RMAN>]**
```
SET DBID <DBID>;
RUN { SET CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/backup/rman/cf_%F'; RESTORE CONTROLFILE FROM AUTOBACKUP; }
ALTER DATABASE MOUNT;
CATALOG START WITH '/backup/arch/' NOPROMPT;
LIST ARCHIVELOG ALL;
LIST BACKUP SUMMARY;
```
**실제 화면**
```
channel ORA_DISK_1: AUTOBACKUP found: /backup/rman/cf_c-1775805046-20261031-04

List of Files Unknown to the Database
File Name: /backup/arch/1_3_1248108853.dbf
cataloging done

Key     Thrd Seq     S Low Time
1       1    7       A 30-OCT-26    Name: /backup/arch/1_7_1248024313.dbf
...
11      1    12      A 31-OCT-26    Name: /backup/arch/1_12_1248024313.dbf
12      1    1       A 31-OCT-26    Name: /backup/arch/1_1_1248108853.dbf
13      1    2       A 31-OCT-26    Name: /backup/arch/1_2_1248108853.dbf
14      1    3       A 31-OCT-26    Name: /backup/arch/1_3_1248108853.dbf
```

**N 정하기**

아카이브 파일 이름의 마지막 숫자(`_1248…`)가 **인카네이션을 구분하는 번호**다.

| 파일 이름 끝 | 세대 | Seq |
|---|---|---|
| `_1248024313` | 이전 세대 (5장 RESETLOGS 전) | 7~12 |
| **`_1248108853`** | **현재 세대** (5장 RESETLOGS 뒤) | **1, 2, 3** |

- 현재 세대의 가장 큰 Seq가 **N**이다. 실제 값은 **3**이었다.
- D-1에서 적은 Current log sequence(4)보다 1 작다.
- seq 4 행은 3번 아카이브에 들어갔다. `CATALOG`가 새로 찾은 `1_3_…`이 바로 그 파일이다.
- 같은 Seq가 두 번 나오는 줄(실제로는 10, 11)은 이전 세대 기록이 중복 등록된 것이다. 이번 복구와 관계없다.
- `LIST BACKUP SUMMARY`에서 5장 C-3 이후의 Level 0(`LV 0`)이 있는지 본다.

**[RMAN>]** `<N+1>` 자리에 계산한 숫자를 넣는다. 실제로는 N이 3이라 `4`를 넣었다.
```
RUN { SET UNTIL SEQUENCE <N+1> THREAD 1; RESTORE DATABASE; RECOVER DATABASE; }
ALTER DATABASE OPEN RESETLOGS;
exit
```
**실제 화면**
```
RMAN> RUN { SET UNTIL SEQUENCE 4 THREAD 1; RESTORE DATABASE; RECOVER DATABASE; }
executing command: SET until clause

channel ORA_DISK_1: restoring datafile 00005 to /oradata/ORCL/pdbseed/system01.dbf
channel ORA_DISK_1: restoring datafile 00006 to /oradata/ORCL/pdbseed/sysaux01.dbf
channel ORA_DISK_1: restoring datafile 00008 to /oradata/ORCL/pdbseed/undotbs01.dbf
channel ORA_DISK_1: reading from backup piece /backup/rman/ORCL_20261031_055690sv_1_1.bkp
channel ORA_DISK_1: piece handle=... tag=TAG20261031T144639
channel ORA_DISK_1: restore complete, elapsed time: 00:00:25

channel ORA_DISK_1: restoring datafile 00001 to /oradata/ORCL/system01.dbf
...
channel ORA_DISK_1: piece handle=... tag=TAG20261031T171734
channel ORA_DISK_1: restore complete, elapsed time: 00:00:55

channel ORA_DISK_1: restoring datafile 00009 to /oradata/ORCL/pdb/system01.dbf
...
channel ORA_DISK_1: restoring datafile 00013 to /oradata/ORCL/pdb/lab_ts01.dbf
channel ORA_DISK_1: restore complete, elapsed time: 00:00:35
Finished restore at 31-OCT-26

starting media recovery
archived log for thread 1 with sequence 2 is already on disk as file /backup/arch/1_2_1248108853.dbf
archived log for thread 1 with sequence 3 is already on disk as file /backup/arch/1_3_1248108853.dbf
media recovery complete, elapsed time: 00:00:01

RMAN> ALTER DATABASE OPEN RESETLOGS;
Statement processed
```

**화면 읽는 법**
- 데이터 파일 12개를 3묶음으로 복원했다. 합계 약 2분(25초, 55초, 35초)이다.
- **PDB$SEED(5, 6, 8번)는 14:46 백업(`TAG…144639`, 이전 세대)에서 복원됐다.** 5장의 새 Level 0이 BACKUP OPTIMIZATION 때문에 SEED를 건너뛰었기 때문이다. RMAN은 RESETLOGS 이전 세대의 백업도 문제없이 골라 썼다.
- 나머지 루트와 PDB 파일은 5장 뒤의 새 Level 0(`TAG…171734`)에서 복원됐다. 그래서 적용할 로그가 seq 2와 3뿐이었고 복구가 1초 만에 끝났다.

**명령을 이렇게 쓴 이유**
- `startup nomount`: 컨트롤 파일이 없으니 인스턴스(메모리와 프로세스)만 올린다. spfile만 있으면 된다.
- `SET UNTIL SEQUENCE N+1`: "N+1번 로그 **직전**까지" 복구한다는 뜻이다. N+1번은 온라인 리두 로그였고 디스크와 함께 사라졌다.
- 이 줄 없이 `RECOVER DATABASE`만 하면 끝에 `RMAN-06054: media recovery requesting unknown archived log … sequence N+1`이 난다. 이 시나리오에서는 예상된 메시지다. 그대로 `OPEN RESETLOGS`로 열면 결과는 같다.
- `OPEN RESETLOGS`: 불완전 복구 뒤에는 반드시 필요하다. 리두 로그 파일도 이때 새로 만든다.

### D-4. 확인과 새 백업

**[oracle $]**
```
sqlplus / as sysdba
```
**[SQL> 루트]**
```
show pdbs
alter pluggable database all open;
alter session set container = pdb;
select count(*) from lab.orders;
select seq, note from lab.event_log order by seq;
select name from v$tempfile;
exit
```
**실제 화면**
```
         3 PDB                            READ WRITE NO
     10500
         1 baseline
         2 B: after level0 backup
         3 C: before controlfile loss
         4 D: archived - should survive
/oradata/ORCL/pdb/temp01.dbf
```
**seq 4는 있고 seq 5는 없다.** 마지막 아카이브 로그까지 살렸고, 온라인 리두 로그에만 있던 커밋 1건을 잃었다. 예상과 정확히 같다.

**[oracle $]**
```
rman target /
```
**[RMAN>]**
```
LIST INCARNATION OF DATABASE;
BACKUP AS COMPRESSED BACKUPSET INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG;
exit
```
**실제 화면**
```
3       3       ORCL     1775805046       PARENT  2334776    31-OCT-26
4       4       ORCL     1775805046       CURRENT 2338779    31-OCT-26
...
piece handle=/backup/rman/cf_c-1775805046-20261031-06 comment=NONE
```
인카네이션 4가 CURRENT가 됐다.

**기록 (실측)**

| 항목 | 값 |
|---|---|
| startup nomount | 17:35:16 |
| 데이터 파일 복원 | 약 2분 (12개) |
| OPEN RESETLOGS | 17:39 무렵 |
| RTO | 약 4~5분 (nomount부터) |
| RPO | 커밋 1건 (seq 5) |

**이 결과로 설명할 수 있는 것**
- 리두 로그를 **다른 디스크에 다중화(멤버 추가)**했다면 리두가 살아남아 완전 복구가 됐다.
- `archive_lag_target`으로 아카이브 주기를 짧게 하면 잃는 범위가 줄어든다.
- "RPO를 줄이려면 무엇을 바꿔야 하나"에 대한 근거가 된다.

---

## 7. 백업 자동화

### 7-1. 백업 스크립트

**[oracle $]** 블록 전체를 한 번에 붙여 넣는다.
```
cat > /backup/scripts/rman_backup.sh <<'EOF'
#!/bin/bash
# 사용법: rman_backup.sh 0   (Level 0, 주 1회)
#         rman_backup.sh 1   (Level 1, 매일)
export ORACLE_HOME=/u01/app/oracle/product/19.0.0.0/dbhome_1
export ORACLE_SID=orcl
export PATH=$ORACLE_HOME/bin:$PATH
export NLS_DATE_FORMAT='YYYY-MM-DD HH24:MI:SS'
LEVEL=${1:-1}
LOG=/backup/logs/rman_L${LEVEL}_$(date +%Y%m%d_%H%M).log
rman target / log="$LOG" <<RMAN
BACKUP AS COMPRESSED BACKUPSET INCREMENTAL LEVEL ${LEVEL} DATABASE PLUS ARCHIVELOG;
DELETE NOPROMPT OBSOLETE;
CROSSCHECK BACKUP;
CROSSCHECK ARCHIVELOG ALL;
RMAN
if grep -qE "RMAN-[0-9]+|ORA-[0-9]+" "$LOG"; then echo "BACKUP FAILED: $LOG"; exit 1; fi
echo "BACKUP OK: $LOG"
find /backup/logs -name 'rman_*.log' -mtime +30 -delete
EOF
chmod 750 /backup/scripts/rman_backup.sh
/backup/scripts/rman_backup.sh 1
```
**실제 화면**
```
RMAN> RMAN> RMAN> RMAN> BACKUP OK: /backup/logs/rman_L1_20261101_0030.log
```
붙여 넣을 때 화면에 입력 줄과 결과가 섞여 보일 수 있다. 아래 7-3에서 파일 내용을 직접 확인한다.

**줄마다 넣은 이유**
- `export ORACLE_HOME …`: cron은 `.bash_profile`을 읽지 않는다. 환경 변수를 스크립트 안에서 정해야 한다.
- Level 1: Level 0 이후 **바뀐 블록만** 백업한다. 실제로 40초 만에 끝났다.
- `DELETE OBSOLETE`: 보관 정책(7일)보다 오래된 백업을 지워 디스크가 차지 않게 한다.
- `CROSSCHECK`: 기록에는 있지만 디스크에서 사라진 백업이 있는지 대조한다.
- `grep RMAN-, ORA-`: 실패를 한 줄로 알 수 있게 한다. 운영에서는 이 줄을 메일이나 메신저 알림으로 바꾼다.

### 7-2. cron 등록

**[oracle $]**
```
(crontab -l 2>/dev/null; echo "30 2 * * 0 /backup/scripts/rman_backup.sh 0 >> /backup/logs/cron.log 2>&1"; echo "30 2 * * 1-6 /backup/scripts/rman_backup.sh 1 >> /backup/logs/cron.log 2>&1") | crontab -
crontab -l
```
**실제 화면**
```
30 2 * * 0 /backup/scripts/rman_backup.sh 0 >> /backup/logs/cron.log 2>&1
30 2 * * 1-6 /backup/scripts/rman_backup.sh 1 >> /backup/logs/cron.log 2>&1
```
일요일 02:30에 Level 0, 월요일부터 토요일 02:30에 Level 1을 받는다. 시각은 서버 시간대 기준이다. **그 시각에 VM이 켜져 있어야** 돈다.

### 7-3. 확인

**[oracle $]**
```
cat /backup/scripts/rman_backup.sh
grep -E "Finished backup|Finished Control|RMAN-|ORA-" /backup/logs/rman_L1_*.log
df -h /backup
rman target /
```
**[RMAN>]**
```
LIST BACKUP SUMMARY;
REPORT NEED BACKUP;
exit
```
**[root #]**
```
systemctl is-active crond
```
**실제 화면**
```
#!/bin/bash
...
find /backup/logs -name 'rman_*.log' -mtime +30 -delete

Finished backup at 2026-11-01 00:30:11
Finished backup at 2026-11-01 00:30:41
Finished backup at 2026-11-01 00:30:42
Finished Control File and SPFILE Autobackup at 2026-11-01 00:30:46

/dev/mapper/VGDb-LVDbBackup  100G  2.8G   98G   3% /backup

22      B  1  A DISK        01-NOV-26       1       1       YES        TAG20261101T003011
23      B  1  A DISK        01-NOV-26       1       1       YES        TAG20261101T003011

active
```

**정상이면**
- `cat`: `#!/bin/bash`부터 `find …`까지 빠진 줄이 없다.
- `grep`: `Finished` 줄만 있고 `RMAN-`, `ORA-` 줄이 없다.
- `LIST BACKUP SUMMARY`: `LV`가 `1`인 줄(Level 1)이 생겼다.
- `crond`: `active`가 나온다.
- `cron.log`: **첫 cron 실행 전에는 없다.** 첫 실행 다음 날 `tail -3 /backup/logs/cron.log`에 `BACKUP OK`가 보이면 자동화까지 검증된 것이다.

---

## 8. 결과 정리 (실측)

| 시나리오 | 장애 | 복구 방법 | 복구 종류 | RTO | RPO |
|---|---|---|---|---|---|
| A-1 | 잘못된 DELETE 1000행 | Flashback Query와 INSERT | 논리 복구 | 5분 이내 | 0 |
| A-2 | DROP TABLE | Flashback Table, 인덱스와 제약 조건 이름 복원 | 논리 복구 | 1분 이내 | 0 |
| B | 데이터 파일 1개 삭제 | OFFLINE, RESTORE와 RECOVER TABLESPACE, ONLINE | 완전 복구, DB 운영 유지 | 약 6분 (순수 1~2분) | 0 |
| C | 컨트롤 파일 2개 삭제 | 자동 백업에서 복원, 온라인 리두 적용, RESETLOGS | 완전 복구 | 약 2~3분 | 0 |
| D | `/oradata` 전체 삭제 | 컨트롤 파일과 DB 전체 복원, UNTIL SEQUENCE, RESETLOGS | 불완전 복구 | 약 4~5분 | 커밋 1건 |

RTO는 로그에 남은 시각으로 계산한 대략값이다.

| 백업 구성 | 값 |
|---|---|
| 첫 Level 0 | 약 2분, 4.7GB → 660MB |
| Level 1 | 약 40초 |
| 주기 | 일요일 Level 0, 월~토 Level 1 |
| 보관 | 7일, 컨트롤 파일 자동 백업 켬, 백업 검증(`RESTORE VALIDATE`) |
| 실습 뒤 서버 점검 | PASS 17, FAIL 0 |

---

## 9. 면접 대비 질문

**RESTORE와 RECOVER의 차이는?**
RESTORE는 백업에서 파일을 꺼내 놓는 것(백업 시점)이고, RECOVER는 로그를 적용해 원하는 시점까지 앞으로 감는 것이다.

**NOARCHIVELOG면 시나리오 B를 할 수 있나?**
못 한다. 백업 이후의 변경을 되살릴 로그가 없다. DB 전체를 백업 시점으로 되돌리는 것만 되고, 그것도 DB를 내린 상태여야 한다.

**컨트롤 파일만 잃었는데 왜 RESETLOGS를 하나?**
복원한 컨트롤 파일은 백업 시점의 것이라 로그 정보가 지금과 맞지 않는다. 손실이 없어도 로그 순서를 새로 시작해 맞춘다.

**C와 D의 결과가 다른 이유는?**
C는 온라인 리두가 살아 있어 마지막 커밋까지 복구했다. D는 리두까지 잃어 마지막 아카이브까지만 살렸다.

**D에서 데이터를 잃지 않으려면?**
온라인 리두 로그를 다른 디스크에 다중화한다. 아카이브를 자주 만들거나 다른 서버로 보낸다(Data Guard).

**BACKUP OPTIMIZATION이 복원에 어떤 영향을 주었나?**
바뀌지 않은 PDB$SEED를 새 백업에서 건너뛰었다. 그래서 D에서 SEED는 이전 세대 백업에서 복원됐다. RMAN은 세대를 건너 백업을 골랐다.

**백업이 정말 쓸 수 있는지 어떻게 아나?**
`RESTORE DATABASE VALIDATE`로 정기 검증하고, 실제 복구 리허설을 한다. 이 실습이 그 리허설이다.

**DROP한 테이블을 되살린 뒤 확인할 것은?**
인덱스와 제약 조건 이름이 `BIN$`으로 남는다. 원래 이름으로 직접 되돌린다.

---

## 10. 원래대로 되돌리기 (선택)

실습용 객체를 지우려면 실행한다. 다음 실습에 계속 쓸 거라면 남겨 둔다.

**[oracle $]**
```
sqlplus / as sysdba
```
**[SQL> 루트]**
```
alter session set container = pdb;
drop user lab cascade;
drop tablespace lab_ts including contents and datafiles;
exit
```
cron을 끄려면 [oracle $]에서 `crontab -e`로 해당 두 줄만 지운다. 다른 작업이 없으면 `crontab -r`로 전부 지워도 된다.
