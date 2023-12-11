---
layout: post
title: SQL JOIN 문을 비교해보자
date: 2023-12-11
---

두 테이블 사이의 연관된 데이터를 조회할 때 SQL의 JOIN 문을 사용할 수 있습니다. JOIN 문에는 어떤 것들이 있는지 알아보고, 각각의 특징을 비교해보겠습니다.

## 테이블 정의

```sql
-- Database : MySQL
CREATE TABLE student (
     id         INT PRIMARY KEY,
     name       VARCHAR(40),
     major      VARCHAR(40)
);

CREATE TABLE course (
    id          INT PRIMARY KEY,
    student_id  INT,
    name        VARCHAR(40),
    credit      FLOAT
);
```

{% include image.html url='/assets/image/student_table.png' description='fig 1. Student Table' %}
{% include image.html url='/assets/image/cource_table.png' description='fig 2. Course Table'%}


## INNER JOIN (내부 조인)

INNER JOIN 은 조인 컬럼을 기준으로 두 테이블 사이의 공통된 데이터만을 조회합니다. INNER JOIN 은 다음과 같이 사용할 수 있습니다.

```sql
-- 명시적 내부 조인 (Explicit Inner Join)
SELECT student.id, student.name, course.id, course.name, course.credit
FROM student INNER JOIN course ON student.id = course.student_id;
```

{% include image.html url='/assets/image/inner_join.png' description='fig 3. Inner Join Result'%}

묵시적 내부 조인이라 하여 다음과 같이 사용할 수도 있습니다. 결과는 위와 동일하며, 조금 더 간단하게 사용할 수 있습니다.

```sql
-- 묵시적 내부 조인 (Implicit Inner Join)
SELECT student.id, student.name, course.id, course.name, course.credit
FROM student, course
WHERE student.id = course.student_id;
```

## OUTER JOIN (외부 조인)

OUTER JOIN 은 조인 컬럼을 기준으로 두 테이블 사이의 공통된 데이터와 공통되지 않은 데이터를 모두 조회합니다. OUTER JOIN 은 다음 세 가지로 나뉩니다.

- LEFT OUTER JOIN
- RIGHT OUTER JOIN
- FULL OUTER JOIN

### LEFT OUTER JOIN (왼쪽 외부 조인)

LEFT OUTER JOIN 은 조인 컬럼을 기준으로 왼쪽 테이블의 모든 데이터와 오른쪽 테이블의 공통된 데이터를 조회합니다. 오른쪽 테이블의 공통된 데이터가 없다면 `NULL` 로 표시됩니다.

```sql
SELECT student.id, student.name, course.id, course.name, course.credit
FROM student LEFT OUTER JOIN course ON student.id = course.student_id;
```

{% include image.html url='/assets/image/left_outer_join.png' description='fig 4. Left Outer Join Result'%}

위 결과에서 `student_id = 4` 인 컬럼은 course 테이블에 없기 때문에 `NULL` 로 표시됩니다.

### RIGHT OUTER JOIN (오른쪽 외부 조인)

RIGHT OUTER JOIN 은 조인 컬럼을 기준으로 오른쪽 테이블의 모든 데이터와 왼쪽 테이블의 공통된 데이터를 조회합니다. 왼쪽 테이블의 공통된 데이터가 없다면 `NULL` 로 표시됩니다.

```sql
SELECT student.id, student.name, course.id, course.name, course.credit
FROM student RIGHT OUTER JOIN course ON student.id = course.student_id;
```

{% include image.html url='/assets/image/right_outer_join.png' description='fig 5. Right Outer Join Result'%}

위 결과에서 `student_id = 5` 인 컬럼은 student 테이블에 없기 때문에 `NULL` 로 표시됩니다. 결국 RIGHT OUTER JOIN 과 LEFT OUTER JOIN 은 쿼리에 사용된 테이블의 순서만 바뀐다면 동일한 결과를 얻을 수 있으므로, 둘 중 하나만 사용하면 됩니다.

### FULL OUTER JOIN (전체 외부 조인)

FULL OUTER JOIN 은 두 테이블의 모든 데이터를 조회합니다. LEFT OUTER JOIN 과 RIGHT OUTER JOIN 의 결과를 합친 것과 동일합니다.

```sql
SELECT student.id, student.name, course.id, course.name, course.credit
FROM student FULL OUTER JOIN course ON student.id = course.student_id;
```

PostgresQL, Oracle 에서는 FULL OUTER JOIN 을 지원하지만, MySQL 에서는 FULL OUTER JOIN 을 지원하지 않습니다. 따라서 FULL OUTER JOIN 과 동일한 결과를 얻으려면 다음과 같이 LEFT OUTER JOIN 과 RIGHT OUTER JOIN 을 합쳐서 사용해야 합니다.

```sql
-- MySQL 에서 FULL OUTER JOIN 을 사용하는 방법
SELECT student.id, student.name, course.id, course.name, course.credit
FROM student LEFT OUTER JOIN course ON student.id = course.student_id
UNION
SELECT student.id, student.name, course.id, course.name, course.credit
FROM student RIGHT OUTER JOIN course ON student.id = course.student_id;
```

{% include image.html url='/assets/image/full_outer_join.png' description='fig 6. Full Outer Join Result'%}


## CROSS JOIN (교차 조인)

CROSS JOIN 은 두 테이블의 모든 행의 가능한 조합[(곱집합)](https://ko.wikipedia.org/wiki/%EA%B3%B1%EC%A7%91%ED%95%A9)을 모두 조회합니다. CROSS JOIN 은 다음과 같이 사용할 수 있습니다.

```sql
SELECT student.id, student.name, course.id, course.name, course.credit
FROM student CROSS JOIN course;
```

{% include image.html url='/assets/image/cross_join.png' description='fig 7. Cross Join Result'%}

CROSS JOIN 이 INNER, OUTER 조인과 다른 점은 조인 컬럼을 반드시 사용하지 않아도 된다는 것입니다. 하지만 조인 컬럼이나 별도의 조건을 사용하지 않는다면 두 테이블의 모든 행의 가능한 조합을 조회하므로 테이블의 행의 수가 많다면 매우 많은 데이터를 조회하게 됩니다.

다음과 같이 CROSS JOIN 에 조인 컬럼을 지정한다면 INNER JOIN 과 동일한 결과를 얻을 수 있습니다.

```sql
SELECT student.id, student.name, course.id, course.name, course.credit
FROM student CROSS JOIN course ON student.id = course.student_id;
```

{% include image.html url='/assets/image/inner_join.png' description='fig 8. Corss Join with Join Column Result'%}