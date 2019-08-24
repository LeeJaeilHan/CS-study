# 오라클 스터디

## Table

### Schema Object

- Schema : Schema Object라는 Data Structure를 담는 논리적인 컨테이너
- Schema Object 예시: Table, Index, Partition, View, Sequence, Dimension, Synonym, PL/SQL

### Schema Object Storage

- Schema Object는 segment라는 logical storage structure로 저장된다.
- Tablespace와 Schema는 관계가 없다.

### Table Overview

- table은 Oracle DB의 기본 data 조직 단위
- Relation table과 Object table로 나눈다
- Object table은 각 row가 object를 나타낸다
- Relation table의 조직 성격에 따라
  - heap-organized table(default) : unordered
  - Index-organized table : PK 기준으로 row 정렬
  - external table : readonly
- Column
  - virtual : disk space 사용하지 않음
  - invisible : 이름으로 명시하기 전까지 보이지 않음
- Row
  - Data block에 저장된다
  - row별로 10byte physical address인 rowid가 있다
  - rowid는 index 구성에 사용된다

### Data Type

- Character
  - VARCHAR2 - 가변길이
  - CHAR - 고정길이
- Numeric
  - NUMBER
  - BINARY_FLOAT
  - BINARY_DOUBLE
- Datatime
  - DATE
  - TIMESTAMP
- RowID
  - pseudoColumn
- Format Model

### Integrity Constraints

- Data Dictionary에 저장됨

### Table Compression

- table compression은 database application에 transparent하다(손실압축 아니여서 압축을 알아 챌 수 없다)
- transparent = 있는데 없는 것 같이 보임(투명하여 보이지 않음)
- Oracle DB가 지원하는 dictionary based Compression
  - Basic table compression => bulk load operation
  - Advanced row compression => OLTP application
- 압축 level
  - table, partition, subpartition
- Hyper Columnar Compression 종류
  - Warehouse
  - Archieve
  - row의 set을 저장할 때 Compression Unit 사용한다 

## Table Clusters

- 공통column을 공유하고 같은 block에 연관된 data를 저장하는 table 그룹
- cluster key : 공통 column
- 장점 : Disk I/O, access time, less storage

### Index Cluster

- cluster key값이 같은 레코드가 한 블록에 모이도록 저장하는 구조
  - cluster key는 unique해야 한다
- 클러스터 인덱스를 스캔하면서 값을 찾을 때는 Random access가 값 하나당 한번씩만 발생한다
- 한 블럭에 모두 담을 수 없으면 클러스터 체인으로 연결한다
- ![image-20190823020025319](/Users/lji/Library/Application Support/typora-user-images/image-20190823020025319.png)
- 넓은 범위를 검색할 때 유리하다 -> 클러스터에 도달해서 sequential 방식으로 스캔하기 때문
- 실무에서 자주 사용하지 않는 이유는 DML 부하
  - 정해진 블록을 찾아서(정렬 유지) 값을 입력해야 하기 때문에
  - Truncate Table을 클러스터에서 사용할 수 없다

### Hash Cluster

- 클러스터 키로 데이터 검색하고 저장할 위치를 찾을 때 hash function을 사용한다
- hash 함수가 인덱스 역할을 대신한다.
- 물리적인 인덱스를 따로 가지고 있지 않으므로 single I/O 만 발생한다

### Attribute-Clustered Table

- I/O reduction에 강점

#### Zone

- 적절한 column들의 min, max값을 저장한 연속적인 data block의 set

#### Interleaving order

- data warehouse 내부 dimensional 계층에 유용하다

### Temporary Table

- transaction, session 동안만 data를 잡고 있는 table

### External Table

- 외부 소스 접근을 DB table에 접근하듯 하는 table
- Readonly



## Index

- table row 접근 속도를 높히는 schema object
- 인덱스를 생성하는 것이 좋은 column
  - ① WHERE절이나 join조건 안에서 자주 사용되는 컬럼
  - ② null 값이 많이 포함되어 있는 컬럼
  - ③ WHERE절이나 join조건에서 자주 사용되는 두 개이상의 컬럼들
- 인덱스 불필요한 경우
  - ① 테이블이 작을 때
  - ② 테이블이 자주 갱신될 때
- Usability : unusable <- optimizer
- Visibility : invisible index <- DML operators
- Key는 논리적인 개념, Index는 DB에 저장되는 구조(schema object)

### Composite index

- table의 여러 column 기반 index
- 선언할 때 column의 순서가 중요하다

```sql
CREATE UNIQUE INDEX emp_empno_ename_indx 
     ON emp(empno, ename);     
```

### Unique/ Nonunique Index

- nonunique index는 index key와 rowid로 정렬된다(오름차순)

```sql
CREATE UNIQUE INDEX emp_ename_indx 
     ON emp(ename); 
CREATE INDEX  dept_dname_indx 
     ON dept(dname);     
```

### Index Storage

- index segment에 index data가 저장된다

### B-Tree Index

- Branch Block -> searching
- Leaf Block -> key value 저장



### Index Scan

- Full Index Scan
  - 인덱스 리프 블록을 수평적으로 탐색하는 방식
  - 최적의 인덱스가 없는 경우 차선으로 선택
  - table full scan보다 I/O줄일 수 있거나, 정렬된 결과를 쉽게 얻을 수 있는 경우
