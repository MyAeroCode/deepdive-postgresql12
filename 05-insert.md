## Insert

### 기본 구문

```sql
[ WITH [ RECURSIVE ] with_query [, ...] ]
INSERT INTO table_name [ AS alias ] [ ( column_name [, ...] ) ]
    [ OVERRIDING { SYSTEM | USER } VALUE ]
    { DEFAULT VALUES | VALUES ( { expression | DEFAULT } [, ...] ) [, ...] | query }
    [ ON CONFLICT [ conflict_target ] conflict_action ]
    [ RETURNING * | output_expression [ [ AS ] output_name ] [, ...] ]
```

<br/>

### 기본 설명

테이블에 `VALUES` 또는 `SELECT-RESULT`를 사용하여 0개 이상의 행을 삽입합니다. 타겟 컬럼의 순서는 임의의 순서로 나열할 수 있으며, 생략된 경우 `CREATE TABLE`에서 정의된 순서대로 정의됩니다.

```sql
CREATE TABLE Point2D (
    x int,
    y int
);

-- 암묵적 컬럼 순서
INSERT INTO Point2D VALUES
    (0, 0), -- (0, 0)
    (1, 2), -- (1, 2)
    (2, 4); -- (2, 4)

-- 명시적 컬럼 순서
INSERT INTO Point2D(y, x) VALUES
    (6, 3), -- (3, 6)
    (8, 4); -- (4, 8)
```

<br/>

생략된 컬럼은 기본값이 정의되어 있다면 `DEFAULT`, 그렇지 않다면 `null`로 대체됩니다. 또는 명시적으로 `DEFAULT` 또는 `null`를 선언할 수 있습니다.

```sql
CREATE TABLE Point3D (
    x int,
    y int,
    z int default 0
);

-- 암묵적 NULL
INSERT INTO Point3D(x, z) VALUES
    (1, 1), -- (1, null, 1)
    (2, 2); -- (2, null, 2)

-- 암묵적 DEFAULT
INSERT INTO Point3D(y, x) VALUES
    (6, 3), -- (3, 6, 0)
    (8, 4); -- (4, 8, 0)

-- 명시적 DEFAULT, NULL
INSERT INTO Point3D VALUES
    (5, 10, default), -- (5, 10, 0)
    (6, 12, null   ); -- (6, 12, null)
```

<br/>

모든 컬럼을 `DEFAULT` 또는 `null`으로 채우고 싶다면 `DEFAULT VALUES`를 사용하면 됩니다.

```sql
CREATE TABLE Point3D (
    x int default 0,
    y int default 1,
    z int
);

INSERT INTO Point3D default values; -- (0, 1, null)
```

<br/>

타입이 올바르지 않은 경우 형변환을 시도합니다. 이 경우 `부동소수점` 타입의 오차를 주의해야 합니다. `2.01`은 `2`로 캐스팅되지만, `5.9999`는 `6`으로 캐스팅됩니다.

```sql
CREATE TABLE Point3D (
    x int,
    y int,
    z int
);

INSERT INTO Point3D VALUES
    (0.5, 2.01, '3'), -- (1, 2, 3)
    (4, 5.9999, '6'); -- (4, 6, 6)
```

<br/>

### `WITH` Clause

`SELECT`의 `WITH`절과 동일합니다.

<br/>

### `ON CONFLICT` Clause

#### 기본 구문

where `conflict_target` can be one of:

```sql
    ( { index_column_name | ( index_expression ) } [ COLLATE collation ] [ opclass ] [, ...] ) [ WHERE index_predicate ]
    ON CONSTRAINT constraint_name
```

and `conflict_action` is one of:

```sql
    DO NOTHING
    DO UPDATE SET { column_name = { expression | DEFAULT } |
                    ( column_name [, ...] ) = [ ROW ] ( { expression | DEFAULT } [, ...] ) |
                    ( column_name [, ...] ) = ( sub-SELECT )
                  } [, ...]
              [ WHERE condition ]
```

<br/>

#### 제약 위반 행동

`ON CONFLICT`절은 `DEFERRABLE`이 아닌 유니크 제약이 설정된 `단일컬럼` 또는 `다중컬럼`에 유니크 제약 위반이 발생했을 때 취할 행동을 정의합니다. `DEFERRABLE`에 대한 자세한 설명은 `Chapter 01`을 참조해주세요.

<br/>

유니크 제약 위반이 발생했을 때, 다음 행동을 취할 수 있습니다.

-   `DO NOTHING` : `신규 행`을 삽입하지 않습니다.
-   `DO UPDATE SET` : `기존 행`을 갱신합니다. 충돌을 발생시킨 `기존 행`은 `excluded` 테이블로 전달됩니다.

<br/>

#### `DO NOTHING`

