## SELECT

### 기본 구문

```sql
[ WITH [ RECURSIVE ] with_query [, ...] ]
SELECT [ ALL | DISTINCT [ ON ( expression [, ...] ) ] ]
    [ * | expression [ [ AS ] output_name ] [, ...] ]
    [ FROM from_item [, ...] ]
    [ WHERE condition ]
    [ GROUP BY grouping_element [, ...] ]
    [ HAVING condition [, ...] ]
    [ WINDOW window_name AS ( window_definition ) [, ...] ]
    [ { UNION | INTERSECT | EXCEPT } [ ALL | DISTINCT ] select ]
    [ ORDER BY expression [ ASC | DESC | USING operator ] [ NULLS { FIRST | LAST } ] [, ...] ]
    [ LIMIT { count | ALL } ]
    [ OFFSET start [ ROW | ROWS ] ]
    [ FETCH { FIRST | NEXT } [ count ] { ROW | ROWS } ONLY ]
    [ FOR { UPDATE | NO KEY UPDATE | SHARE | KEY SHARE } [ OF table_name [, ...] ] [ NOWAIT | SKIP LOCKED ] [...] ]
```

<br/>

### 동작 순서

1. `WITH` 목록에 정의된 요소들이 계산됩니다. 여기서 계산된 결과는 `FROM`절에서 사용할 수 있으며, `NOT MATERIALIZED`를 사용하지 않는 이상 `FROM`에서 2번 이상 사용되더라도 계산은 한번만 이루어집니다.

<br/>

2. `FROM` 목록에 정의된 요소들이 계산됩니다. 두 개 이상의 요소가 주어진 경우 `corss-join` 됩니다.

<br/>

3. 만약 `WHERE`절이 사용되었다면, 주어진 조건을 만족하지 않는 행은 결과집합에서 제외됩니다.

<br/>

4. 만약 `GROUP BY`절 또는 `집계함수`가 사용되었다면, 결과집합을 그룹화한 뒤에 집계함수의 결과를 계산합니다. 만약 `HAVING`절이 사용되었다면, 주어진 조건을 만족하지 않는 그룹은 결과집합에서 제외됩니다.

<br/>

5. `SELECT`절에 주어진 `행` 또는 `그룹`의 목록을 사용하여 실제 결과집합을 계산합니다.

<br/>

6. `SELECT DISTINCT`는 결과집합에서 중복된 행을 제거합니다. `SELECT ALL(default)`는 중복된 행도 포함하여 반환합니다.

<br/>

7. 집합 연산자 (`UNION`, `INTERSECT`, `EXCEPT`)를 계산합니다. `ALL`를 사용하지 않는 이상 결과집합에서 중복된 행은 제거됩니다.

<br/>

8. `ORDER BY`가 사용되었다면 해당 순서로 결과집합을 정렬합니다. 사용되지 않았다면 빨리 찾아낸 순서대로 반환됩니다.

<br/>

9. `LIMIT`(=`FETCH FIRST`) 또는 `OFFSET`이 사용되었다면, 결과집합에서 해당 부분집합만 반환합니다.

<br/>

10. `FOR UPDATE`, `FOR NO KEY UPDATE` 또는 `FOR KEY SHARE`이 사용되었다면 결과집합에 포함된 행에 대해 `LOCK`을 겁니다.

<br/>

### `WITH` Clause

`WITH`절은 서브쿼리를 가상의 뷰로 만들어 `FROM`절에 제공할 수 있도록 도와줍니다. 각 서브쿼리는 `SELECT`, `TABLE`, `VALUES`, `INSERT`, `UPDATE`, `DELETE` 중 하나가 올 수 있으며, 만약 부작용을 동반하는 문장(`INSERT`, `UPDATE`, `DELETE`)이라면 `RETURNING`절의 결과를 사용합니다. `RETURNING`절을 사용되지 않아도 에러를 발생시키지는 않지만, `FROM`절에서 해당 뷰를 참조할 수 없게 됩니다.

<br/>

`WITH RECURSIVE` 형태로 사용된다면 뷰가 자기 자신을 참조하는 것을 허용합니다. 이 경우 재귀참조는 아래처럼 `UNION [ALL | DISTINCT]`의 우측에 존재해야 합니다.

```sql
WITH RECURSIVE
test AS (
    NON-RECURSIVE-SELECT
    UNION [ALL | DISTINCT]
    RECURSIVE-SELECT
)
...
```

```sql
WITH RECURSIVE
test AS (
    SELECT 1 AS num
    UNION ALL
    SELECT num+1 FROM test WHERE num < 10
)
SELECT * FROM test;
```

![](./images/04-01.png)

```sql
CREATE TABLE test (
    child int,
    parent int NULL
);
INSERT INTO test VALUES (1, 5), (2, 5), (3, 5), (5, 10), (10, 15), (15, null);

WITH RECURSIVE
temp AS (
	-- first node.
    SELECT
		test.child,
		test.parent,
		1 AS depth,
		test.child::text AS path
	FROM test
	WHERE test.child = 1

	UNION ALL

	-- next node.
	SELECT
		test.child,
		test.parent,
		depth + 1,
		path || ' → ' || test.child
	FROM test, temp
	WHERE temp.parent = test.child
)
SELECT * FROM temp;
```

![](./images/04-02.png)

<br/>

`RECURSIVE`는 단 한번만 사용되어야 하며, 재귀를 사용하는 모든 뷰에 적용됩니다. 단, 재귀를 사용하지 않는 뷰에 대해서는 적용되지 않습니다.

```sql
WITH RECURSIVE
test1 AS (
    SELECT 1 AS num1
    UNION ALL
    SELECT num1+1 FROM test1 WHERE num1 < 1000
),
test2 AS (
    SELECT 1 AS num2
    UNION ALL
    SELECT num2+1 FROM test2 WHERE num2 < 1000
)
SELECT * FROM test1, test2 WHERE num1 = num2;
```

<br/>

자신보다 이전에 선언된 형제만 참조할 수 있습니다.

```sql
WITH
test1 AS (
    SELECT * FROM test2 -- ERROR
),
test2 AS (
    SELECT 1 AS x
)
SELECT * FROM test1;
```

<br/>

`PRIMARY QUERY`와 `WITH QUERY`는 동일한 시간값을 가지고 실행되므로 `WITH`에서 데이터를 수정해도 `RETUNING`을 제외한 다른 방법으로 변경된 데이터를 감지하는 것은 불가능합니다. 따라서 아래처럼 두 서브쿼리가 동일한 행을 수정하려고 하면 예측할 수 없는 결과가 만들어집니다.

```sql
CREATE TABLE test (
    val int
);
INSERT INTO test VALUES (0);

WITH
m1 AS (
    UPDATE test SET val = 1 WHERE val = 0
),
m2 AS (
    UPDATE test SET val = 2 WHERE val = 0
)
SELECT * FROM test;
```

<br/>

`WITH`에 사용된 서브쿼리는 여러번 참조되어도 한 번만 계산되도록 제한되어 있기 때문에 별도의 실행계획을 사용하지만, `NOT MATERIALIZED`가 함께 사용하면 이러한 제한을 제거할 수 있으며, 주 쿼리와 서브 쿼리를 풀어내어 계산하므로 더 좋은 실행계획을 찾을 가능성이 높아집니다. 그러나 이것은 휘발성이 존재하는 쿼리인 경우에만 작동하고, 휘발성이 아닌 쿼리(`재귀적`이거나 `부작용이 없는` 쿼리)라고 판단되면 `NOT MATERIALIZED`는 무시됩니다.