- Fast Full Index Scan
  - 인덱스 트리 구조를 무시하고, 인덱스 세그먼트 전체를 multiblock read 방식으로 스캔한다
  - 물리적으로 디스크에 저장되어 있는 순서대로 인덱스 블록을 읽는다(인덱스 키 순서대로 정렬된 결과을 얻지는 못한다)
  - 인덱스가 파티션 되어 있지 않아도 병렬 쿼리 가능하다
- Index Range Scan
  - 루트 노드에서 리프 노드까지 수직적으로 탐색한 후, 리프 블록을 필요한 범위만 스캔하는 방식
  - index를 구성하는 선두 column이 조건절에 사용되어야 index range scan 사용 가능하다
- Index Unique Scan
  - 수직적 탐색만으로 데이터를 탐색하는 방식
- Index Skip Scan
  - Root 또는 브랜치 블록에서 읽은 컬럼 값 정보를 이용해 조건에 부합하는 레코드를 포함할 가능성이 있는 리프 블록만 골라서 액세스하는 방식
  - Distinct value 개수가 많을수록 유용하다

### Index Clustering Factor(CF)

- clustering_factor 수치가 테이블 블록(table_blocks)에 가까울수록 데이터가 잘 정렬돼 있음을 의미함. CLUSTERING_FACTOR 수 만큼 Random I/O를 발생하면 전체 데이터를 액세스 할 수 있음
- 레코드 개수(num_rows)에 가까울수록 흩어져 있음을 의미함.
- cluster factor가 높으면 I/O 접근 횟수가 많아진다

B-Tree Index 종류

- Reverse key index -> Range search 힘들다
- Descending index -> DESC

### Index Compression

- prefix compression(key compression)
- Index entry는 2조각을 가진다
  - grouping 조각 : prefix
  - unique 조각 : suffix
- Advanced Index Compression
  - 

### Bitmap Index

- 키 값에 중복이 없고, 키 값별로 하나의 비트맵 레코드를 가진다
- 비트맵을 저장하기 위해 2개 이상의 블록이 필요하면 B* Tree 인덱스 구조를 사용한다
- 수정/변경이 잘 일어나지 않는 경우 좋다

```sql
CREATE BITMAP INDEX emp_deptno_indx 
     ON emp(deptno); 
```

### Function-Based Index

- 테이블의 column을 가공한 값을 인덱스로 사용
- 테이블 설계상 문제를 해결
- 오류데이터 검색문제를 해결

#### 인덱스 삭제 시

```sql
DROP INDEX emp_empno_ename_indx; 
```

#### 인덱스 데이터사전

```sql
SELECT index_name, index_type 
     FROM USER_INDEXES 
     WHERE table_name='EMP'; 
```

## Index-Organized Table(IOT)

![image-20190823011541660](/Users/lji/Library/Application Support/typora-user-images/image-20190823011541660.png)

- Random access가 발생하지 않도록 인덱스 구조로 생성된 테이블 -> sequnce 방식으로 데이터 엑세스 -> 넓은 범위 엑세스 할 경우 유리하다
- 대량의 데이터를 처리할 때 Random access는 가장 큰 부하
- PK 외 column이 별로 없는 통계성 테이블에 최적, 폭이 좁고 긴 테이블
- 인덱스 리프 블록 = 데이터 블록
- 일반적인 힙 구조 table에서 데이터 삽입은 Random 방식 <-> IOT는 정렬 상태를 유지하면서 삽입(삽입 시 성능 저하)

Partitioned IOT

Overflow 영역

Secondary 인덱스



**(3) 인덱스 손익분기점**

- Index Rang Scan에 의한 테이블 액세스가 Table Full Scan 보다 느려지는 지점을 인덱스 손익 분기점이라함.
- 인덱스에 의한 액세스가 Full Table Scan 보다 더 느려지게 만드는 가장 핵심적이 두가지 요인
  - 1.인덱스 rowid에 의한 테이블 액세스는 Random 액세스인 반면 Full Table Scan은 Sequential 액세스
  - 2.디스크 I/O시, 인덱스 rowid에 의한 테이블 액세스는 Single Block Read 방식을 사용하는 반면, Full Table Scan은 Multiblock Read 방식을 사용

- 일반적으로 5 ~ 20% 수준에서 결정 되지만 CF에 따라 크게 달라짐.
- 인덱스 CF가 나쁘면 같은 테이블 블록을 여러 번 반복 액세스하면서 논리적I/O 개수가 증가하고 물리적 I/O발생량도 증가하기 때문임.
- 75~77테스트는 CF에 따른 인덱스 스캔과 테이블 풀 스캔의 손익 분기점을 도식화한 그래프임.
- CF가 좋은 경우, CF가 나쁜경우, Full Table Scan 세 가지 상황 기준으로 설명
- 인덱스 손익 분기점 데이터는 상화에 따라 크게 달라지므로 정해진 수치 또는 범위를 정해 놓고 인덱스의 효용성을 판단하면 틀리기 쉬움.
- 테이블 Reorg로의 CF 향상은 최후의 수단으로 사용해야 함.

**손익분기점을 극복하기 위한 기능들**

- 1.IOT(Index-Organized Table) : 테이블을 인덱스 구조로 생성하는 것을 말함.
  - 항상 정렬된 상태 유지함.
  - 테이블 레코드를 읽기 위한 추가적인 Random 액세스 불필요함.