먼저 `DO NOTHING`은 충돌을 유발시킨 신규 행을 무시하고 넘어갑니다. 먼저 쿼리를 생각해보겠습니다.

```sql
CREATE TABLE Point3D (
    x int,
    y int,
    z int,
    CONSTRAINT point3d_pk PRIMARY KEY(x, y)
);

INSERT INTO Point3D VALUES
    (0, 0, 0),
    (0, 0, 1),
    (0, 0, 2)
ON CONFLICT (x, y) DO NOTHING;
```

`Point3D`는 xy평면에서 중복된 점을 허용하지 않기 때문에, `(x=0, y=0)`이 연속으로 삽입되어 충돌이 발생하므로, 위의 쿼리는 아무것도 삽입되지 말아야 할 것처럼 보이지만, `DEFERRABLE`옵션의 기본값인 `NOT DEFERRABLE`는 각 행이 삽입될 때 마다 위반을 검사하므로, 첫번째 행이 삽입된 시점에서는 위반제약이 발생하지 않습니다.

<br/>

즉, 맨 처음에 삽입된 `(0, 0, 0)`은 남아있게됩니다.

| x   | y   | z   |
| --- | --- | --- |
| 0   | 0   | 0   |

<br/>

#### `DO UPDATE SET`

반면에 `DO UPDATE SET`은 충돌을 발생시킨 `신규 행`을 사용하여 `기존 행`을 갱신합니다. 아래의 쿼리를 생각해보겠습니다.

```sql
CREATE TABLE Point3D (
    x int,
    y int,
    z int,
    CONSTRAINT point3d_pk PRIMARY KEY(x, y)
);

-- 첫 번째 삽입
INSERT INTO Point3D VALUES
    (0, 0, 0),
    (1, 1, 1);

-- 두 번째 삽입
INSERT INTO Point3D VALUES
    (0, 0, 5),
    (1, 1, 5),
    (2, 2, 5)
ON CONFLICT (x, y) DO UPDATE SET (x, y, z) = (excluded.x, excluded.y, excluded.z);
```

두 번째 삽입에서 `(x=0, y=0)`, `(x=1, y=1)`이 중복 삽입되어 충돌이 발생했지만, `DO UPDATE SET` 구문으로 인해 `삽입`이 아닌 `갱신`이 이루어집니다. 이 때, 충돌을 발생시킨 행은 `excluded` 테이블로 전달되는데,

-   `(x=0, y=0)`의 충돌을 발생시킨 행은 `0,0,5`이고,
-   `(x=1, y=1)`의 충돌을 발생시킨 행은 `1,1,5`이므로,

`(x=0, y=0)`와 `(x=1, y=1)`를 삽입할 때의 `excluded`는 서로 다름을 알 수 있습니다. 다만, `(x=2, y=2)`은 충돌하지 않았으므로 정상적으로 삽입됩니다.

**첫 번째 삽입:**
| x | y | z |
| --- | --- | --- |
| 0 | 0 | 0 |
| 1 | 1 | 1 |

**두 번째 삽입:**
| x | y | z |
| --- | --- | --- |
| 0 | 0 | 5 |
| 1 | 1 | 5 |
| 2 | 2 | 5 |

<br/>

그러나 아래의 쿼리는 다음과 같은 에러를 발생시키면서, 삽입이 실패합니다.

```sql
CREATE TABLE Point3D (
    x int,
    y int,
    z int,
    CONSTRAINT point3d_pk PRIMARY KEY(x, y)
);

-- 첫 번째 삽입
INSERT INTO Point3D VALUES
    (0, 0, 0);

-- 두 번째 삽입
INSERT INTO Point3D VALUES
    (0, 0, 1),
    (0, 0, 2)
ON CONFLICT (x, y) DO UPDATE SET (x, y, z) = (excluded.x, excluded.y, excluded.z);
```

```
Schema Error: error: ON CONFLICT DO UPDATE command cannot affect row a second time
```

`(x=0, y=0)`에 대해 충돌을 발생시킨 행이 여러개이기 때문에, 무엇을 사용해여 갱신해야 하는지 모호하기 때문입니다. 이 경우에는 `excluded`가 다음과 같은 상황입니다.

**excluded of (x=0, y=0):**

| x   | y   | z   |
| --- | --- | --- |
| 1   | 1   | 1   |
| 1   | 1   | 1   |

즉, `excluded`는 단일 행 제약을 가지고 있는것으로 볼 수 있습니다.

<br/>

`DO UPDATE SET ... WHERE`를 사용하면 특정 조건을 만족하는 행에 대해서만 `DO UPDATE SET`으로 작동하도록 지정할 수 있습니다. 조건을 만족하지 않는 행은 `DO NOTHING`으로 작동됩니다.

