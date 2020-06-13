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

<br/>

### `FROM` Clause

```sql
where from_item can be one of:

    [ ONLY ] table_name [ * ] [ [ AS ] alias [ ( column_alias [, ...] ) ] ]
                [ TABLESAMPLE sampling_method ( argument [, ...] ) [ REPEATABLE ( seed ) ] ]
    [ LATERAL ] ( select ) [ AS ] alias [ ( column_alias [, ...] ) ]
    with_query_name [ [ AS ] alias [ ( column_alias [, ...] ) ] ]
    [ LATERAL ] function_name ( [ argument [, ...] ] )
                [ WITH ORDINALITY ] [ [ AS ] alias [ ( column_alias [, ...] ) ] ]
    [ LATERAL ] function_name ( [ argument [, ...] ] ) [ AS ] alias ( column_definition [, ...] )
    [ LATERAL ] function_name ( [ argument [, ...] ] ) AS ( column_definition [, ...] )
    [ LATERAL ] ROWS FROM( function_name ( [ argument [, ...] ] ) [ AS ( column_definition [, ...] ) ] [, ...] )
                [ WITH ORDINALITY ] [ [ AS ] alias [ ( column_alias [, ...] ) ] ]
    from_item [ NATURAL ] join_type from_item [ ON join_condition | USING ( join_column [, ...] ) ]
```

`FROM`절은 해당 쿼리에서 사용할 데이터 소스의 목록을 지정하는 역할을 합니다. 만약 여러개의 소스가 지정되었다면, 모든 소스를 카테시안 곱셈(`CROSS-JOIN`)한 결과를 반환합니다. 이 때, `WHERE`절을 명시적으로 추가하여 카테시안 곱셈의 작은 부분집합을 가져올 수 있습니다.

<br/>

데이터 소스는 다음 중 하나를 허용합니다.

-   테이블
-   서브쿼리
-   `WITH`절에 선언된 쿼리
-   함수호출

<br/>

데이터 소스로 `테이블`을 사용할 수 있습니다. 만약 `ONLY`를 테이블 이름의 앞에 사용했다면, 해당 테이블만 스캔 되도독(=자식 테이블이 스캔되지 않도록) 지정할 수 있습니다. 만약 `*`를 테이블 이름의 뒤에 사용했다면, 명시적으로 모든 자식 테이블을 데이터 소스로 지정할 수 있습니다.

```sql
-- X의 자식 테이블은 제외.
FROM ONLY X;

-- X의 모든 자식 테이블도 포함.
FROM X;
FROM X*;
```

<br/>

데이터 소스로 `서브쿼리`를 사용할 수 있습니다. 이 경우, 해당 서브쿼리는 `Temporary Table`로 구체화되어 쿼리에서 사용됩니다. 각각의 서브쿼리는 반드시 괄호로 감싸야하며 `alias`도 함께 주어져야 합니다. `VALUES ...` 또한 서브쿼리로 사용할 수 있습니다.

```sql
SELECT *
FROM ( SELECT 1 AS x, 2 AS y ) AS TEST;

-- 또는
SELECT *
FROM ( SELECT 1, 2 ) AS TEST(x, y);

-- 또는
SELECT *
FROM ( VALUES (1, 2) ) AS TEST(x, y);

-- Multiple Values
SELECT *
FROM ( VALUES (1, 2), (3, 4), ... ) AS TEST(x, y);
```

<br/>

데이터 소스로 `WITH절에 선언된 쿼리`를 사용할 수 있습니다. 만약 `WITH-QUERY`의 이름과 중복되는 테이블이 있다면 `WITH-QUERY`가 우선적으로 사용됩니다. 즉, 기존 테이블의 이름은 가려집니다.

```sql
-- 테이블 생성
CREATE TABLE TEST (
    x int
);
INSERT INTO TEST VALUES (1), (2), (3);

-- 기존 테이블의 이름과 겹치도록 WITH 쿼리를 생성
WITH TEST AS (
    SELECT * FROM ( VALUES (4), (5), (6) ) AS TEST(y)
)
SELECT * FROM TEST;
```

![](./images/04-03.png)

<br/>

데이터 소스로 `함수호출`을 사용할 수 있습니다. 일반적으로는 모든 함수를 사용할 수 있지만, 보편적으로는 집합을 반환하는 함수가 사용됩니다. `WITH ORDINALITY`가 함께 사용되면, 마지막 컬럼에 `bigint`형식의 데이터 순번이 부여됩니다.