- 2.클러스터 테이블(Clustered Table) : 키 값이 같은 레코드는 같은 블록에 모이도록 저장
  - 클러스터 인덱스를 이용할 때는 테이블 Random 액세스가 키 값별로 한 번씩만 발생
  - 클러스터에 도달해서는 Sequential 방식으로 스캔하기 때문에넓은 범위를 읽더라도 비효율은 없음.

- 3.파티션닝 : 대량 범위 조건으로 자주 사용되는 컬럼 기준으로 테이블을 파티셔닝 한다면, Full Table Scan 하더라도 일부 파티션만 읽고 멈추도록 할 수 있음
- 클러스터는 기준 키 값이 같은 레코드를 블록 단위로 모아 놓지만, 파티셔닝은 세그먼트 단위로 모아 놓는 점이 다름.



## Partition

Divide and Conquer의 정신

데이터베이스 관리자의 관점에서 볼 때, 파티션 된 오브젝트는 일괄적으로 또는 개별적으로 관리가 가능한 여러 개의 조각으로 구성되며, 따라서 오브젝트 관리에 대한 유연성을 크게 개선할 수 있습니다.
애플리케이션의 관점에서 볼 때에는 파티션 된 테이블은 파티션 되지 않은 테이블과 아무런 차이가 없습니다. 그러므로 애플리케이션 변경 작업은 필요하지 않습니다.

테이블은 '파티셔닝 키(partitioning key)'을 통해 분할되며 파티셔닝 키란 특정 로우가 어떤 파티션에 위치하는지 정의하는 일련의 컬럼을 말합니다.

![image-20190823105753268](/Users/lji/Library/Application Support/typora-user-images/image-20190823105753268.png)

![image-20190823105818525](/Users/lji/Library/Application Support/typora-user-images/image-20190823105818525.png)

IOT(index-organized table)에는 Range Partitioning, List Partitioning, Hash Partitioning이 적용될 수
있습니다.

세가지 유형의 파티션 된 인덱스

- local index
- Global partitioned index
- Global Non-partitioned Index

관리성 개선 - 파티션 단위로 작업 실행 -> 유지보수 용이

성능 개선 - 병렬 처리, 조인 실행 속도

## View

- \- 뷰는 하나의 가상 테이블이라 생각 하면 된다.
- \- 뷰는 실제 데이터가 저장 되는 것은 아니지만 뷰를 통해 데이터를 관리 할수 있다.
- \- 뷰는 복잡한 Query를 통해 얻을 수 있는 결과를 간단한 Query로 얻을 수 있게 한다.
- \- 한 개의 뷰로 여러 테이블에 대한 데이터를 검색 할 수 있다.
- \- 특정 평가 기준에 따른 사용자 별로 다른 데이터를 액세스할 수 있도록 한다.

## Other Schema Object

### Sequence

- \- 유일(UNIQUE)한 값을 생성해주는 오라클 객체이다.
- \- 시퀀스를 생성하면 기본키와 같이 순차적으로 증가하는 컬럼을 자동적으로 생성 할 수 있다.
- \- 보통 PRIMARY KEY 값을 생성하기 위해 사용 한다.
- \- 메모리에 Cache되었을 때 시퀀스값의 액세스 효율이 증가 한다.
- \- 시퀀스는 테이블과는 독립적으로 저장되고 생성된다.

### Synonym(동의어)

오라클 객체에 대한 대체이름(alias)

① 데이터베이스의 투명성을 제공하기 위해서 사용 한다고 보면 된다. 시노님은 다른 유저의 객체를 참조할 때 많이 사용을 한다.

② 만약에 실무에서 다른 유저의 객체를 참조할 경우가 있을 때 시노님을 생성해서 사용을 하면은 추후에 참조하고 있는 오프젝트가 이름을 바꾸거나 이동할 경우 객체를 사용하는 SQL문을 모두 다시 고치는 것이 아니라 시노님만 다시 정의하면 되기 때문에 매우 편리 하다.

③ 객체의 긴 이름을 사용하기 편한 짧은 이름으로 해서 SQL코딩을 단순화 시킬 수 있다.

④ 또한 객체를 참조하는 사용자의 오브젝트를 감추 수 있기 때문에 이에 대한 보안을 유지할 수 있다. 시노님을 사용하는 유저는 참조하고 있는 객체를에 대한 소유자, 이름, 서버이름을 모르고 시노님 이름만 알아도 사용 할 수 있다.

## Data Integrity

데이터 무결성 = 데이터 값이 정확한 상태

- 엔티티 무결성
  - 엔티티에 존재하는 모든 인스턴스는 고유, NOT NULL
  - Primary Key = Unique Key + Not NULL
  - Unique Key = 동일한 값을 가지지 못한다
- 참조 무결성
  - 외래 식별자 속성은 엔티티 주 식별자 값과 일치하거나 NULL이여야 한다
  - Foreign Key
    - 관계 속성의 FK는 참조 엔티티의 PK거나 NULL
    - 참조 엔티티의 PK가 수정/삭제될 경우 참조한 모든 값은 수정/삭제되어야 한다
- 도메인 무결성
  - 엔터티의 특정 속성 값은 같은 데이터 타입과 길이, 같은 널 여부, 같은 기본 값, 같은 허용 값 등 동일한 범주의 값만이 존재해야 한다
  - Check : 속성 값에는 특정한 범위의 값이나 특정 규칙을 따르는 값만이 존재
  - Default
  - Data Type : 속성에 데이터 타입을 지정해 특정 형식을 유지
  - NULL / NOT NULL
