## 실행계획

### 실행계획이란?

쿼리가 요구한 데이터를 추출하기 위해 `DBMS`가 차례로 수행하는 전략.

<br/>

### 샘플 데이터 생성

```sql
-- Create Table.
CREATE TABLE Location (
    id int PRIMARY KEY,
    x int NOT NULL,
    y int NOT NULL,
    z int NOT NULL
);

CREATE TABLE Building (
    location_id int PRIMARY KEY REFERENCES Location(id),
    name text
);


-- Create Data.
INSERT INTO Location
SELECT
    id,
    trunc(random()*10000) AS x,
    trunc(random()*10000) AS y,
    trunc(random()*10000) AS z
FROM generate_series(0, 1000000) AS id;

INSERT INTO Building
SELECT
    trunc(random()*1000000) AS location_id,
    md5(random()::text) AS name
FROM generate_series(0, 1000000)
ON CONFLICT (location_id) DO NOTHING;


-- Create Index.
CREATE INDEX idx_location_x_y ON Location(x, y);
CREATE INDEX idx_location_z ON Location(z);
```

<br/>

## 데이터 읽기

### Seq Scan

테이블의 `모든 페이지`를 읽어서 모든 데이터를 메모리에 적재합니다. 건당 읽기속도가 매우 빠르기 때문에, 대부분의 튜플이 요구되거나, 테이블의 크기가 작다면, `인덱스 스캔`보다 훨씬 빠르게 데이터를 가져올 수 있습니다. 단, 모든 튜플을 읽기 때문에, 대용량 테이블에서 소량의 데이터만 추출하는 경우에는 병목으로 작용할 수 있습니다.

![](./images/09-01.png);

```sql
EXPLAIN
SELECT * FROM LOCATION;
```

```text
 QUERY PLAN
-------------------------------------------------------------------
 Seq Scan on location  (cost=0.00..15406.01 rows=1000001 width=16)
```

<br/>

### Index Scan

인덱스를 타고 `조건절`을 만족하는 데이터만 메모리에 적재합니다. 건당 읽기속도는 `순차 스캔`보다는 느리지만, 조건절을 만족하는 행만 가져올 수 있으므로, 대용량 테이블에서 소량의 데이터를 가져오는 경우에 적합합니다.

![](./images/09-02.png);

```text
EXPLAIN SELECT * FROM LOCATION
WHERE x = 404 AND y = 404;

 QUERY PLAN
----------------------------------------------------------------------------------
 Index Scan using idx_location_x_y on location  (cost=0.42..8.45 rows=1 width=16)
   Index Cond: ((x = 404) AND (y = 404))
```

<br/>

단, 테이블의 대부분을 읽어야 된다면 `순차 스캔`이 선택될 수 있습니다.

![](./images/09-03.png);

```text
EXPLAIN SELECT * FROM LOCATION
WHERE x != 404 AND y != 404;

 QUERY PLAN
------------------------------------------------------------------
 Seq Scan on location  (cost=0.00..20406.01 rows=999801 width=16)
   Filter: ((x <> 404) AND (y <> 404))
```

<br/>

### Index Only Scan

인덱스 이외의 컬럼을 사용하지 않았다면, 인덱스만 읽어도 모든 데이터를 가져올 수 있습니다. 즉, 테이블 액세스가 발생하지 않습니다.

```text
EXPLAIN SELECT x, y FROM LOCATION
WHERE x < 5 AND y < 5;

 QUERY PLAN
---------------------------------------------------------------------------------------
 Index Only Scan using idx_location_x_y on location  (cost=0.42..24.26 rows=1 width=8)
   Index Cond: ((x < 5) AND (y < 5))
```

<br/>

### Bitmap Index Scan

`순차 스캔`으로 읽기에는 적고 `인덱스 스캔`으로 읽기에는 많은 경우, `PostgreSQL`은 `비트맵 인덱스 스캔`을 선택합니다. 먼저 인덱스를 사용하여 `읽어야 할 페이지`에 체크한 뒤, 체크된 페이지만 빠르게 읽습니다.

![](./images/09-04.png);

```text
EXPLAIN SELECT * FROM LOCATION
WHERE x < 100;

 QUERY PLAN
------------------------------------------------------------------------------------
 Bitmap Heap Scan on location  (cost=176.48..5866.96 rows=9298 width=16)
   Recheck Cond: (x < 100)
   ->  Bitmap Index Scan on idx_location_x_y  (cost=0.00..174.16 rows=9298 width=0)
         Index Cond: (x < 100)
```

<br/>

여러개의 인덱스를 함께 사용할 수 있습니다.

![](./images/09-05.png)

```text
EXPLAIN SELECT * FROM LOCATION
WHERE x < 100 AND z < 100;

 QUERY PLAN
------------------------------------------------------------------------------------------
 Bitmap Heap Scan on location  (cost=354.02..673.65 rows=88 width=16)
   Recheck Cond: ((x < 100) AND (z < 100))
   ->  BitmapAnd  (cost=354.02..354.02 rows=88 width=0)
         ->  Bitmap Index Scan on idx_location_x_y  (cost=0.00..174.16 rows=9298 width=0)
               Index Cond: (x < 100)
         ->  Bitmap Index Scan on idx_location_z  (cost=0.00..179.56 rows=9485 width=0)
               Index Cond: (z < 100)
```

<br/>

`OR`도 가능합니다.