```sql
SELECT *
FROM   unnest( array['x', 'y', 'z'] ) AS test(token);
```

![](./images/04-04.png)

```sql
SELECT *
FROM   unnest(array['x', 'y', 'z']) WITH ORDINALITY AS test(token, seq);
```

![](./images/04-05.png)

<br/>

기본적으로 `FROM`에서 `Right-Hand Element`는 `Left-Hand Element`의 데이터를 읽을 수 없습니다. 그러나 `SUB-SELECT` 또는 `FUNCTION-CALL`의 앞에 `LATERAL` 키워드를 함께 사용하면, 해당 요소의 좌측에서 선언된 데이터를 읽을 수 있습니다.

```sql
-- 샘플 데이터 테이블
CREATE TABLE test (
	x int,
	y int
);
INSERT INTO test VALUES (1, 1), (1, 2), (1, 3), (2, 1),  (2, 5), (2, 6);
```

먼저 잘못된 쿼리부터 살펴보겠습니다. 우측 인라인뷰에서 `(test.)x`와 `(test.)y`를 읽으려고 했지만, 기본적으로 좌측의 테이블의 데이터를 읽을 수 없으므로 에러가 발생합니다.

```sql
SELECT *
FROM   test, ( SELECT x + y ) as sub("x+y");

[output]
ERROR:  column "x" does not exist
There is a column named "x" in table "test", but it cannot be referenced from this part of the query.
```

하지만 `LATERAL` 키워드를 함께 사용하면 좌측의 `test`의 컬럼을 읽을 수 있습니다.

```sql
SELECT *
FROM   test, LATERAL ( SELECT x + y ) as sub("x+y");
```

![](./images/04-06.png)

이것을 `JOIN`과 함께 사용하면 `상호 연관 서브쿼리`와 결과가 같아집니다. 예를 들어, 각 `x`마다 그룹을 지어 `y`가 평균보다 큰 행만 출력하는 작업을 `상호 연관 서브쿼리`로 작성해보면 다음과 같습니다.

```sql
SELECT *
FROM   test t1
WHERE  y > (
    SELECT avg(t2.y)
    FROM   test t2
    WHERE  t2.x = t1.x
);
```

![](./images/04-07.png)

위의 쿼리가 동작하는 방식은 다음과 같습니다.

1. `메인쿼리`에서 `t1.x`를 읽을 때 마다 `서브쿼리`에 전달합니다.
2. `서브쿼리`에서 `t1.x`에 해당하는 `t2.y`를 읽어 평균을 구하고 반환합니다.
3. `WHERE`절에서 평균보다 큰 행만 가져옵니다.

`메인쿼리`에서 참조되는 데이터가 읽힐 때 마다 `서브쿼리`에 삽입되어 호출됩니다. 호출된 횟수는 실행계획의 `LOOP` 항목을 통해 확인할 수 있습니다.

![](./images/04-09.png)

<br/>

이것과 동일한 결과를 반환하는 쿼리를 `LATERAL JOIN`으로도 작성할 수 있습니다.

```sql
SELECT t1.*
FROM   test as t1 JOIN LATERAL (
    SELECT avg(t2.y)
    FROM   test as t2
    WHERE  t2.x = t1.x
) as sub
ON    t1.y > avg;
-- OR
WHERE t1.y > avg;
```

위의 쿼리가 동작하는 방식은 다음과 같습니다.

1. `좌측`에서 `t1.x`를 읽을 때 마다 `우측`에 전달합니다.
2. `우측`에서 `t1.x`에 해당하는 `t2.y`를 읽어 평균을 구합니다.
3. `ON`절 또는 `WHERE`절에서 평균보다 큰 행만 가져옵니다.

`좌측쿼리`에서 참조되는 데이터가 읽힐 때 마다 `우측쿼리(=LATERAL Query)`에 삽입되어 호출됩니다. 호출된 횟수는 실행계획의 `LOOP` 항목을 통해 확인할 수 있습니다.

![](./images/04-08.png)

<br/>

`LATERAL`은 함수호출의 앞에 사용할 수 있지만, 이 경우에는 `Noise Word`입니다. 있으나 없으나 `LATERAL`로 간주되기 때문입니다.

```sql
SELECT  temp2.*
FROM   (VALUES (array[1, 2, 3], array[4, 5, 6])) as temp1(x, y),
       unnest(x, y) as temp2(a, b); -- Implicit LATERAL
```