- 업무 무결성
  - 기업에서 업무를 수행하는 방법이나 데이터를 처리하는 규칙을 의미
  - 업무 무결성은 범위가 넓어 주로 프로그램에서 체크한다.
  - 업무 무결성을 물리적으로 강제하는 대표적인 방법에 트리거(Trigger)가 존재한다.
  - Trigger : 속성 값이 입력/수정/삭제될 경우 자동으로 데이터 처리

## Data Dictionary

데이터사전: 대부분 읽기전용으로 제공되는 테이블 및 뷰들의 집합, 데이터베이스 전반에 대한 정보를 제공

오라클DB는 명령을 실행할 때마다 data dictionary를 접근한다

- \- 오라클의 사용자 정보
- \- 오라클 권한과 롤 정보
- \- 데이터베이스 스키마 객체(TABLE, VIEW, INDEX, CLUSTER, SYNONYM, SEQUENCE..) 정보
- \- 무결성 제약조건에 관한 정보
- \- 데이터베이스의 구조 정보
- \- 오라클 데이터베이스의 함수 와 프로지저 및 트리거에 대한 정보
- \- 기타 일반적인 DATABASE 정보

## Dynamic Performance View

현재 DB 활동을 기록하는 가상 테이블

fixed View( DBA도 변경하거나 제거할 수 없다)

## SQL

- **DDL (Data Definition Language)** : 데이터베이스 객체(테이블,뷰,인덱스..)의 구조를 정의 합니다.

| SQL문  |                            내 용                             |
| :----: | :----------------------------------------------------------: |
| CREATE |               데이터베이스 객체를 생성 합니다.               |
|  DROP  |               데이터베이스 객체를 삭제 합니다.               |
| ALTER  | 기존에 존재하는 데이터베이스 객체를 다시 정의하는역할을 합니다. |



- **DML (Data Manipulation Language)** : 데이터의 검색,삽입,삭제,갱신등을 처리

| SQL문  |                   내 용                   |
| :----: | :---------------------------------------: |
| INSERT |  데이터베이스 객체에 데이터를 입력 한다.  |
| DELETE |  데이터베이스 객체의 데이터를 삭제 한다.  |
| UPDATE |  데이터베이스 객체안의 데이터 수정 한다.  |
| SELECT | 데이터베이스 객체안의 데이터를 검색 한다. |



- **DCL (Data Control Language)** : 데이터베이스 사용자의 권한을 제어

| SQL문  |                     내 용                      |
| :----: | :--------------------------------------------: |
| GRANT  |     데이터베이스 객체에 권한을 부여 한다.      |
| REVOKE | 이미 부여된 데이터베이스 객체 권한을 취소한다. |

### CREATE - 테이블 생성

```sql
CREATE TABLE DEPT2(
    DEPTNO  NUMBER  CONSTRAINT dept_pk_deptno  PRIMARY KEY,
    DNAME   VARCHAR2(40),
    LOC     VARCHAR2(50)) ;
```

### Constraint - 제약조건

#### NOT NULL

```sql
CREATE TABLE emp3(
     ename VARCHAR2(20)  CONSTRAINT emp_nn_ename NOT NULL );
```

#### UNIQUE

```sql
ALTER TABLE emp2
     ADD CONSTRAINT emp2_uk_deptno UNIQUE (deptno);
```

#### CHECK

```sql
ALTER TABLE emp2
     ADD CONSTRAINT emp2_ck_comm
     CHECK (comm >= 1 AND comm <= 100);
```

#### DEFAULT

```sql
CREATE TABLE emp4(
     ... (컬럼생략) ...,
     hiredate DATE DEFAULT SYSDATE );     
```

#### PRIMARY KEY

```sql
CREATE TABLE emp5(
     empno NUMBER CONSTRAINT emp5_pk_empno PRIMARY KEY );     
```

#### FOREIGN KEY

```sql
ALTER TABLE emp2 ADD CONSTRAINT emp2_fk_deptno
     FOREIGN  KEY (deptno) REFERENCES dept(deptno);    
```

### 테이블 컬럼 관리

#### ADD

```sql
ALTER TABLE emp ADD (addr VARCHAR2(50));    
```

#### MODIFY

```sql
ALTER TABLE emp MODIFY (ename VARCHAR2(50));
```

#### DROP

```sql
-- 컬럼의 삭제 문법 
SQL> ALTER TABLE table_name DROP COLUMN column_name

-- 제약 조건의 삭제 예제
SQL> ALTER TABLE emp DROP PRIMARY KEY ;

-- CASCADE 연산자와 함께 사용하면 외래키에 의해 참조되는 기본키도 삭제 할 수 있다.
SQL> ALTER TABLE emp DROP CONSTRAINT emp_pk_empno CASCADE;
```

### 기존 테이블 복사

```sql
-- 제약조건은 NOT NULL만 복사된다
CREATE TABLE emp2
     AS SELECT * FROM emp;
```

### 테이블의 테이블스페이스 변경

```sql
-- 한번 실습해 보세요. (test라는 테이블스페이스가 있어야 겠죠)
SQL> ALTER TABLE emp
     MOVE TABLESPACE test;
```

### 테이블의 TRUNCATE