```sql
CREATE TABLE Point3D (
    x int,
    y int,
    z int,
    CONSTRAINT point3d_pk PRIMARY KEY(x, y)
);

-- 첫 번째 삽입
INSERT INTO Point3D VALUES
    (0, 0, 0),
    (1, 1, 0),
    (2, 2, );

-- 두 번째 삽입
INSERT INTO Point3D VALUES
    (0, 0, 5),
    (1, 1, 5),
    (2, 2, 5)
ON CONFLICT (x, y) DO UPDATE SET (x, y, z) = (excluded.x, excluded.y, excluded.z) WHERE point3d.x=1;
```

위의 쿼리는 `(x=1)`인 행에 대해서만 `DO UPDATE SET`으로 작동하고, 그 외에는 `DO NOTHING`으로 작동하므로, 최종 결과는 다음과 같습니다.

**두 번째 삽입 :**

| x   | y   | z   |
| --- | --- | --- |
| 0   | 0   | 0   |
| 1   | 1   | 5   |
| 2   | 2   | 0   |

<br/>

#### 표현식 기반 충돌검사

유니크 인덱스에서 사용된 `표현식`도 사용할 수 있습니다. 아래의 예제는 `각 요소의 합`이 중복된 다른 행이 있을 때, 유니크 제약 위반이 발생합니다.

```sql
CREATE TABLE Point3D (
    x int,
    y int,
    z int
);
CREATE UNIQUE INDEX point3d_uq ON Point3D((x+y+z));

INSERT INTO Point3D VALUES
    (3, 2, 1),  -- 각 요소의 합이 6
    (0, 0, 6)   -- 각 요소의 합이 6
ON CONFLICT ((x+y+z)) DO NOTHING;
```

**삽입 결과 :**

| x   | y   | z   |
| --- | --- | --- |
| 3   | 2   | 1   |

<br/>

### `OVERRIDING` Clause

`OVERRIDING`절은 `GENERATED ALWAYS`, `SEQUENCE`, `SERIAL`에 관련된 제약을 지정합니다.

<br/>

-   `OVERRIDING SYSTEM VALUE`

기본적으로 `GENERATED ALWAYS`로 지정된 컬럼이 `DEFAULT`가 아닌 값이 주어지면 에러가 발생합니다. 이 구문은 해당 제약을 무시합니다.

<br/>

-   `OVERRIDING USER VALUE`

`GENERATED ALWAYS`로 지정된 컬럼이 명시적으로 지정되더라도, 그것을 무시하고 다음 값을 가져와 덮어씁니다. 이것은 테이블을 복사할 때 매우 유용합니다. 다음과 같은 쿼리를 생각해봅시다.

```sql
INSERT INTO table2
OVERRIDING USER VALUE
SELECT * FROM table1
```

위의 쿼리는 `table1`에서 사용된 `SERIAL` 컬럼의 값을 무시하고 `table2`와 연관된 시퀀스에서 새로운 값을 추출하여 덮어씁니다.

<br/>

### `RETURNING` Clause

#### 기본 구문

```sql
   [ RETURNING * | output_expression [ [ AS ] output_name ] [, ...] ]
```

`RETURNING` 절은 성공적으로 `삽입` 또는 `갱신 (DO UPDATE SET)`이 이루어진 행의 결과 중, `전체 컬럼` 또는 `일부 컬럼` 또는 `표현식 결과`를 반환합니다.

<br/>

단, `SELECT-FROM`에는 `INSERT`, `UPDATE`, `DELETE` 문장을 사용할 수 없으므로 `SELECT-WITH`에 해당 내용을 적어야 합니다.

<br/>

#### 예제

아래의 쿼리에서 `RETURNING`절을 수정하면서 결과를 살펴보겠습니다.

```sql
CREATE TABLE Point3D (
    x int,
    y int,
    z int,
    CONSTRAINT point3d_pk PRIMARY KEY(x, y)
);

-- 첫 번째 삽입
INSERT INTO Point3D VALUES
    (0, 0, 0),
    (1, 1, 0),
    (2, 2, 0);

-- 두 번째 삽입
WITH RES AS (
    INSERT INTO Point3D VALUES
        (0, 0, 5),
        (1, 1, 5),
        (2, 2, 5)
    ON CONFLICT (x, y) DO UPDATE SET (x, y, z) = (excluded.x, excluded.y, excluded.z) WHERE Point3D.x % 2 = 0
    --        v
    RETURNING *
)
SELECT * FROM RES;
```

<br/>

`RETURNING *`은 변경된 행의 전체 컬럼을 반환합니다.

```sql
RETURNING *
```

| x   | y   | z   |
| --- | --- | --- |
| 0   | 0   | 5   |
| 2   | 2   | 5   |

