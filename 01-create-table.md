## Create Table

---

### 테이블 정의 구문

```sql
CREATE [{ TEMPORARY | TEMP } | UNLOGGED ] TABLE [ IF NOT EXISTS ] table_name (
    ~~ 컬럼 정의 구문 중략 ~~
    ~~ 테이블 제약 구문 중략 ~~
)
[ INHERITS( parent_table [, ...] ) ]
[ ON COMMIT { PRESERVE ROWS | DELETE ROWS | DROP }]
[ WITH ( storage_parameter [= value] [, ...]) ]
[ TABLESPACE tablesapce_name ]
```

<br/>

### 상세 해설

-   `{ TEMPORARY | TEMP }`

이 키워드가 함께 쓰인 테이블은 임시 테이블로 정의됩니다. `ASNI Standard`에 의하면 모든 세션에 임시 테이블이 생성되어야 하지만 `PostgreSQL`에서는 해당 세션에서만 임시 테이블을 생성합니다. 세션이 종료되면 해당 테이블은 `DROP`됩니다.

```sql
START TRANSACTION;
CREATE TEMPORARY TABLE test (
    x int
);
COMMIT;

SELECT * FROM test; -- ERROR. NO SUCH TABLE.
```

<br/>

-   `UNLOGGED`

이 키워드가 함께 쓰인 테이블은 `DML LOG`를 저장하지 않습니다. 대량의 데이터 작업에서 성능적으로 유리하지만, 장애가 발생하면 데이터를 복구할 수 없습니다. 데이터베이스를 재시작하면 `TRUNCATE`됩니다.

<br/>

-   `IF NOT EXISTS`

테이블이 이미 있다면 무시합니다. 이 키워드는 에러를 발생시키지 않기 때문에 중요합니다. 예를 들어 아래의 명령어는 `DROP`까지 닿지 않습니다.

```sql
CREATE TABLE T(...);
CREATE TABLE T(...); -- ERROR
DROP TABLE T;
```

하지만 `IF NOT EXISTS`가 함께 쓰였다면, 두 번째 줄은 무시되므로 `DROP`까지 수행됩니다.

```sql
CREATE TABLE IF NOT EXISTS T(...);
CREATE TABLE IF NOT EXISTS T(...); -- IGNORE
DROP TABLE T;
```

<br/>

-   `INHERITS`

`PostgreSQL`에서 지원하는 특별한 구문입니다. 부모의 `Cloumn Definition`과 `Check / NOT NULL` 제약조건을 상속받습니다. `Table Constraint`나 `Index`까지 가져오지 않으므로 주의해야 합니다. 또한 `SELECT`로 부모를 조회하면 자식 데이터도 같이 조회되므로, 이것을 방지하려면 `FROM ONLY`를 사용해야 합니다.

```sql
-- DDL
CREATE TABLE test1 (
    x int
);
CREATE TABLE test2 (
    y int
)
INHERITS (test1);

-- DML
INSERT INTO test1 VALUES (1);
INSERT INTO test2 VALUES (2, 2);

-- SELECT ... FROM
SELECT * FROM test1;
/*
        x
    --------
        1
        2
*/

-- SELECT ... FROM ONLY
SELECT * FROM ONLY test1;
/*
        x
    --------
        2
*/
```

<br/>

-   `ON COMMIT` (임시 테이블에서만 사용할 수 있음.)

트랜잭션에서 `COMMIT`을 만나면 수행할 동작을 정의합니다. `PRESERVE ROWS`는 테이블의 데이터를 보존합니다. `DELETE ROWS`는 데이터의 내용을 삭제합니다. `DROP`은 테이블 자체를 삭제합니다.

```sql
START TRANSACTION;
CREATE TABLE test (
    x int
)
ON COMMIT DROP;
INSERT INTO test VALUES (1);
COMMIT;

SELECT * FROM test; -- ERROR. NO SUCH TABLE.
```

<br/>

-   `WITH`

해당 테이블에만 주어진 테이블 파라미터를 적용합니다. 단, 부모 테이블이 `OIDS = true`라면, `WITH`을 사용해도 강제로 `OIDS = true`로 지정됩니다.

<br/>

-   `TABLESPACE`

테이블 스페이스란 데이터가 저장되는 물리적 공간, 즉 `FILE`을 의미합니다. 이 옵션을 지정하면 명시된 테이블 스페이스에 데이터가 저장됩니다.

