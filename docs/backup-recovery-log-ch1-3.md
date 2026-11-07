# Oracle 19c 백업 · 복구 실습 실행 기록: 1~3장 (2026-10-31)

> 「Oracle19c_백업복구_실습가이드」 1~3장을 실행한 기록입니다.
> 단계마다 가이드의 기대값과 실제 결과를 비교하고, 실행 로그(직접 실행한 결과)로 검증했습니다.
> 추정은 **(추정)**으로 표시했습니다.

---

## 1. 결과 요약

| 구분 | 결과 |
|---|---|
| 대상 | `dbts93` · Oracle 19.3 · CDB `ORCL` + PDB `PDB` · ARCHIVELOG |
| **DBID** | **1775805046** ← 5 · 6장에서 사용 |
| 진행 범위 | 1장 사전 점검 · 2장 RMAN 설정과 Level 0 백업 · 3장 시나리오 A (DELETE · DROP) |
| 판정 | ✅ **1~3장 모두 통과**. 가이드와 다르게 나온 곳 4건, 내 입력 실수 3건은 모두 데이터에 영향 없이 정리됨 |
| 다음 단계 | 스냅샷 `시나리오B-전` → 4장 시나리오 B. 데이터 파일 번호는 **13** |

---

## 2. 단계별 검증

### 1장 사전 점검 · 실습 데이터

| 단계 | 기대값 | 실제 (시각) | 판정 |
|---|---|---|---|
| 1-1 DB 상태 | ARCHIVELOG · CDB YES · PDB READ WRITE | `ORCL 1775805046 ARCHIVELOG YES` · PDB `READ WRITE` · Current log sequence 10 (14:27) | ✅ |
| 1-1 파일 위치 | 컨트롤 파일 2개 · 리두 3개 | `/oradata/ORCL/control01.ctl`, `control02.ctl` · `redo01~03.log` | ✅ 가이드 기준 값과 같음 |
| 1-1 공간 | `/backup` 20GB 이상 | `/oradata` 4.7G 사용 · `/backup` **99G 여유** | ✅ |
| 1-2 폴더 | 세 폴더 `oracle oinstall` | `/backup/logs` · `rman` · `scripts` 모두 `drwxr-xr-x oracle oinstall` (14:31) | ✅ |
| 1-3 PDB 확인 | `show con_name` → `PDB` | `PDB` | ✅ |
| 1-3 실습 객체 | 테이블스페이스 · 사용자 · 테이블 2개 | `Tablespace created` · `User created` · `Grant succeeded` · `Table created` ×2 | ✅ |
| 1-3 데이터 | `10000` · `1 baseline` | `10000` · `1 baseline 2026-10-31 14:40:13` | ✅ |

- **입력 실수**: `select member form v$logfile;` → `ORA-00923`. `from`을 `form`으로 친 오타이고, 바로 고쳐서 다시 실행했습니다. 영향 없음.

### 2장 RMAN 설정 · Level 0 백업

| 단계 | 기대값 | 실제 (시각) | 판정 |
|---|---|---|---|
| 2-1 접속 | DBID가 1-1과 같음 | `connected to target database: ORCL (DBID=1775805046)` (14:41) | ✅ |
| 2-1 설정 6개 | 각각 `successfully stored` | 6개 모두 저장, 마지막 `SHOW ALL`에서 `# default` 표시 사라짐 | ✅ |
| 2-2 Level 0 | `Finished Control File and SPFILE Autobackup` | 14:46:24 시작 → 14:48:16 끝 · `cf_c-1775805046-20261031-01` | ✅ **약 2분** |
| 2-2 백업 대상 | 루트 · PDB$SEED · PDB 전체 + 아카이브 | 루트 4개(파일 1·3·4·7) · PDB 5개(9~13, **`lab_ts01.dbf` = 13번** 포함) · SEED 3개(5·6·8) · 아카이브 seq 7~11 | ✅ |
| 2-2 `LIST BACKUP SUMMARY` | Level 0(`LV 0`)과 아카이브(`A`) | 7줄: `A` 2개, `0` 3개, `F` 2개 | ✅ (아래 설명) |
| 2-2 `REPORT NEED BACKUP` | 빈 표 | 빈 표 | ✅ |
| 2-2 `RESTORE DATABASE VALIDATE` | 에러 없이 `Finished restore` | 데이터 백업 조각 3개 모두 `validation complete` | ✅ |
| 2-2 파일 | `.bkp` 여러 개 + `cf_c-…` | `.bkp` 5개 + `cf_c-1775805046-20261031-01`, 합계 **660M** | ✅ |

**압축 효과**: `/oradata` 사용량 4.7G → 백업 660M (약 7분의 1)

**가이드와 다르게 나온 곳**
1. **19c의 `CONTROLFILE AUTOBACKUP` 기본값이 이미 `ON`**이었습니다. 처음 `SHOW ALL`에서 `ON; # default`로 보였습니다. 가이드에 이 설명을 추가했습니다.
2. **`LIST BACKUP SUMMARY`에 `F`(컨트롤 파일 · spfile 자동 백업)가 2줄**이었습니다.
   - Key 7 (14:48:16): Level 0이 끝나며 만든 자동 백업입니다. `/backup/rman/cf_c-…-01`
   - **Key 1 (14:41:02, `Compressed NO`)**: 1-3에서 테이블스페이스를 만든 **구조 변경 때문에** 자동으로 생긴 백업입니다 **(추정)**. 2-1에서 형식을 바꾸기 전이라 기본 위치(`$ORACLE_HOME/dbs/c-1775805046-20261031-00`)에 있을 것으로 보입니다. 이름의 끝 번호가 `-00`과 `-01`로 이어지는 것도 이 추정과 맞습니다.
   - 가이드에 설명을 추가했습니다.
   - 확인 명령 **[oracle $]**: `ls -l $ORACLE_HOME/dbs/c-1775805046-*`