<br/>

`RETURNING {column | expr} (AS alias), [...]`은 변경된 행의 일부 컬럼 또는 간단한 표현식의 결과를 반환합니다. 어떤 컬럼에는 별명을 부여할 수 있습니다.

```sql
RETURNING x, z
```

| x   | z   |
| --- | --- |
| 0   | 5   |
| 2   | 5   |

```sql
RETURNING x, z AS k
```

| x   | k   |
| --- | --- |
| 0   | 5   |
| 2   | 5   |

```sql
RETURNING x, z AS k, 1+1 AS t
```

| x   | k   | t   |
| --- | --- | --- |
| 0   | 5   | 2   |
| 2   | 5   | 2   |

<br/>

#### 공집합 반환

삽입되거나 갱신된 행이 없다면 `RETURNING` 절은 공집합을 반환합니다. 이것을 사용하여 삽입되거나 갱신된 행의 개수와 유무를 알 수 있습니다.

```sql
CREATE TABLE Point3D (
    x int,
    y int,
    z int,
    CONSTRAINT point3d_pk PRIMARY KEY(x, y)
);

-- 첫 번째 삽입
INSERT INTO Point3D VALUES
    (0, 0, 0);

-- 두 번째 삽입
WITH RES AS (
    INSERT INTO Point3D VALUES
        (0, 0, 1),
        (0, 0, 2),
        (0, 0, 3),
        (0, 0, 4),
        (0, 0, 5)
    ON CONFLICT DO NOTHING
    RETURNING 1
)
...
```

<br/>

```sql
SELECT COUNT(*) AS affected FROM RES;
```

| affected |
| :------: |
|    0     |

<br/>

```sql
SELECT EXISTS (SELECT * FROM RES) AS affected;
```

| affected |
| :------: |
|  false   |

<br/>

`COUNT(*)`는 집계함수이므로 간단한 표현식에 포함되지 않기 때문에, 아래처럼 변경된 행의 개수를 가져올 수 없습니다.

```sql
-- 두 번째 삽입
WITH RES AS (
    INSERT INTO Point3D VALUES
        (0, 0, 1),
        (0, 0, 2),
        (0, 0, 3),
        (0, 0, 4),
        (0, 0, 5)
    ON CONFLICT DO NOTHING
    RETURNING count(*)
)
SELECT * FROM RES;
```

<br/>

#### `WITH` 주의점

`SELECT`문장의 동작순서에서 `WITH`절은 최우선으로 처리됩니다. 하지만 `WITH` 절에 부작용이 존재하는 쿼리를 작성했더라도, 해당 부작용은 검출되지 않습니다. 다음 쿼리를 생각해보겠습니다.

```sql
CREATE TABLE Point3D (
    x int,
    y int,
    z int,
    CONSTRAINT point3d_pk PRIMARY KEY(x, y)
);

-- 첫 번째 삽입
INSERT INTO Point3D VALUES
    (0, 0, 0);

-- 두 번째 삽입
WITH RES AS (
    INSERT INTO Point3D VALUES
        (1, 1, 1),
        (2, 2, 2)
    ON CONFLICT DO NOTHING
    RETURNING *
)
SELECT * FROM Point3D;
```

<br/>

`SELECT`는 `WITH`를 최우선으로 처리하므로 `(1, 1, 1)`, `(2, 2, 2)`가 `Point3D`테이블에 정상적으로 삽입되었지만, 메인 쿼리는 부작용 이전의 결과를 반환합니다.

**Point3D :**

| x   | y   | z   |
| --- | --- | --- |
| 0   | 0   | 0   |

<br/>

`RES`가 사용되지 않아 `INSERT`자체가 일어나지 않았다고 착각할 수 있겠지만, 다시 `Point3D`를 쿼리하면 `INSERT`는 성공적으로 동작했음을 알 수 있습니다.

**Point3D :**

| x   | y   | z   |
| --- | --- | --- |
| 0   | 0   | 0   |
| 1   | 1   | 1   |
| 2   | 2   | 2   |

<br/>

메인 쿼리에서 부작용을 포함한 결과를 가져와야 한다면, `부작용 이전의 결과`에 `RETURNING으로 가져온 부작용`을 더해야 합니다.

```sql
WITH RES AS (
    INSERT INTO Point3D VALUES
        (1, 1, 1),
        (2, 2, 2)
    ON CONFLICT DO NOTHING
    RETURNING *
)
SELECT * FROM Point3D -- 부작용 이전의 결과
UNION ALL
SELECT * FROM RES; -- 부작용
```

**Point3D + RES :**

| x   | y   | z   |
| --- | --- | --- |
| 0   | 0   | 0   |
| 1   | 1   | 1   |
| 2   | 2   | 2   |
