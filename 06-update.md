## Update

### 기본 구문

```sql
[ WITH [ RECURSIVE ] with_query [, ...] ]
UPDATE [ ONLY ] table_name [ * ] [ [ AS ] alias ]
    SET { column_name = { expression | DEFAULT } |
          ( column_name [, ...] ) = [ ROW ] ( { expression | DEFAULT } [, ...] ) |
          ( column_name [, ...] ) = ( sub-SELECT )
        } [, ...]
    [ FROM from_item [, ...] ]
    [ WHERE condition | WHERE CURRENT OF cursor_name ]
    [ RETURNING * | output_expression [ [ AS ] output_name ] [, ...] ]
```

<br/>

### 기본 설명

`명시적으로 주어진 조건`을 충족하는 모든 행에 대해, `명시적으로 주어진 값`을 사용하여 `명시적으로 주어진 컬럼`을 수정합니다.

-   조건이 주어지지 않는다면 모든 행을 수정합니다.
-   값이 주어지지 않는다면 에러가 발생합니다.
-   명시적으로 주어지지 않은 컬럼은 이전의 값을 유지합니다.

<br/>

### `WITH` Clause

`SELECT`의 `WITH`절과 동일합니다.

<br/>

### Sample Table

`Update`문을 실습하기 위한 예제 테이블입니다.

```sql
CREATE TABLE Point3D (
    x int default 0,
    y int,
    z int
);

INSERT INTO Point3D VALUES
    (1, 2, 3),
    (4, 5, 6),
    (7, 8, 9);
```

| x   | y   | z   |
| --- | --- | --- |
| 1   | 2   | 3   |
| 4   | 5   | 6   |
| 7   | 8   | 9   |

<br/>

### `SET column_name = expr`

표현식의 결과를 사용할 수 있습니다.

```sql
UPDATE Point3D SET x = 1 - 1;
```

| x   | y   | z   |
| --- | --- | --- |
| 0   | 2   | 3   |
| 0   | 5   | 6   |
| 0   | 8   | 9   |

<br/>

등호의 우측에서 이전의 값을 사용할 수 있습니다.

```sql
UPDATE Point3D SET x = x + 1;
```

| x   | y   | z   |
| --- | --- | --- |
| 2   | 2   | 3   |
| 5   | 5   | 6   |
| 8   | 8   | 9   |

<br/>

튜플로 묶거나 콤마로 구분하여 다중 컬럼을 수정을 할 수 있습니다.

```sql
-- 튜플 스타일
UPDATE Point3D SET (x, z) = (z, x);

-- 콤마 스타일
UPDATE Point3D SET x=z, z=x;
```

| x   | y   | z   |
| --- | --- | --- |
| 3   | 2   | 1   |
| 6   | 5   | 4   |
| 9   | 8   | 7   |

<br/>

위의 두 스타일을 혼용하여 사용할 수도 있습니다.

```sql
UPDATE Point3D SET y=0, (z, x)=(x, z);
```

| x   | y   | z   |
| --- | --- | --- |
| 3   | 0   | 1   |
| 6   | 0   | 4   |
| 9   | 0   | 7   |

<br/>

### `SET column_name = DEFAULT`

`DEFAULT`를 사용하여 해당 컬럼의 기본값으로 갱신할 수 있습니다. 기본값이 명시되어있지 않다면 `null`로 대체됩니다.

```sql
UPDATE Point3D SET (x, y, z) = (DEFAULT, DEFAULT, DEFAULT);
```

| x   | y    | z    |
| --- | ---- | ---- |
| 0   | null | null |
| 0   | null | null |
| 0   | null | null |

<br/>

### `SET column_name = (SUB-SELECT)`

단일 행을 반환하는 서브쿼리를 사용하여 갱신할 수 있습니다. `PostgreSQL`에서 `VALUES`문은 `SELECT`문과 호환되므로 이것을 사용할수도 있습니다.

```sql
-- SELECT 문
UPDATE Point3D SET (x, y, z) = (SELECT z, 0, x);

-- VALUES 문
UPDATE Point3D SET (x, y, z) = (VALUES (z, 0, x));
```

| x   | y   | z   |
| --- | --- | --- |
| 3   | 0   | 1   |
| 6   | 0   | 4   |
| 9   | 0   | 7   |

<br/>

### `WHERE` Clause

수정될 행의 조건을 명시적으로 지정할 수 있습니다.

```sql
UPDATE Point3D SET (x, y, z) = (0, 0, 0) WHERE x = 1;
```

