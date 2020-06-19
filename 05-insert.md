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

먼저 `DO NOTHING`은 충돌을 유발시킨 신규 행을 무시하고 넘어갑니다. 먼저 쿼리를 생각해보겠습니다.

```sql
CREATE TABLE Point3D (
    x int,
    y int,
    z int,
    CONSTRAINT point3d_pk PRIMARY KEY(x, y, z)
);

INSERT INTO Point3D VALUES
    (1, 1, 1),
    (1, 1, 1)
ON CONFLICT (x, y, z) DO NOTHING;
```

위의 쿼리는 `(1, 1, 1)`이 중복으로 삽입되어 충돌이 발생하므로, 아무것도 삽입되지 말아야 할 것처럼 보이지만, `DEFERRABLE`옵션의 기본값인 `NOT DEFERRABLE`는 각 행이 삽입될 때 마다 위반을 검사하므로, 첫번째 행이 삽입된 시점에서는 위반제약이 발생하지 않습니다.

<br/>

즉, 첫번째 `(1, 1, 1)`은 남아있게됩니다.

| x   | y   | z   |
| --- | --- | --- |
| 1   | 1   | 1   |

<br/>

반면에 `DO UPDATE SET`은 충돌을 발생시킨 `신규 행`을 사용하여 `기존 행`을 갱신합니다. 아래의 쿼리를 생각해보겠습니다.

```sql
CREATE TABLE Point3D (
    x int,
    y int,
    z int,
    CONSTRAINT point3d_pk PRIMARY KEY(x, y, z)
);

-- 첫 번째 삽입
INSERT INTO Point3D VALUES
    (1, 1, 1),
    (2, 2, 2)

-- 두 번째 삽입
INSERT INTO Point3D VALUES
    (1, 1, 1),
    (2, 2, 2,),
    (3, 3, 3)
ON CONFLICT (x, y, z) DO UPDATE SET (x, y, z) = (excluded.x, excluded.y, excluded.z);
```

두 번째 삽입에서 `1,1,1`, `2,2,2`가 중복 삽입되어 충돌이 발생했지만, `DO UPDATE SET` 구문으로 인해 `삽입`이 아닌 `갱신`이 발생합니다. 이 때, 충돌을 발생시킨 행은 `excluded` 테이블로 전달되는데,

-   `1,1,1`의 충돌을 발생시킨 행은 두 번째 삽입의 `1,1,1`이고,
-   `2,2,2`의 충돌을 발생시킨 행은 두 번째 삽입의 `2,2,2`이므로,

`1,1,1`과 `2,2,2`를 삽입할 때의 `excluded`는 서로 다름을 알 수 있습니다. `3,3,3`은 충돌하지 않았으므로 정상적으로 삽입됩니다.

| x   | y   | z   |
| --- | --- | --- |
| 1   | 1   | 1   |
| 2   | 2   | 2   |
| 3   | 3   | 3   |

<br/>

그러나 아래의 쿼리는 다음과 같은 에러를 발생시키면서, 삽입이 실패합니다.

```sql
CREATE TABLE Point3D (
    x int,
    y int,
    z int,
    CONSTRAINT point3d_pk PRIMARY KEY(x, y, z)
);

-- 첫 번째 삽입
INSERT INTO Point3D VALUES
    (1, 1, 1);

-- 두 번째 삽입
INSERT INTO Point3D VALUES
    (1, 1, 1),
    (1, 1, 1)
ON CONFLICT (x, y, z) DO UPDATE SET (x, y, z) = (excluded.x, excluded.y, excluded.z);
```

```
Schema Error: error: ON CONFLICT DO UPDATE command cannot affect row a second time
```

`1,1,1`에 대해 충돌을 발생시킨 행이 여러개이기 때문에, 무엇을 사용해여 갱신해야 하는지 모호하기 때문입니다. 이 경우에는 `excluded`가 다음과 같은 상황입니다.

| x   | y   | z   |
| --- | --- | --- |
| 1   | 1   | 1   |
| 1   | 1   | 1   |

즉, `excluded`는 단일 행 테이블 제약을 가지고 있는것으로 볼 수 있습니다.

<br/>

#### 표현식 기반 충돌검사

유니크 인덱스에서 사용된 `표현식`도 가능합니다. 아래의 예제는 `각 요소의 합`이 중복된 다른 행이 있을 때, 유니크 제약 위반이 발생합니다.

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

<br/>

#### `OVERRIDING` Clause

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
