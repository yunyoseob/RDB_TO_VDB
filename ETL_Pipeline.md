# ETL Pipeline

## Source DB

Relational Database: PostgreSQL

## Target DB

Vector Database: Elasticsearch

## ETL Management Schema

1. Source DB Tables (수집 대상 RDB 테이블)
2. common_code (공통 테이블)
3. vdb_load_mm_meta (적재 메타 기본 테이블)
4. vdb_load_md_meta (적재 메타 상세 테이블)
5. vdb_load_status (적재 상태 테이블)
6. vdb_load_err (적재 오류 테이블)

## ETL Process Task 1: Validation

Source DB - Target DB meta validation

common_code에서 Source DB 정보와 Target DB 정보 매핑

**매핑 과정에서 테이블 매핑, 컬럼 매핑 등 검증 작업**

### 1. 적재 메타 기본 테이블에서 수행해야 하는 DAG를 확인

|DAG|DAG 설명|
|--|--|
|LoadDAG|증분값 추출 수집-적재 작업 (Insert from RDB to VDB) |
|SyncDAG|수집 - 적재 데이터 싱크 작업 (Update & Delete)|
|ReLoadDAG|적재 오류 데이터 재적재 (Error Data Reload)|

예시)

**vdb_load_mm_meta**

|id|dag name|status|Last Execute Time|is_success|
|--|--|--|--|--|
|1|LoadDAG|Stop|2025-03-23 00:00:00|True|
|2|SyncDAG|Stop|2025-03-23 00:00:00|True|
|3|ReLoadDAG|Stop|2025-03-23 00:00:00|True|


### 2. 적재 수행 예시: 적재 메타 기본 테이블에서 status 컬럼을 Start로 Update 

예시)

**vdb_load_mm_meta**

|id|dag_name|status|Last Execute Time|is_success
|--|--|--|--|--|
|1|LoadDAG|Start|2025-03-24 00:00:00|False|

### 3. 적재 메타 상세 테이블을 통한 Source DB - Target DB 필드 검증 및 공통 코드에서 Source - Target의 정의가 제대로 되어 있는지 확인

예시)

**vdb_load_md_meta**

|id|dag_name|source_db|source_table_name|source_column_name|target_db|target_table_name|target_column_name|use_yn|creat_at|update_at|author|
|--|--|--|--|--|--|--|--|--|--|--|--|
|1|LoadDAG|RDB|doc|doc_text|VDB|doc|doc_text|True|2025-03-24 00:00:00|2025-03-24 00:00:00|000000|
|2|LoadDAG|RDB|doc|doc_text|VDB|doc|doc_vector|True|2025-03-24 00:00:00|2025-03-24 00:00:00|000000|

### 4. common_code 각 테이블에서 offset 정보 조회

**Source DB Table offset 조회**

```sql
select code_group_name, code_name
  from common_code
 where code_info = 'postgre_sql'
   and code_group_id = 'table_name'
   and code_id = 'current_offset'
```

|code_group_name|code_name|
|--|--|
|doc|2025-03-24 00:00:00.000|

모든 검증이 끝나면, vdb_load_status 테이블에 해당 과정 "validation" 완료 된 내용을 업데이트

검증 과정 오류 시, vdb_load_status 테이블에 해당 과정 "validation" 실패 된 내용을 Update 및 vdb_load_err 테이블에 오류 내용 Insert

## ETL Process Task 2: Extract from Source DB

common_code -> Get Source DB Tables offset

vdb_load_md_meta -> Get Source DB - Target DB Meta infomation

Source DB Table -> Document Craking (Data)

```sql
select doc_name, doc_text, load_time
  from doc
 where load_time > '2025-03-23 00:00:00.000'
```

모든 검증이 끝나면, vdb_load_status 테이블에 해당 과정 "extract" 완료 된 내용을 업데이트

검증 과정 오류 시, vdb_load_status 테이블에 해당 과정 "extract" 실패 된 내용을 Update 및 vdb_load_err 테이블에 오류 내용 Insert

## ETL Process Task 3: Transform Raw Data

Chunking, Embedding 등의 작업 수행

모든 검증이 끝나면, vdb_load_status 테이블에 해당 과정 "transform" 완료 된 내용을 업데이트

검증 과정 오류 시, vdb_load_status 테이블에 해당 과정 "transform" 실패 된 내용을 Update 및 vdb_load_err 테이블에 오류 내용 Insert

## ETL Process Task 4: Load Data

Target DB에 Insert 작업 수행

모든 검증이 끝나면, vdb_load_status 테이블에 해당 과정 "load" 완료 된 내용을 업데이트

검증 과정 오류 시, vdb_load_status 테이블에 해당 과정 "load" 실패 된 내용을 Update 및 vdb_load_err 테이블에 오류 내용 Insert