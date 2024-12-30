---
title: "SQL RECURSIVE 사용법"
layout: single
categories: SQL
tag: [SQL]
toc: true
toc_sticky: true
author_profile: false
sidebar:
    nav: "docs"
---
# 참고
MySQL 기준으로 코드를 작성했습니다.

# RECURSIVE의 특징
재귀를 통한 반복으로 데이터를 반복 삽입 등에 활용이 됨  
<br/>
**기본 양식**
```
WITH RECURSIVE 가상테이블명(컬럼1, 컬럼2, ...) AS
(
  SELECT 컬럼1활용, 컬럼2활용, ...
  UNION ALL # UNION이 사용될 수도 있음
  SELECT 컬럼1활용, 컬럼2활용, ... FROM 가상테이블명 WHERE 컬럼 < 숫자 # 여기서 반복 멈추는 조건
)
SELECT 컬럼A, 컬럼B, 컬럼C, ... FROM 가상테이블명
```
이렇게 구성 되어있다.  
좀 더 이해를 위해 예시를 보자  
```
SET SESSION cte_max_recursion_depth = 1000000; ##  1

INSERT INTO users (name, age) ##  4
WITH RECURSIVE cte (n) AS   ##  2
(
  SELECT 1
  UNION ALL
  SELECT n + 1 FROM cte WHERE n < 1000000
)
SELECT ##  3
    CONCAT('User', LPAD(n, 7, '0')),
    FLOOR(1 + RAND() * 1000) AS age
FROM cte;
```
* 순서는 1 -> 2 -> 3 -> 4 순서대로 진행으로 보면된다.  
* SET SESSION cte_max_recursion_depth은 최대 재귀횟수를 정한다. (기본은 1000)
  * cte_max_recursion_depth는 고정된 글로 정확하게 써야한다.
* WITH RECURSIVE
  1. etc는 가상테이블 이름을 정하고 n은 컬럼명
  2. 'SELECT 1'에서 1은 n을 가리킴 (가상테이블 1행에 값 1 추가)
  3. UNION ALL을 통해 가상테이블에 새로운 행 추가
  4. 'SELECT n + 1'에서 'n=1'이므로 2가되고 n=2로 변경 (가상테이블 2행에 값 2 추가)
  5. 'WHERE n < 1000000' 에서 2 < 1000000 이므로 다시 재귀반복
  6. 가상테이블의 각 행에 데이터 1,2,3,4,5,6,... 쌓임
  7. n=1000000되는 순간 재귀 반복 끝
* SELECT
  1. 가상테이블 etc에 저장된 데이터를 활용하여 구체적으로 데이터를 구성
  2. CONCAT은 'User'와 'LPAD(n, 7, '0')'를 서로 붙임
  3. LPAD(필드, 길이, 채우는 문자)이며 LPAD(n, 7, '0')에서 n=1일 경우,  
길이가 7이면서 1을 표시, 남는 공간은 왼쪽에서부터 0을 채운다.  
즉 '0000001'이 된다.
  5. FLOOR()는 소수점을 버리는 함수
  6. RAND()는 0~1사이에 랜덤으로 수 표시
  7. 두번째 컬럼에서도 n을 활용할 수 있으나 필요에 따라 사용 안함
* INSERT
  * 만들어진 가상 테이블을 실제 테이블에 삽입