3. **lab_ts의 데이터 파일 번호가 13**입니다. 가이드 4장 B-3의 대안 예시(14)를 **13**으로 고쳤습니다.

### 3장 시나리오 A: 사용자 실수

| 단계 | 기대값 | 실제 (시각) | 판정 |
|---|---|---|---|
| T1 기록 | 시각 출력 | **T1 = 2026-10-31 15:44:54** | ✅ |
| A-1 실수 | `9000` | `1000 rows deleted` → `9000` | ✅ |
| A-1 과거 조회 | `10000` | `as of timestamp T1` → `10000` | ✅ |
| A-1 되살리기 | `10000` | `1000 rows created` → `10000` | ✅ |
| A-2 DROP | `ORA-00942` | `Table dropped` (15:50:23) → `ORA-00942` | ✅ |
| A-2 휴지통 | ORDERS(TABLE) + 인덱스(INDEX) | `BIN$…XK…` = `SYS_C007550`(INDEX, 15:50:22) · `BIN$…XL…` = `ORDERS`(TABLE, 15:50:23) | ✅ |
| A-2 되살리기 | `Flashback complete` → `10000` | 같음 | ✅ |
| A-2 이름 확인 | 인덱스 · 제약 조건이 `BIN$` | 인덱스 `BIN$…XK…` · 제약 조건 `BIN$…XJ…` | ✅ |
| A-2 이름 되돌리기 | 원래 이름 | `Index altered` · `Table altered` → 둘 다 **`SYS_C007550`** (16:21) | ✅ |

**결과**: 잃은 데이터 **0행**

| 기록 | 값 |
|---|---|
| DELETE 복구 걸린 시간 | 약 5분 이내 (15:44:54 T1 기록 → 15:50 DROP 시작 전 복구 완료) |
| DROP 복구 걸린 시간 | 테이블 · 데이터: 1분 이내 · 이름 정리까지: 약 30분 (가이드 오류로 지연) |
| 잃은 데이터 | 0행 |

**가이드 오류 (수정 완료)**
1. **이름 확인 조회문의 `ORA-00918: column ambiguously defined`**
   - 원인: 두 뷰에 모두 `index_name` 열이 있는데 `select index_name`으로 써서 어느 쪽인지 정하지 못했습니다.
   - 수정: `select i.index_name, c.constraint_name …`
2. **인덱스 이름 되돌리기에서 `ORA-01418`**
   - 원인: 휴지통의 TABLE 줄 이름(`…XL…`)을 넣었습니다. 테이블은 이미 `ORDERS`로 돌아와 그 이름이 없습니다. 가이드 설명이 어느 줄을 쓰는지 분명하지 않았습니다.
   - 수정: "INDEX 줄의 BIN$ 이름을 쓴다", "새 이름은 ORIGINAL_NAME"을 명시하고, 제약 조건 이름을 되돌리는 명령을 추가했습니다.
3. **자리 표시 `①의_CONSTRAINT_NAME`을 그대로 실행해 `ORA-23292`**
   - 수정: 자리 표시를 `<…>`로 바꾸고 "꺾쇠까지 지우고 실제 값으로 바꾼다"는 경고를 넣었습니다.

**입력 실수 (영향 없음)**
- `to_timestamp('2026-10-31 15:44:54, '…` 닫는 따옴표 누락 → `ORA-01756`. 고쳐서 다시 실행했습니다. 가이드에 따옴표 확인 안내를 추가했습니다.

**알게 된 것**
- DROP하면 **인덱스가 먼저(15:50:22), 테이블이 나중(15:50:23)**에 휴지통으로 갑니다.
- 테이블 · 인덱스 · 제약 조건의 `BIN$` 이름은 서로 다릅니다(`…XL…` · `…XK…` · `…XJ…`). 끝에서 몇 번째 글자 하나만 달라서 헷갈리기 쉽습니다.
- Flashback Table은 테이블 이름만 되돌립니다. **인덱스와 제약 조건 이름은 DBA가 직접 되돌려야 합니다.**

---

## 3. 이전 기록 보완 (10-30 수동 설치)

제출 로그 앞부분에 10-30 PC 종료 뒤의 확인이 있었습니다.

| 항목 | 결과 | 해석 |
|---|---|---|
| alert log `crash recovery` 검색 | `Crash Recovery excluding pdb 2 / pdb 3 which was cleanly closed.` 5줄 | 인스턴스 복구(크래시 복구)가 **몇 차례 실제로 있었습니다.** 깨끗하게 닫혀 있던 PDB는 제외하고 루트만 복구했다는 뜻입니다. 시각이 같이 찍히지 않아 PC 종료 때의 것인지는 확인되지 않았습니다 **(추정 불가)** |

확인 명령 (선택, **[oracle $]**). 각 줄 앞의 시각이 함께 나옵니다.
```
grep -B3 "Crash Recovery excluding" $ORACLE_BASE/diag/rdbms/orcl/orcl/trace/alert_orcl.log | grep -E "^20[0-9]{2}-|Crash"
```

---

## 4. 다음: 시나리오 B 시작 전 확인

| 확인 | 값 |
|---|---|
| 스냅샷 | `시나리오B-전` 찍기 |
| lab.orders 행 수 | 10000 (B-1에서 500행 추가 → 10500) |
| event_log | seq 1 (B-1에서 seq 2 추가) |
| lab_ts 파일 번호 | **13** (`/oradata/ORCL/pdb/lab_ts01.dbf`) |
| 최신 백업 | Level 0, 2026-10-31 14:46~14:48 |
| DBID | 1775805046 |
