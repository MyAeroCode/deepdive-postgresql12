## Alter, Drop Table

---

### 테이블 수정 구문

-   요소의 이름 변경

```sql
-- 테이블 이름 변경
ALTER TABLE
    [ IF EXISTS ] [ ONLY ] table_name
    RENAME TO new_name

-- 컬럼 이름 변경
ALTER TABLE
    [ IF EXISTS ] table_name
    RENAME [ COLUMN ] old_name TO new_name

-- 제약 이름 변경
ALTER TABLE
    [ IF EXISTS ] [ ONLY ] name [ * ]
    RENAME CONSTRAINT constraint_name TO new_constraint_name
```

<br/>

-   요소의 추가/제거

```sql
ALTER TABLE [ IF EXISTS ] [ ONLY ] name [ * ] _action [, ... ]

-- 가능한 액션
where _action is one of :
    -- 컬럼 추가/삭제
    ADD [ COLUMN ] [ IF NOT EXISTS ] column_name ~~컬럼 정의 구문 중략~~
    DROP [ COLUMN ] [ IF EXISTS ] column_name [ RESTRICT | CASCADE ]

    -- 컬럼 데이터 타입 변경
    ALTER [ COLUMN ] column_name [ SET DATA ] TYPE data_type

    -- 컬럼 기본값 지정/삭제
    ALTER [ COLUMN ] column_name SET DEFAULT expression
    ALTER [ COLUMN ] column_name DROP DEFAULT

    -- 컬럼 NOT NULL 제약 여부
    ALTER [ COLUMN ] column_name { SET | DROP } NOT NULL


    -- 테이블 제약
    ADD table_constraint
    ALTER CONSTRAINT constraint_name [ DEFERRABLE | NOT DEFERRABLE ] [ INITIALLY DEFERRED | INITIALLY IMMEDIATE ]
    DROP CONSTRAINT [ IF EXISTS ]  constraint_name [ RESTRICT | CASCADE ]

    -- 트리거
    ENABLE TRIGGER [ trigger_name | ALL | USER ]
    DISABLE TRIGGER [ trigger_name | ALL | USER ]

    -- 상속
    INHERIT parent_table
    NO INHERIT parent_table

    -- 소유자
    OWNER TO { new_owner | CURRENT_USER | SESSION_USER }

    -- 테이블 스페이스
    SET TABLESPACE new_tablespace

    -- 로그여부
    SET { LOGGED | UNLOGGED }

    -- 테이블 파라미터
    SET ( storage_parameter = value [, ... ] )
    RESET ( storage_parameter [, ... ] )
```

<br/>

-   파티션 테이블 관리

```sql
ALTER TABLE [ IF EXISTS ] name _partition_action

where _partition_action is one of :

    -- 특정 파티션을 분리
    DETACH PARTITION partition_name

    -- 특정 파티션을 장착
    ATTACH PARTITION partition_name { FOR VALUES partition_bound_spec | DEFAULT }
```

**partition_bound_spec :**

```sql
-- for list partition
IN ( partition_bound_expr [, ...] )

-- for range partition
FROM ( { partition_bound_expr | MINVALUE + MAXVALUE } [, ...] )
  TO ( { partition_bound_expr | MINVALUE + MAXVALUE } [, ...] )

-- for hash partition
WITH ( MODULES numberic_literal, REMAINDER numberic_literal )
```

---

### 테이블 삭제 구문

```sql
DROP TABLE [ IF EXISTS ] name [, ...] [ CASCADE | RESTRICT  ]
```

이 명령어는 테이블과 데이터를 모두 삭제시키기 때문에, 단순히 데이터만 삭제하고 싶은 경우에는 `DELETE` 또는 `TRUNCATE`가 적합합니다.