![](./images/04-10.png)

<br/>

### `WHERE` Clause

```sql
WHERE boolean_expression
```

주어진 표현식이 `true`인 행만 결과집합에 포함시킵니다.

<br/>

`boolean`을 반환하는 연산자는 다음이 있습니다.

**동등 및 대소비교:**

-   `=`
-   `>`
-   `<`
-   `>=`
-   `<=`
-   `<>` or `!=`
-   `BETWEEN a AND b` : 주어진 숫자가 `a`이상 `b`이하라면 `true`.

```sql
SELECT 1 = 1; -- true
SELECT 2 != 2 -- false
SELECT 1 BETWEEN 1 AND 3; -- true
SELECT 3 BETWEEN 1 AND 3; -- true
```

<br/>

**논리 연산자 :**

-   `NOT`
-   `AND`
-   `OR`

```sql
SELECT true AND true; -- true
SELECT true OR false; -- true
SELECT NOT false; -- true
```

<br/>

**LIKE 표현식 :**

-   `str LIKE exp` : `str`이 `exp`에 일치하면 `true`를 반환합니다.

`LIKE`에 사용할 수 있는 특수기호는 다음과 같습니다.

| 기호 | 의미                    |
| :--: | ----------------------- |
|  \_  | 임의의 한 글자에 매칭   |
|  %   | 임의의 여러 글자에 매칭 |

```sql
SELECT 'Hello, World!' LIKE 'H_'; -- false
SELECT 'Hello, World!' LIKE 'H%'; -- true
SELECT 'Hello, World!' LIKE '%W%'; -- true
```

<br/>

**SMILAR TO 정규 표현식 :**

-   `str SIMILAR TO exp`
-   `str NOT SIMILAR TO exp`

`SIMILAR TO`에 사용할 수 있는 특수기호는 다음과 같습니다.

|  기호  | 의미                                          |
| :----: | --------------------------------------------- |
|   \|   | 대체자. a 또는 b                              |
|   \*   | 앞의 아이템이 0개 이상인 경우에 매칭          |
|   +    | 앞의 아이템이 1개 이상인 경우에 매칭          |
|  {m}   | 앞의 아이템이 m개인 경우에 매칭               |
|  {m,}  | 앞의 아이템이 m개 이상인 경우에 매칭          |
| {m, n} | 앞의 아이템이 m개 이상 n개 이하인 경우에 매칭 |
|   ()   | 여러 아이템을 하나의 아이템으로 묶음          |
| [...]  | 문자열을 각 문자로 쪼갬                       |

```sql
SELECT 'Hello, World!' SIMILAR TO '[a-zA-Z]*'; -- false
SELECT 'Hello, Hello!' SIMILAR TO '(Hello(,|!) ){2}'; -- true
```

<br/>

**POSIX 정규 표현식 :**

-   `str ~ pattern` : 대소문자를 `엄격히` 비교하여 `str`이 `pattern`에 일치하면 `true`.
-   `str ~* pattern`: 대소문자를 `느슨하게` 비교하여 `str`이 `pattern`에 일치하면 `true`.
-   `str !~ pattern` : 대소문자를 `엄격히` 비교하여 `str`이 `pattern`에 일치하지 않으면 `true`.
-   `str !~* pattern` : 대소문자를 `느슨하게` 비교하여 `str`이 `pattern`에 일치하지 않으면 `true`.

```sql
SELECT 'Hello, World!' ~ '^H.*!$'; -- true;
```

자세한 사항은 공식문서의 [POSIX-REGEXP](https://www.postgresql.org/docs/12/functions-matching.html#FUNCTIONS-POSIX-REGEXP)를 참조해주세요.

<br/>

**포함 및 미포함 :**

-   `IN` : 주어진 항목이 리스트에 있다면 `true`.
-   `NOT IN` : 주어진 항목에 리스트에 없다면 `true`.
-   `EXISTS` : 주어진 서브쿼리가 공집합이 아니라면 `true`.
-   `NOT EXISTS` : 주어진 서브쿼리가 공집합이 이라면 `true`.

```sql
-- Single Column.
SELECT 1 IN (1, 2, 3); -- true

-- Multi Column.
SELECT (1, 2) NOT IN ((1, 2), (3, 4), (5, 6)); -- false
```

```sql
SELECT EXISTS ( SELECT 1 ); -- true;

SELECT NOT EXISTS ( SELECT 1 WHERE false ); -- true;
```