```sql
CREATE TABLE TEST (
    x int
)
TABLESPACE TEST_TABLESPACE;
```

테이블 스페이스 생성 구문은 다음과 같습니다.

```sql
CREATE TABLESPACE tablespace_name
[ OWNER { new_owner | CURRENT_USER | SESSION_USER }]
LOCATION 'absolute_directory_path'
[ WITH (tablespace_option = value [, ...])]
```

각 테이블 스페이스에는 생성시에 지정한 소유자만 접근할 수 있습니다. `OWNER` 절을 생략하면 현재 유저가 지정됩니다. 생성된 이후에 사용할 수 있는 명령어의 구문은 다음과 같습니다.

```sql
-- 이름 변경
ALTER TABLESPACE old_name RENAME TO new_name;

-- 소유자 변경
ALTER TABLESPACE tablespace_name OWNER TO new_owner;

-- 삭제
DROP TABLESPACE tablespace_name;
```

<br/>

### 표준과 다른 점

-   `Empty Table`

표준에서는 컬럼이 없는 테이블을 허용하지 않지만 `PostgreSQL`은 이러한 제약을 무시합니다. 컬럼을 삭제할 때 예외가 발생하기 때문입니다. 따라서 아래의 쿼리는 `PostgreSQL`에서 허용됩니다.

```sql
CREATE TABLE empty_table ();
```

---

### 컬럼 정의 구문

```sql
CREATE TABLE table_name (
    column_name type [column_constraint, ...] [, ...]

    ~~ 테이블 제약 구문 중략 ~~
);
```

### 컬럼 제약

-   `NULL` : 이 컬럼은 비어있을 수 있습니다.

```sql
CREATE TABLE TEST (
    x int NOT NULL
);
```

<br/>

-   `NOT NULL` : 이 컬럼은 비어있을 수 없습니다.

```sql
CREATE TABLE TEST (
    x int NOT NULL
);
```

<br/>

-   `UNIQUE` : 이 컬럼의 값은 중복될 수 없습니다.

```sql
CREATE TABLE TEST (
    x int UNIQUE
);
```

<br/>

-   `PRIMARY KEY` : 이 컬럼의 값으로 각 행을 유일하게 식별할 수 있습니다. (`NOT NULL` + `UNIQUE`)

```sql
CREATE TABLE TEST (
    x int PRIMARY KEY
);
```

<br/>

-   `FOREGIEN KEY` : 이 컬럼의 값은 다른 테이블에 있는 컬럼값 중 하나임을 보장합니다.

**기본 구문 :**

`REFERENCES 테이블명(필드명)` 형태로 작성합니다. 참조되는 필드는 `UNIQUE` 제약이 존재해야 합니다.

```sql
CREATE TABLE TEST1 (
    x int UNIQUE
);

CREATE TABLE TEST2 (
    x int REFERENCES TEST1(x)
);
```

<br/>

**일치 제약 :**
왜래키 제약의 뒤에 `MATCH FULL | MATCH SIMPLE`을 함께 적을 수 있습니다. `MATCH FULL`은 외래키에 적힌 필드값이 하나라도 `NULL`임을 허용하지 않습니다. (단, 전부 `NULL`인 경우는 허용되며, 이 경우에는 어떤 행과도 일치하지 않아도 됩니다.) `MATCH SIMPLE`은 외래키에 적힌 필드값이 `NULL`임을 허용합니다. 기본값은 `MATCH SIMPLE`입니다.

<br/>

**데이터 조작 제약 :**

참조하고 있는 테이블의 데이터를 삭제하거나 갱신하려는 경우에 수행할 동작을 정의합니다. 각각 `ON DELETE 동작명`, `ON UPDATE 동작명` 형태로 작성합니다. 동작명은 다음 중 하나입니다.

1. `NO ACTION` (기본값)
   삭제 및 갱신을 수행하면 왜래키 제약에 위배될 경우 에러를 발생합니다.

<br/>

2. `RESTRICT`
   기본적으로는 `NO ACTION`과 동일합니다. `NO ACTION`과 다른 점은 `deferred`를 거부한다는 것 입니다.

<br/>

3. `CASCADE`
   삭제되는 행을 참조하는 모든 행을 삭제하고.
   갱신되는 행을 참조하는 모든 행을 갱신합니다.

<br/>