- \- 테이블을 Truncate하면 테이블의 모든 행이 삭제되고 사용된 공간이 해제 된다.
- \- TRUNCATE TABLE은 DDL명령이므로 롤백 데이터가 생성되지 않는다.
- \- DELETE명령으로 데이터를 지우면 롤백명렁어로 복구 할 수 있지만, TRUNCATE로 데이터를 삭제하면 롤백을 할 수가 없다.
- \- 행당 인덱스도 같이 잘려 나간다.
- \- 외래키가 참조중인 테이블은 TRUNCATE할 수 없다.
- \- TRUNCATE 명령을 사용하면 삭제 트리거가 실행되지 않는다.

### 테이블 삭제(DROP)

```sql
-- emp 테이블 삭제
SQL> DROP TABLE emp;

-- CASCADE CONSTRAINT는 외래키에 의해 참조되는 기본키를 포함한 테이블일 경우 
-- 기본키를 참조하던  외래 키 조건도 같이 삭제 한다.
SQL> DROP TABLE emp CASCADE CONSTRAINT;
```

### INSERT

```sql
-- 모든 데이터를 입력할 경우
SQL> INSERT INTO emp
     VALUES(7369, 'SMITH', 'CLERK', 7902, TO_DATE('80/12/17'),  800, NULL,  20);

-- 원하는 데이터만 입력할 경우
SQL> INSERT INTO dept (deptno, dname)
     VALUES(10, 'ACCOUNTING' );

-- SELECT 문장을 이용한 INSERT
SQL> INSERT INTO dept2
     SELECT * FROM dept;
```

### UPDATE

```sql
UPDATE emp
     SET deptno = 30
     WHERE empno = 7902;
```

### DELETE

```sql
DELETE FROM emp
     WHERE empno = 7902 ;
```

### SELECT

![image-20190823125311611](/Users/lji/Library/Application Support/typora-user-images/image-20190823125311611.png)

![image-20190823125335808](/Users/lji/Library/Application Support/typora-user-images/image-20190823125335808.png)

### JOIN

- 둘 이상의 테이블을 연결하여 데이터를 검색하는 방법

### 조인의 방법

#### - Equal Join

- 성능을 높이려면 인덱스를 사용하는 것이 좋다
- 콤마(,) 대신 **INNER JOIN**을 사용 할 수 있으며, **INNER**는 생략 가능하다. Join 조건은 **ON** 절에 온다.
- **NATURAL JOIN**을 사용 하면 동일한 컬럼을 내부적으로 모두조인 하므로, **ON**절이 생략 가능하다.
- NATURAL JOIN의 단점은 동일한 이름을 가지는 칼럼은 모두 조인이 되는데, **USING** 문을 사용하면 컬럼을 선택해서 조인을 할 수가 있다.

```sql
-- dept 테이블과 emp 테이블을 조인하는 예제
SELECT` `e.empno, e.ename, d.dname
  ``FROM` `dept d, emp e
 ``WHERE` `d.deptno = e.deptno;
-- INNER JOIN절을 이용하여 조인하는 예제
SELECT e.empno, e.ename, d.dname
  FROM dept d 
 INNER JOIN emp e
    ON d.deptno = e.deptno;
-- NATURAL JOIN절을 이용하여 조인하는 예제
SELECT  e.empno, e.ename, d.dname
  FROM  dept d 
NATURAL JOIN emp e;    
-- JOIN~USING절을 이용하여 조인하는 예제
SELECT e.empno, e.ename, deptno 
  FROM emp e 
  JOIN dept d 
 USING (deptno);
```

#### - Non-Equal Join

- \- 테이블의 어떤 column도 Join할 테이블의 column에 일치하지 않을 때 사용하고, 조인조건은 동등( = )이외의 연산자를 갖는다.
- \- BETWEEN AND, IS NULL, IS NOT NULL, IN, NOT IN
- \- 거의 사용하지 않는다

#### - Self Join

- \- Equi Join과 같으나 하나의 테이블에서 조인이 일어나는 것이 다르다.
- \- 같은 테이블에 대해 두 개의 alias를 사용하여 FROM절에 두 개의 테이블을 사용하는 것 처럼 조인한다.

```sql
SELECT` `e.ename, a.ename ``"Manager"
  ``FROM` `emp e, emp a
 ``WHERE` `e.empno = a.mgr;
```

#### - Cross Join(Cartesian Product)

- \- 검색하고자 했던 데이터뿐 아니라 조인에 사용된 테이블들의 모든 데이터가 반환 되는 현상
- \- Cartesian Product는 조인 조건을 정의하지 않은 경우 발생한다.
- \- 테이블의 개수가 N이라면 Cartesian Product를 피하기 위해서는 적어도 N-1개의 등가 조건을 SELECT 문안에 포함시켜야 하며 각 테이블의 컬럼이 적어도 한번은 조건절에 참조되도록 해야 한다.
- \- **CROSS JOIN**을 이용하면 Cartesian Product 값을 얻을 수 있다.

```sql
-- CROSS JOIN절을 이용하여 Cartesian Product 값을 얻는 예제
SELECT`  `e.empno, e.ename, d.dname
  ``FROM`  `dept d ``CROSS` `JOIN` `emp e;
```

#### - Outer Join

