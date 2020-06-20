## Delete

### 기본 구문

```sql
[ WITH [ RECURSIVE ] with_query [, ...] ]
DELETE FROM [ ONLY ] table_name [ * ] [ [ AS ] alias ]
    [ USING from_item [, ...] ]
    [ WHERE condition | WHERE CURRENT OF cursor_name ]
    [ RETURNING * | output_expression [ [ AS ] output_name ] [, ...] ]
```

<br/>

### 기본 설명

`명시적으로 주어진 조건`을 충족하는 모든 행을 삭제합니다. 주건이 주어지지 않는다면 모든 행을 삭제합니다. 다만, 모든 행을 삭제하려는 경우 `TRUNCATE`가 훨씬 더 빠르게 작동합니다. 그 이유에 대해서는 `TRUNCATE`장에서 설명합니다.

<br/>

### 샘플 테이블

```sql
CREATE TABLE Point2D (
    x int,
    y int
);

INSERT INTO Point2D VALUES
    (1, 1),
    (2, 2),
    (3, 3);
```

<br/>

### 기본 사용법

`WHERE`절로 삭제할 데이터의 조건을 명시하는 것이 가장 기본적인 사용법입니다. 조건이 명시되지 않으면 모든 행을 삭제하지만, 이 경우에는 `TRUNCATE`가 더 유용한 해결책임을 잊으면 안됩니다.

```sql
DELETE FROM Point2D WHERE x=2 OR x=3;
```

| x   | y   |
| --- | --- |
| 1   | 1   |

<br/>

### `USING` Clause

외부 테이블을 참조하여 데이터를 제거해야 한다면, `USING`절을 사용하여 외부 테이블을 불러올 수 있습니다.

```sql
CREATE TABLE Target (
    x int
);

INSERT INTO Target VALUES (1), (2);

DELETE FROM Point2D USING Target WHERE Point2D.x = Target.x;
```

| x   | y   |
| --- | --- |
| 3   | 3   |

<br/>

### `RETURNING` Clause

삭제된 행을 반환합니다. 기본적인 사용법은 `INSERT`의 `RETRUNING`과 동일합니다.

```sql
WITH RES AS (
    DELETE FROM Point2D WHERE x=1 OR x=2
    RETURNING *
)
SELECT * FROM RES;
```

| x   | y   |
| --- | --- |
| 1   | 1   |
| 2   | 2   |

<br/>

### 표준과의 호환성 비교

다음 기능은 `SQL Standard`에서 정의된 표준이 아닙니다.

-   `RETURNING` Clause
-   `USING` Clause
-   `WITH` Clause

---

## Truncate

### 기본 구문

```sql
TRUNCATE [ TABLE ] [ ONLY ] name [ * ] [, ... ]
    [ RESTART IDENTITY | CONTINUE IDENTITY ] [ CASCADE | RESTRICT ]
```

<br/>

### 기본 설명

주어진 테이블들의 모든 데이터를 신속하게 제거합니다. 조건이 명시되지 않은 `DELETE`과 동일하지만 `할당된 페이지를 회수`하는 방식을 사용하여 테이블 스캔 없이 데이터를 제거할 수 있습니다. 이것은 대용량 테이블에서 매우 유용합니다.

<br/>

### `DELETE`와의 차이점

-   `DELETE`는 행을 삭제할 때 마다, 그 로그를 기록하므로 느립니다.
    `TRUNCATE`는 페이지 할당을 취소하고, 그 로그를 기록하므로 매우 빠릅니다.

<br/>

-   `DELETE`는 테이블 제거를 위해 스캔을 진행하지만,
    `TRUNCATE`는 단순히 페이지 할당을 취소하기 때문에 스캔작업이 없습니다.

<br/>

-   `DELETE`는 행 마다 락을 걸지만,
    `TRUNCATE`는 테이블과 페이지에만 락을 겁니다. (락이 더 적게 사용됨.)

<br/>

### `DROP`과의 차이점

-   `DROP`과 `TRUNCATE` 모두 페이지 할당을 취소합니다.

<br/>

-   `DROP`은 테이블 그 자체를 삭제하지만,
    `TRUNCATE`는 테이블은 유지합니다.

<br/>

### 기본 사용법

단일 테이블을 초기화합니다.