| x   | y   | z   |
| --- | --- | --- |
| 0   | 0   | 0   |
| 4   | 5   | 6   |
| 7   | 8   | 9   |

<br/>

### `FROM` Clause

필요한 경우, 등호의 우측에서 다른 테이블의 컬럼을 사용할 수 있습니다. 먼저 외부 테이블을 생성하겠습니다.

```sql
CREATE TABLE Point2D (
    x int,
    y int
);
```

<br/>

외부 테이블이 단일 행인 경우에는 이야기가 쉬워집니다.

```sql
INSERT INTO Point2D VALUES (0, 0);

UPDATE Point3D SET (x, y) = (Point2D.x, Point2D.y) FROM Point2D;
```

| x   | y   | z   |
| --- | --- | --- |
| 0   | 0   | 3   |
| 0   | 0   | 6   |
| 0   | 0   | 9   |

<br/>

외부 테이블이 다중 행인 경우에는 `WHERE`절을 사용하여 범위를 줄입니다.

```sql
INSERT INTO Point2D VALUES
    (1, -2),
    (4, -5),
    (7, -8);

UPDATE Point3D SET (x, y) = (Point2D.x, Point2D.y) FROM Point2D WHERE Point3D.x = Point3D.x;
```

| x   | y   | z   |
| --- | --- | --- |
| 1   | -2  | 3   |
| 4   | -5  | 6   |
| 7   | -8  | 9   |

<br/>

단, 일치하는 것이 없는 행은 무시됩니다.

```sql
INSERT INTO Point2D VALUES
    (4, -5),
    (7, -8);

UPDATE Point3D SET (x, y) = (Point2D.x, Point2D.y) FROM Point2D WHERE Point3D.x = Point3D.x;
```

| x   | y   | z   |
| --- | --- | --- |
| 1   | 2   | 3   |
| 4   | -5  | 6   |
| 7   | -8  | 9   |

<br/>

### `ONLY`

상위 테이블의 변경은 하위 테이블에도 영향을 끼칩니다.

```sql
CREATE TABLE Point2D (
    x int,
    y int
);

CREATE TABLE Point3D (
    z int
)
INHERITS (Point2D);

INSERT INTO Point2D VALUES
    (0, 0),
    (1, 1),
    (2, 2);

INSERT INTO Point3D VALUES
    (1, 2, 3),
    (4, 5, 6),
    (7, 8, 9);
```

<br/>

먼저, 각각의 테이블을 조회해보겠습니다.

```sql
SELECT * FROM Point2D;
```

| x   | y   |
| --- | --- |
| 0   | 0   |
| 1   | 1   |
| 2   | 2   |
| 1   | 2   |
| 4   | 5   |
| 7   | 8   |

```sql
SELECT * FROM Point3D;
```

| x   | y   | z   |
| --- | --- | --- |
| 1   | 2   | 3   |
| 4   | 5   | 6   |
| 7   | 8   | 9   |

<br/>

`Point2D`를 조회하면 자식 테이블인 `Point3D`의 `(x,y)`도 같이 조회되는 것을 확인할 수 있습니다. 이것을 방지하려면 `부모 테이블의 앞에 ONLY`를 적으면 됩니다.

```sql
SELECT * FROM ONLY Point2D;
```

| x   | y   |
| --- | --- |
| 0   | 0   |
| 1   | 1   |
| 2   | 2   |

<br/>

위와 비슷하게 `ONLY`를 사용하지 않고 `Point2D`를 수정하면 `Point3D`에도 영향을 미칩니다.

```sql
UPDATE Point2D SET (x, y) = (0, 0);

SELECT * FROM Point3D;
```

| x   | y   | z   |
| --- | --- | --- |
| 0   | 0   | 3   |
| 0   | 0   | 6   |
| 0   | 0   | 9   |

<br/>

이것을 방지하려면 `ONLY`를 붙이면 됩니다.

```sql
UPDATE ONLY Point2D SET (x, y) = (0, 0);

SELECT * FROM Point3D;
```

| x   | y   | z   |
| --- | --- | --- |
| 1   | 2   | 3   |
| 4   | 5   | 6   |
| 7   | 8   | 9   |

<br/>

`PostgreSQL`은 상속을 통해 `파티셔닝`을 구현하므로, 위의 내용은 파티셔닝된 테이블에도 동일하게 적용됩니다.

<br/>

#### `RETURNING` Clause

`INSERT`의 `RETURNING`절과 동일합니다.