4. `SET NULL`, `SET DEFAULT`
   삭제되거나 갱신되는 행을 참조하는 컬럼의 값을 `NULL` | `DEFAULT`로 갱신합니다.

참조되는 컬럼이 자주 변경되는 경우, 해당 컬럼에 인덱스를 추가하는 것이 성능에 도움이 될 수 있으며, `NO ACTION`을 제외한 동작은 `deferred` 될 수 없습니다. `deferred`에 대해서는 맨 마지막 장에서 설명합니다.

<br/>

-   `CHECK` : 해당 `Bool Expression` 이 참이어야 합니다.

`CHECK(참거짓표현식)` 형태로 작성합니다.

```sql
CREATE TABLE TEST (
    x int CHECK(x > 0)
);
```

`NO INHERIT`를 함께 사용하면, 자식 테이블에는 해당 제약이 상속되지 않습니다.

```sql
-- DDL
CREATE TABLE TEST1 (
    x int CHECK(x > 0) NO INHERIT
);
CREATE TABLE TEST2 ( ) INHERITS (TEST1);

-- DML
INSERT INTO TEST1 VALUES (-1); -- ERROR
INSERT INTO TEST2 VALUES (-1); -- OK
```

---

### 테이블 제약 구문

```sql
CREATE TABLE table_name (
    ~~ 컬럼 정의 구문 중략 ~~ ,

    CONSTRAINT constraint_name 제약
    [, ...]
);
```

<br/>

### 테이블 제약

-   `CHECK`

컬럼 제약과 동일합니다.

```sql
CREATE TABLE TEST (
    x int,

    CONSTRAINT c CHECK(x > 10) NO INHERIT
);
```

<br/>

-   `PRIMARY KEY`, `UNIQUE`

컬럼 제약과 동일하지만, 여러개의 컬럼을 묶을 수 있습니다.

```sql
CREATE TABLE TEST (
    x int,
    y int,

    CONSTRAINT pk_test PRIMARY KEY(x, y),
    CONSTRAINT uq_test UNIQUE (x, y)
);
```

<br/>

-   `FOREIGN KEY`

컬럼 제약과 동일하지만, `FOREGIN KEY (...)`를 사용하여 참조 컬럼의 조합을 명시해야 합니다. 피참조 컬럼 조합은 `UNIQUE` 제약이 있어야 합니다.

```sql
CREATE TABLE TEST1 (
    x int,
    y int,

    CONSTRAINT pk_test1 PRIMARY KEY(x, y)
);

CREATE TABLE TEST2 (
    x int,
    y int,

    CONSTRAINT fk_test2_x_y FOREIGN KEY (x, y) REFERENCES TEST1(x, y)
);
```

<br/>

-   `EXCLUDE`

`컬럼 WITH 연산자` 형태의 조건이 주어지고 모두 `true`가 된다면 에러를 발생시킵니다.

```sql
CREATE TABLE TEST (
    x int,
    y int,

    CONSTRAINT c EXCLUDE(x WITH =, y WITH =)
);
```

위의 테이블은 두 개의 조건을 평가합니다.

1. 서로다른 두 행의 `x`가 동일(`=`)하다.
2. 서로다른 두 행의 `y`가 동일(`=`)하다.

위의 두 조건이 전부 `true`가 되면 에러가 발생하므로, 이 조건은 `UNIQUE(x, y)`와 같습니다. 성능면에서는 `UNIQUE`보다 불리하지만, 적절하게 사용한다면 엄청난 유연성을 제공합니다. 예를 들어 `CIRCLE WITH &&`을 사용하면 겹쳐지지 않는 원만 저장될 수 있도록 설정할 수 있습니다.

```sql
CREATE TABLE CIRCLES (
    x CIRCLE,

    CONSTRAINT c EXCLUDE(x WITH &&)
);
```

추가로 `WHERE` 절을 사용하면 제약 대상을 설정할 수 있습니다. 아래는 `x < 10` 인 경우에만 `UNIQUE`와 동일한 제약이 적용됩니다.

```sql
CREATE TABLE TEST (
    x int,

    CONSTRAINT c EXCLUDE(x WITH =) WHERE (x < 10)
);
```

---

### 제약이 평가되는 시점을 조절

다음 제약들은 평가되는 시점을 뒤로 늦출 수 있습니다.

-   `UNIQUE`
-   `PRIMARY KEY`
-   `FOREGIN KEY`
-   `EXCLUDE`

다음 제약은 뒤로 늦출 수 없습니다.

