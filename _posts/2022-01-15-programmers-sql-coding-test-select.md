---
layout: post
title: 프로그래머스 SQL 코딩 테스트 문제풀이 (1)
date: 2022-01-15
excerpt: "programmers sql coding test 1"
tags: [SQL]
comment: true
---


# 개요
- 수학이 너무 지겨워서 즉흥적으로 풀어본 프로그래머스 SQL 코딩 테스트 정답을 기록해두는 페이지
- Oracle을 기준으로 작성하였으며, 동물 보호소의 데이터를 관리하고 조회하는 쿼리를 작성하도록 구성되어 있다.


## 1. SELECT


```SQL
# 모든 레코드 조회하기
SELECT * from ANIMAL_INS order by ANIMAL_ID;


# 역순으로 조회하기
SELECT NAME, DATETIME from ANIMAL_INS order by ANIMAL_ID desc;


# 아픈 동물 조회하기
SELECT ANIMAL_ID, NAME FROM ANIMAL_INS WHERE INTAKE_CONDITION = 'Sick' ORDER BY ANIMAL_ID;


# 어린 동물 조회하기
SELECT ANIMAL_ID, NAME FROM ANIMAL_INS WHERE INTAKE_CONDITION != 'Aged' ORDER BY ANIMAL_ID;


# 동물 아이디와 이름 조회하기
SELECT ANIMAL_ID, NAME FROM ANIMAL_INS ORDER BY ANIMAL_ID;


# 여러 기준으로 정렬하기(이름을 먼저 조회하고, 이름이 같으면 보호를 나중에 시작한 동물 순으로 조회)
SELECT ANIMAL_ID, NAME, DATETIME FROM ANIMAL_INS ORDER BY NAME ASC, DATETIME DESC;


# 상위 N개 조회하기(Inine View와 Rownum을 이용했으나, 더 깔끔하고 효율적인 방법이 있을 것 같다.)
# PostgreSQL에도 비슷한 기능을 Limit이라는 키워드로 제공하지만, Limit은 정렬 연산이 완료된 다음에 출력되는 레코드 개수를 제한한다.
# 하지만 Oracle에서는 정렬 연산 이전의 레코드 순서를 기준으로 출력을 제한해준다.
SELECT NAME FROM (SELECT * FROM ANIMAL_INS ORDER BY DATETIME) WHERE ROWNUM < 2;
```


## 2. SUM, MIN, MAX


```SQL
# 가장 늦게 들어온 동물 조회하기
SELECT MAX(DATETIME) FROM ANIMAL_INS;


# 가장 일찍 들어온 동물 조회하기
SELECT MIN(DATETIME) FROM ANIMAL_INS;


# 동물이 총 몇 마리인지 조회하기
SELECT COUNT(DISTINCT(ANIMAL_ID)) FROM ANIMAL_INS;


# 중복 제거하기
SELECT COUNT(DISTINCT(NAME)) FROM ANIMAL_INS WHERE NAME IS NOT NULL;
```


## 3. Group By


- Group By는 Pandas Dataframe을 조작할 때도 많이 사용되는 개념이어서 생각보다는 수월하게 풀 수 있었다.
- Pandas에서 Group By만 수행하면 Group By 객체가 리턴되고 집계 연산을 실행해야 원하는 결과가 나오는데, 이는 Oracle에서도 마찬가지
- Group By 기준 컬럼을 이용해서 집계 연산을 수행해줘야 쿼리가 정상적으로 동작한다. 


```SQL
# 고양이와 개는 몇 마리인가요
SELECT ANIMAL_TYPE, COUNT(ANIMAL_TYPE) AS COUNT FROM ANIMAL_INS WHERE ANIMAL_TYPE IN ('Cat', 'Dog') GROUP BY ANIMAL_TYPE ORDER BY ANIMAL_TYPE ASC;


# 동물 보호소에 들어온 동물 이름 중 두 번 이상 쓰인 이름과 해당 이름이 쓰인 횟수를 조회하기
# Having 절은 Group By로 나눠진 각 그룹에 적용할 연산이 있으면 사용. 이 경우는 각 그룹(동물)들 중 2마리 이상 있는 동물을 찾기에 필요함
# FROM, WHERE, GROUP BY, HAVING, SELECT, ORDER BY 순서로 쿼리가 동작한다.
SELECT NAME, COUNT(NAME) AS COUNT FROM ANIMAL_INS WHERE NAME IS NOT NULL GROUP BY NAME HAVING COUNT(NAME) >= 2 ORDER BY NAME ASC;


# 각 시간대별로 입양이 몇 건이나 발생했는지 조회하기
# Inline View, 시간 조작 및 형 변환 등 여러 아이디어가 숨어 있는 문제
# Oracle에서 시간을 조작할 때는 Extract 함수를 써야 하며, Extract 시간던위 From 컬럼 형태로 사용한다.
# 이 때, 분(Minute) 및 시간(Hour)를 조회할 때는 시간과 분을 저장하고 있는 TIMESTAMP 자료형에서 추출해야 한다. 만약, 일 정보를 조회하려면 Date자료형에서 조회해야 한다.
# 만약 자료형이 맞지 않는다면, 형 변환(CAST column AS 자료형) 형태로 변환해줘야 한다.
SELECT HOUR, COUNT(HOUR) AS COUNT FROM (SELECT ANIMAL_ID, EXTRACT(HOUR FROM CAST(DATETIME AS TIMESTAMP)) AS HOUR FROM ANIMAL_OUTS) GROUP BY HOUR HAVING HOUR BETWEEN 9 AND 19 ORDER BY HOUR ASC;


# 각 시간대별로 입양이 몇 건이나 발생했는지 조회하기
# 데이터네는 9시부터 19시까지 데이터만 존재하므로, 그 외 시간대에는 0으로 채워야 한다.
SELECT l.hour, nvl(count, 0) AS count
FROM (SELECT TO_CHAR(datetime, 'HH24') AS hour, count(*) AS count
        FROM animal_outs
        GROUP BY TO_CHAR(datetime, 'HH24')
        ORDER BY hour) O,
        (SELECT LEVEL-1 AS hour FROM dual CONNECT BY LEVEL<=24) L
WHERE L.hour = O.hour(+)

ORDER BY L.hour;
```