```text
EXPLAIN SELECT * FROM LOCATION
WHERE x < 100 OR z < 100;

 QUERY PLAN
------------------------------------------------------------------------------------------
 Bitmap Heap Scan on location  (cost=363.07..6050.81 rows=18695 width=16)
   Recheck Cond: ((x < 100) OR (z < 100))
   ->  BitmapOr  (cost=363.07..363.07 rows=18783 width=0)
         ->  Bitmap Index Scan on idx_location_x_y  (cost=0.00..174.16 rows=9298 width=0)
               Index Cond: (x < 100)
         ->  Bitmap Index Scan on idx_location_z  (cost=0.00..179.56 rows=9485 width=0)
               Index Cond: (z < 100)
```

<br/>

### Parallel Seq Scan

병렬로 데이터를 읽어, 각각에서 읽은 결과를 합칩니다(=`Gahter`). 조건절이 주어졌으나 연관된 인덱스가 없는 경우, 조건을 만족하는 행을 찾아내기 위한 차선책으로 사용됩니다. 결과 건수가 낮을수록 그나마 비용도 낮아집니다.

![](./images/09-06.png)

**결과 집합이 적은 경우 :**

```text
EXPLAIN SELECT * FROM LOCATION
WHERE y = 100;

                                 QUERY PLAN
-----------------------------------------------------------------------------
 Gather  (cost=1000.00..11624.34 rows=100 width=16)
   Workers Planned: 2
   ->  Parallel Seq Scan on location  (cost=0.00..10614.34 rows=42 width=16)
         Filter: (y = 100)
```

<br/>

**결과 집합이 많은 경우 :**

```text
EXPLAIN SELECT * FROM LOCATION
WHERE y BETWEEN 10 AND 100;

                                 QUERY PLAN
-----------------------------------------------------------------------------
 Gather  (cost=1000.00..13472.71 rows=8167 width=16)
   Workers Planned: 2
   ->  Parallel Seq Scan on location  (cost=0.00..11656.01 rows=3403 width=16)
         Filter: ((y >= 10) AND (y <= 100))
```

<br/>

### With `Limit` Clause

#### On Seq Scan

`Limit`절을 사용하여 `Seq Scan`의 범위를 좁힐 수 있습니다.

![](./images/09-07.png)

**Limit을 사용하지 않은 경우 :**

```text
EXPLAIN SELECT * FROM LOCATION;

 QUERY PLAN
-------------------------------------------------------------------
 Seq Scan on location  (cost=0.00..15406.01 rows=1000001 width=16)
```

<br/>

**Limit을 사용한 경우 :**

```text
EXPLAIN SELECT * FROM LOCATION LIMIT 10;

 QUERY PLAN
-------------------------------------------------------------------------
 Limit  (cost=0.00..0.15 rows=10 width=16)
   ->  Seq Scan on location  (cost=0.00..15406.01 rows=1000001 width=16)
```

<br/>

#### On Index Scan

선택된 행이 너무 많아 `Seq Scan`으로 변경되는 경우에 사용하면 스캔 범위를 좁혀서 비용을 줄일 수 있습니다. `Limit`을 사용한 경우, 최종 최대비용이 `0.18`로 줄어든 것에 주목해주세요.

**Limit을 사용하지 않은 경우 :**

```text
EXPLAIN SELECT * FROM LOCATION WHERE x > 10;

 QUERY PLAN
-------------------------------------------------------------------
 Seq Scan on location  (cost=0.00..17906.01 rows=998671 width=16)
   Filter: (x > 10)
```

<br/>

**Limit을 사용한 경우 :**

```text
EXPLAIN SELECT * FROM LOCATION WHERE x > 10 LIMIT 10;

 QUERY PLAN
------------------------------------------------------------------------
 Limit  (cost=0.00..0.18 rows=10 width=16)
   ->  Seq Scan on location  (cost=0.00..17906.01 rows=998671 width=16)
         Filter: (x > 10)
```

<br/>

#### On Bitmap Index Scan

`Index Scan`의 결과행이 많아서 `Bitmap Index Scan`으로 변질된 경우, `Limit`을 사용하여 `Index Scan`으로 되돌릴 수 있습니다.

**Limit을 사용하지 않은 경우 :**

```text
EXPLAIN SELECT * FROM LOCATION WHERE x BETWEEN 10 AND 100;

 QUERY PLAN
-------------------------------------------------------------------
 Bitmap Heap Scan on location  (cost=176.16..5960.13 rows=8169 width=16)
   Recheck Cond: ((x >= 10) AND (x <= 100))
   ->  Bitmap Index Scan on idx_location_x_y  (cost=0.00..174.12 rows=8169 width=0)
         Index Cond: ((x >= 10) AND (x <= 100))
```

<br/>

**Limit을 사용한 경우 :**

```text
EXPLAIN SELECT * FROM LOCATION WHERE x BETWEEN 10 AND 100 LIMIT 10;

 QUERY PLAN
------------------------------------------------------------------------
 Limit  (cost=0.42..23.53 rows=10 width=16)
   ->  Index Scan using idx_location_x_y on location  (cost=0.42..18871.09 rows=8169 width=16)
         Index Cond: ((x >= 10) AND (x <= 100))
```

<br/>

### With `Offset` Clause

`Offset`절은 실행계획에서 `Limit`으로 나타나지만 범위 축소에는 기여하지 않습니다.

**Offset 사용 전:**

```text
EXPLAIN SELECT * FROM LOCATION;

 QUERY PLAN
-------------------------------------------------------------------
 Seq Scan on location  (cost=0.00..15406.01 rows=1000001 width=16)
```

<br/>

**Offset 사용 후:**

```text
EXPLAIN SELECT * FROM LOCATION OFFSET 100000;

 QUERY PLAN
-------------------------------------------------------------------------
 Limit  (cost=1540.60..15406.01 rows=900001 width=16)
   ->  Seq Scan on location  (cost=0.00..15406.01 rows=1000001 width=16)
```