- \- Equi Join은 조인을 생성하려는 두 개의 테이블의 한쪽 컬럼에서 값이 없다면 테이터를 반환하지 못한다.
- \- 동일 조건에서 조인 조건을 만족하는 값이 없는 행들을 조회하기 위해 Outer Join을 사용 한다.
- \- Outer Join 연산자는 "(+)" 이다.
- \- 조인시 값이 없는 조인측에 "(+)"를 위치 시킨다.
- Outer Join 연산자는 표현식의 한 편에만 올 수 있다.
- Outer Join을 사용하는 테이블에 추가로 조건절이 있다면 (+)연산자를 모두 해야 한다.
- LEFT OUTERL JOIN은 오른쪽 테이블(아래 예제에서 emp테이블)에 조인시킬 컬럼의 값이 없는 경우 사용한다.
- RIGHT OUTERL JOIN은 왼쪽 테이블(아래 예제에서 emp테이블)에 조인시킬 컬럼의 값이 없는 경우 사용한다.
- FULL OUTERL JOIN은 양쪽 테이블 모두 Outer Join걸어야 하는 경우 사용 한다.

```sql
-- Equi Join 으로 부서 번호를 조회하는 예제
SELECT` `DISTINCT``(e.deptno), d.deptno, d.dname
  ``FROM` `emp e, dept d
 ``WHERE` `e.deptno = d.deptno;
-- Outer Join 으로 부서 번호를 조회하는 예제
SELECT DISTINCT(e.deptno), d.deptno
  FROM emp e, dept d
 WHERE e.deptno(+) = d.deptno;
-- ename LIKE 조건절에 (+)연산자를 추가해야 정상적으로 데이터가 조회 된다. 
SELECT DISTINCT(a.deptno), b.deptno
  FROM emp a, dept b
 WHERE a.deptno(+) = b.deptno
   AND a.ename(+) LIKE '%';
-- LEFT OUTER JOIN 조인 예제
SELECT DISTINCT(e.deptno), d.deptno
  FROM dept d 
  LEFT OUTER JOIN emp e
  ON d.deptno = e.deptno; 
-- RIGHT OUTER JOIN 조인 예제
SELECT DISTINCT(e.deptno), d.deptno
  FROM emp e 
 RIGHT OUTER JOIN dept d
    ON e.deptno = d.deptno;
-- FULL OUTER JOIN 조인 예제
SELECT DISTINCT(e.deptno), d.deptno
  FROM emp e 
  FULL OUTER JOIN dept d
    ON e.deptno = d.deptno;    
```

### 조인의 방식

#### - Nested Loop Join

- Random 액세스 위주의 조인방식
  그러므로, 인덱스 구성이 완벽해도 대량의 데이터 조인시 비효율적
- 조인을 한 레코드씩 순차적으로 진행
  아무리 대용량 집합이더라도 매우 극적인 응답속도를 낼 수 있으며, 먼저 액세스되는 테이블의 처리 범위에 의해 전체 일량이 결정
- 다른 조인방식보다 인덱스 구성 전략이 특히 중요하며, 소량의 데이터를 처리하거나 부분범위 처리가 가능한 OLTP성 환경에 적합한 조인방식

#### - Sort-Merged Join

- NL조인을 효과적으로 수행하려면, 조인 컬럼에 인덱스가 필요하다.
- 만약 적절한 인덱스가 없다면, 옵티마이저는 소트머지조인이나 해시조인을 고려한다.

- 처리절차 : 두 테이블을 각각 정렬한 다음에 두 집합을 머지(Merge)하면서 조인을 수행한다.
  - 소트단계:양쪽 집합을 조인 컬럼 기준으로 정렬한다.
  - 머지단계:정렬된 양쪽 집합을 서로 Merge한다.
- 소트머지 조인은 Outer루프와 Inner루프가 Sort Area에 미리 정렬해둔 데이터를 이용할 뿐, 실제 조인과정은 NL조인과 동일하다.
- 하지만, Sort Area가 PGA영역에 할당되므로 래치획득과정이 없기 때문에, SGA를 경유하는 것보다 훨씬 빠르다.
- PGA영역에 저장된 데이터를 이용하기 때문에 빠르므로 소트부하만 감수하면 NL조인보다 유리하다.
  - (update)NL 조인시 버퍼캐시에서 같은 블럭을 반복 엑세스 할 때 발생하는 비효율을 없애기 위한 Buffer Pinning 기능 점차 확대되는 추세, 이 때문에 버전이 올라갈수록 NL 조인이 조금씩 유리해지고 있다.
- 인덱스유무에 영향을 받지 않는다.
- 스캔위주의 액세스방식을 사용한다.
  (단, 양쪽 소스 집합에서 정렬 대상 레코드를 찾는 작업은 인덱스를 이용해 Random엑세스 방식으로 처리, 이 때 액세스량이 많다면, 소트머지 이점이 사라질수 있음)
- 대부분 해시조인인 보다 느린 성능을 보이나, 아래와 같은 상황에서는 소트머지 조인이 유용하다.
  - First테이블에 소트연산을 대체할 인덱스가 있을 때
  - 조인할 First 집합이 이미 정렬되어 있을 때
  - 조인 조건식이 등치(=)조건이 아닐 때

#### - Hash Join

- 소트머지조인과 NL조인이 효과적이지 못한 상황에 대한 대안으로서 개발되었다.
- 조인대상이 되는 두 집합 중에서 작은 집합(Build Input)을 읽어 Hash Area에 해시 테이블을 생성하고, 큰 집합(Probe Input)을 읽어 해시 테이블을 탐색하면서 조인하는 방식이다.
- 장점: NL조인처럼 조인시 발생하는 Random엑세스 부하가 없다.
  소트머지조인처럼 조인전에 양쪽 집합을 정렬해야 하는 부담이 없다.
- 단점: 해시테이블을 생성하는 비용이 수반
  그러므로, Build Input이 작을 때 효과적이다.(PGA에 할당되는 Hash Area에 담길정도로 충분히 작아야 함)