-   `NOT NULL`
-   `CHECK`

지연이 허용되는 제약의 뒤에 `DEFERRABLE [INITIALLY DEFERRED | INITIALLY IMMEDIATE]`를 붙여서 제약이 위반되는지 검사하는 시점을 커밋 시점까지 늦출 수 있습니다.

-   `NOT DEFERRABLE` : 명령 직후
-   `DEFERRABLE INITIALLY DEFERRED` : 커밋 직후
-   `DEFERRABLE INITIALLY IMMEDIATE` : 문장 직후 (`DEFERRABLE`의 기본값)

<br/>

`PostgreSQL`에서 `명령`은 `문장`을 구성하는 요소이므로, 다음과 같이 고쳐 말할 수 있습니다.

-   `NOT DEFERRABLE` : 각 행이 변경된 직후에 검사합니다.
-   `DEFERRABLE INITIALLY DEFERRED` : 모든 아이템을 커밋 직후에 검사합니다.
-   `DEFERRABLE INITIALLY IMMEDIATE` : 모든 아이템이 변경된 직후에 검사합니다.

<br/>

다음은 각 시점별로 에러가 발생하는 위치를 나타냅니다.

<br/>

**INITIALLY DEFERRED :**

```sql
START TRANSACTION;

CREATE TABLE test (
    x int,

    CONSTRAINT c PRIMARY KEY(x) DEFERRABLE INITIALLY DEFERRED
);

INSERT INTO test VALUES(1); -- OK
INSERT INTO test VALUES(1); -- OK

COMMIT; -- ERROR
```

두 번째 삽입 이후에 테이블을 조회하면 두 건이 모두 조회됩니다.

```ts
2 rows affected.

no |   x
ㅡㅡㅡㅡㅡㅡ
1  |   1
2  |   1
```

<br/>

**INITIALLY IMMEDIATE :**

```sql
START TRANSACTION;

CREATE TABLE test (
    x int,

    CONSTRAINT c PRIMARY KEY(x) DEFERRABLE INITIALLY IMMEDIATE
);

INSERT INTO test VALUES(1); -- OK
INSERT INTO test VALUES(1); -- ERROR

COMMIT;
```

<br/>

**NOT DEFERRABLE :**

```sql
START TRANSACTION;

CREATE TABLE test (
    x int,

    CONSTRAINT c PRIMARY KEY(x)
);

INSERT INTO test VALUES(1); -- OK
INSERT INTO test VALUES(1); -- ERROR

COMMIT;
```

<br/>

어라? `DEFERRABLE INITIALLY IMMEDIATE`와 `NOT DEFERRABLE`가 동일한 것 같지만, 이것은 예제가 잘못되었습니다. 둘의 미묘한 차이를 끌어내보겠습니다.

```sql
CREATE TABLE example(
    x integer NOT NULL,
    y integer NOT NULL,
    UNIQUE (x, y) DEFERRABLE INITIALLY IMMEDIATE
);

INSERT INTO example (x, y) VALUES (1,1),(2,2),(3,3);

UPDATE example SET x = x + 1, y = y + 1;

SELECT * FROM example;
```

우리가 예상하는 결과는 다음과 같습니다.

| no  | x   | y   |
| :-: | --- | --- |
|  1  | 2   | 2   |
|  2  | 3   | 3   |
|  3  | 4   | 4   |

<br/>

하지만 `DEFERRABLE INITIALLY IMMEDIATE`을 제거하면 다음 에러가 발생합니다.

```ts
ERROR:  duplicate key value violates unique constraint "example_x_y_key"
DETAIL:  Key (x, y)=(2, 2) already exists.
SQL state: 23505
```

<br/>

이 미묘한 차이는 `for`문으로 설명하면 이해하기 쉽습니다.

```cpp
for(int i=0; i<n; i++) {
    data[i].x += 1;
    data[i].y += 1;

    // NOT DEFERRABLE 검사시점
}
// DEFERRABLE INITIALLY IMMEDIATE 검사시점
```

<br/>

`NOT DEFERRABLE`는 각 아이템이 수정될 때 마다 검사하기 때문에, 다른 아이템이 수정되지 않은 상태에서 검사가 발생되지만 `DEFERRABLE INITIALLY IMMEDIATE`은 문장이 끝날때까지 검사를 연기하므로, 모든 아이템이 수정된 상태에서 검사가 진행됩니다.