```sql
CREATE TABLE Point2D (
    x int,
    y int
);

INSERT INTO Point2D VALUES
    (1, 1),
    (2, 2),
    (3, 3);

TRUNCATE Point2D;
```

<br/>

또는, 여러개의 테이블을 한꺼번에 초기화할 수 있습니다.

```sql
TRUNCATE Point2D, Point3D;
```

<br/>

### `IDENTIFY` Option

테이블과 각 컬럼에 관련된 시퀀스를 유지하거나 초기화 할지 선택할 수 있습니다.

-   `RESTART IDENTITY` : 시퀀스를 초기화합니다.
-   `CONTINUE IDENTITY` : 시퀀스를 유지합니다. (default)

<br/>

### `REFERENCES` Contraint Option

해당 테이블을 초기화 할 때, 다른 테이블에서 `참조 제약 위반`이 발생한다면 어떻게 행동할지 설정할 수 있습니다.

-   `CASCADE` : 다른 테이블도 싹 초기화합니다.
-   `RESTRICT` : 초기화를 거부하고 에러를 발생시킵니다. (default)

```sql
CREATE TABLE Point2D (
    x int,
    y int,
    -- 누군가가 자신을 참조하려면 UNIQUE 제약이 있어야 한다.
    CONSTRAINT pk_test1 PRIMARY KEY(x, y)
);

CREATE TABLE Point3D (
    x int,
    y int,
    z int,
    CONSTRAINT fk_point3d FOREIGN KEY (x, y) REFERENCES Point2D(x, y)
);

INSERT INTO Point2D VALUES
    (1, 1),
    (2, 2),
    (3, 3);

INSERT INTO Point3D VALUES
    (1, 1, 1),
    (2, 2, 2),
    (3, 3, 3);
```

<br/>

**CASCADE :**

```sql
TRUNCATE Point2D CASCADE;
SELECT * FROM Point2D; -- EMPTY
SELECT * FROM Point3D; -- EMPTY
```

<br/>

**RESCTRICT :**

```sql
TRUNCATE Point2D; -- ERROR
TRUNCATE Point2D RESTRICT; -- ERROR.
```

---

## MVCC 이슈

`TRUNCATE`와 일부 `DELETE` 명령어는 `MVCC` 매커니즘에 대해 안전하지 않습니다. 데이터가 삭제된 이후에 커밋이 이루어지면, 모든 세션의 모든 스냅샷이 제거되기 때문입니다.

<br/>

이것은 커밋 이전의 스냅샷을 사용하고, 아직 테이블에 접근하지 않은 트랜잭션에 대해 문제를 일으킬 수 있습니다.

<br/>

비어있지 않은 테이블에 대해 `SELECT`를 시도하는 `TX1`과, `Truncate`를 시도하는 `TX2`를 동시에 실행시켜보겠습니다.

| Timing | TX1              | TX2        |
| :----: | ---------------- | ---------- |
|   1    | `BEGIN`          |            |
|   2    |                  | `BEGIN`    |
|   3    |                  | `TRUNCATE` |
|   4    |                  | `COMMIT`   |
|   5    | `SELECT` (empty) |            |
|   6    | `COMMIT`         |            |

<br/>

`TX1`이 먼저 실행되었기 때문에 `TX2`보다 이전의 스냅샷을 사용하므로, `TX2`에서 데이터를 제거해도 `TX1`에는 영향이 없을 것 처럼 예상되지만, 실제로는 `SELECT` 시점에서 스냅샷을 따라 데이터를 조회하려고 시도하면 `페이지`가 전부 회수되었기 때문에 빈 테이블로 조회될 것 입니다.

<br/>

이러한 상황은 트랜잭션이 `TRUNCATE`를 이전에 시작되었고, 아직 테이블에 액세스 하지 않은 경우에만 발생합니다. 일단 해당 테이블에 무언가 작업을 시도하면, 최소 `ACCESS SHARE`수준의 테이블 락이 걸리기 때문에, 하나의 트랜잭션이 그 테이블을 연속적으로 사용했을 때에는 불일치가 발생하지 않습니다.

<br/>

하지만 어떤 트랜잭션이 이미 데이터를 빼간 상황에서, 다른 트랜잭션이 `TRUNCATE`를 실행하고 커밋한 경우, 가져온 데이터와 실제 테이블에는 불일치가 발생됩니다.