- 해시키 값으로 사용되는 컬럼에 중복값이 거의 없을 경우 효과적이다.(추후 설명)
- Inner루프로 Hash Area에 생성해둔 해시테이블을 이용한다는 것 외에 NL조인과 유사하다.
- 해시테이블 만들 때는 전체범위처리가 불가피하나, Prob Input을 스캔하는 단계는 부분범위 처리가능하다.
- 해시조인은 해시테이블이 PGA영역에 할당되므로, NL조인보다 빠르다.
  NL조인은 Outer테이블에서 읽히는 레코드마다 Inner쪽 테이블 버퍼캐시 탐색을 위해 래치획득을 반복하나, 해시조인은 래치 획득과정없이 PGA에서 빠르게 데이터를 탐색할 수 있다.

### Aggregate function

- \- GROUP BY절을 이용하여 그룹 당 하나의 결과로 그룹화 할 수 있다.
- \- HAVING절을 사용하여 집계함수를 이용한 조건 비교를 할 수 있다.
- \- MIN, MAX 함수는 모든 자료형에 사용 할 수 있다.
- \- 일반적으로 가장 많이 사용하는 집계함수에는AVG(평균), COUNT(개수), MAX(최대값), MIN(최소값), SUM(합계) 등이 있다.

#### COUNT

```sql
SELECT` `COUNT``(deptno) ``FROM` `dept;
```

#### MAX

```sql
-- sal 컬럼값 중에서 제일 큰값을 반환. 즉 가장 큰 급여를 반환
SELECT` `MAX``(sal) salary ``FROM` `emp;
```

#### MIN

```sql
-- sal 컬럼값 중에서 가장 작은 값 반환. 즉 가장 적은 급여를 반환
SELECT` `MIN``(sal) salary ``FROM` `emp;
```

#### AVG

```sql
-- 부서번호 30의 사원 평균 급여를 소수점 1자리 이하에서 반올림
SELECT` `ROUND(``AVG``(sal),1) salary 
  ``FROM` `emp 
 ``WHERE` `deptno = 30;
```

#### SUM

```sql
-- 부서번호 30의 사원 급여 합계를 조회.
SELECT` `SUM``(sal) salary 
  ``FROM` `emp 
 ``WHERE` `deptno = 30;
```

#### STDDEV

```sql
-- 부서번호 30의 사원 급여 표준편차를 반환.    
SELECT` `ROUND(STDDEV(sal),3) salary 
  ``FROM`  `emp 
 ``WHERE` `deptno = 30;
```

### GROUP BY

- \- GROUP BY 절은 데이터들을 원하는 그룹으로 나눌 수 있다.
- \- 나누고자 하는 그룹의 컬럼명을 SELECT절과 GROUP BY절 뒤에 추가하면 된다.
- \- 집계함수와 함께 사용되는 상수는 GROUP BY 절에 추가하지 않아도 된다. (개발자 분들이 많이 실수 함)
- \- 아래는 집계 함수와 상수가 함께 SELECT 절에 사용되는 예이다.

```sql
-- 부서별 사원수 조회
SELECT` `'2005년'` `year``, deptno 부서번호, ``COUNT``(*) 사원수
  ``FROM` `emp
 ``GROUP` `BY` `deptno
 ``ORDER` `BY` `COUNT``(*) ``DESC``; 
```

### DISTINCT

- \- DISTINCT는 주로 UNIQUE(중복을 제거)한 컬럼이나 레코드를 조회하는 경우 사용한다.
- \- GROUP BY는 데이터를 그룹핑해서 그 결과를 가져오는 경우 사용한다.
- \- 하지만 두 작업은 조금만 생각해보면 동일한 형태의 작업이라는 것을 쉽게 알 수 있으며, 일부 작업의 경우 DISTINCT로 동시에 GROUP BY로도 처리될 수 있는 쿼리들이 있다.
- \- 두 기능 모두 Oracle9i까지는 sort를 이용하여 데이터를 만들었지만, Oracle10g 부터는 모두 Hash를 이용하여 처리한다.
- 집계함수를 사용하여 특정 그룹으로 구분 할 때는GROUP BY 절을 사용하며, 특정 그룹 구분없이 중복된 데이터를 제거할 경우에는 DISTINCT 절을 사용

```sql
-- DISTINCT를 사용한 중복 데이터 제거
SELECT` `DISTINCT` `deptno ``FROM` `emp; 
-- GROUP BY를 사용한 중복 데이터 제거
SELECT` `deptno ``FROM` `emp ``GROUP` `BY` `deptno;
-- 아래와 같은 기능은 DISTINCT를 사용하는 것이 훨씬 효율적이다.
SELECT COUNT(DISTINCT d.deptno) "중복제거 수", 
       COUNT(d.deptno) "전체 수"
  FROM emp e, dept d
 WHERE e.deptno = d.deptno;
-- 집계 함수가 필요한 경우는 GROUP BY를 사용해야 한다.
SELECT deptno, MIN(sal)
  FROM emp 
 GROUP BY deptno;
```

### HAVING

- \- WHERE 절에서는 집계함수를 사용 할 수 없다.
- \- HAVING 절은 집계함수를 가지고 조건비교를 할 때 사용한다.
- \- HAVING절은 GROUP BY절과 함께 사용이 된다.

```sql
SELECT` `b.dname, ``COUNT``(a.empno) ``"사원수"
  ``FROM` `emp a, dept b
 ``WHERE` `a.deptno = b.deptno
 ``GROUP` `BY` `dname
HAVING` `COUNT``(a.empno) > 5;
```







## Server-Side Programming

### PL/SQL

- \- PL/SQL 은 Oracle’s Procedural Language extension to SQL 의 약자 이다.

  \- SQL문장에서 변수정의, 조건처리(IF), 반복처리(LOOP, WHILE, FOR)등을 지원하며,오라클 자체에 내장되어 있는 Procedure Language 이다.

  \- DECLARE문을 이용하여 정의되며, 선언문의 사용은 선택 사항 이다.

  \- PL/SQL 문은 블록 구조로 되어 있고 PL/SQL자신이 컴파일 엔진을 가지고 있다.

### PL/SQL 장점

- \- PL/SQL 문은 BLOCK 구조로 다수의 SQL 문을 한번에 ORACLE DB로 보내서 처리하므로 수행속도를 향상 시킬수 있다.

  \- PL/SQL 의 모든 요소는 하나 또는 두개이상의 블록으로 구성하여 모듈화가 가능하다.

  \- 보다 강력한 프로그램을 작성하기 위해서 큰 블록안에 소블럭을 위치시킬 수 있다.

  \- VARIABLE, CONSTANT, CURSOR, EXCEPTION을 정의하고, SQL문장과 Procedural 문장에서 사용 한다.

  \- 단순, 복잡한 데이터 형태의 변수를 선언 한다.

  \- 테이블의 데이터 구조와 컬럼명에 준하여 동적으로 변수를 선언 할 수 있다.

  \- EXCEPTION 처리 루틴을 이용하여 Oracle Server Error를 처리 한다.

  \- 사용자 정의 에러를 선언하고 EXCEPTION 처리 루틴으로 처리 가능 하다.

### PL/SQL 블록

![image-20190823150438667](/Users/lji/Library/Application Support/typora-user-images/image-20190823150438667.png)

- PL/SQL 블록은 선언부(선택적), 실행부(필수적), 예외 처리부(선택적)로 구성되어 있고, BEGIN과 END 키워드는 반드시 기술해 주어야 한다.

- 유형

  - #### Anonymous Block (익명 블록)

    -  이름이 없는 블록을 의미 하며, 실행하기 위해 프로그램 안에서 선언 되고 실행시에 실행을 위해 PL/SQL 엔진으로 전달 된다. 선행 컴파일러 프로그램과 SQL*Plus 또는 서버 관리자에서 익명의 블록을 내장 할 수 있다.

  - #### Procedure (프로시저)

    - 특정 작업을 수행할수 있는 이름이 있는 PL/SQL 블록으로서, 매개 변수를 받을수 있고, 반복적으로 사용할수 있다.

      보통 연속 실행 또는 구현이 복잡한 트랜잭션을 수행하는 PL/SQL블록을 데이터베이스에 저장하기 위해 생성 한다.

  - #### Function (함수)

    -   보통 값을 계산하고 결과값을 반환하기 위해서 함수를 많이 사용 한다.

        대부분 구성이 프로시저와 유사하지만 IN 파라미터만 사용 할 수 있고, 반드시 반환 될 값의 데이터 타입을 RETURN문에 선언해야 한다.

        또한 PL/SQL블록 내에서 RETURN문을 통해서 반드시 값을 반환 해야 한다.

### Package

-   패키지(package)는 오라클 데이터베이스에 저장되어 있는 서로 관련있는 PL/SQL 프로지져와 함수들의 집합 이다.

    패키지는 선언부와 본문 두 부분으로 나누어 진다.

![img](http://www.gurubee.net/imgs/oracle/plsql/package_sp.jpg)

![img](http://www.gurubee.net/imgs/oracle/plsql/package_bd.jpg)

### Trigger

-   INSERT, UPDATE, DELETE문이 TABLE에 대해 행해질 때 묵시적으로 수행되는 PROCEDURE 이다.

    트리거는 TABLE과는 별도로 DATABASE에 저장 된다.

    트리거는 VIEW에 대해서가 아니라 TABLE에 관해서만 정의 될 수 있다.

    행 트리거 : 컬럼의 각각의 행의 데이터 행 변화가 생길때마다 실행되며, 그 데이터 행의 실제값을 제어할 수 있다.

    문장 트리거 : 트리거 사건에 의해 단 한번 실행되며, 컬럼의 각 데이터 행을 제어 할 수 없다.

![img](http://www.gurubee.net/imgs/oracle/plsql/trigger.jpg)

```plsql
SQL> CREATE OR REPLACE TRIGGER triger_test
       BEFORE
       UPDATE ON dept
       FOR EACH ROW
	   
	   BEGIN
        DBMS_OUTPUT.PUT_LINE('변경 전 컬럼 값 : ' || : old.dname);
        DBMS_OUTPUT.PUT_LINE('변경 후 컬럼 값 : ' || : new.dname);
     END;
     /

-- DBMS_OUTPUT.PUT_LINE을 출력
SQL> SET SERVEROUTPUT ON ; 

-- UPDATE문을 실행시키면.. 
SQL> UPDATE dept SET dname = '총무부' WHERE deptno = 30

-- 트리거가 자동 실행되어 결과가 출력된다. 
변경 전 컬럼 값 : 인사과
변경 후 컬럼 값 : 총무부

1 행이 갱신되었습니다.
```

### Java

