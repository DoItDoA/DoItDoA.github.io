---
title: "DB 파티셔닝 개념과 범위분할"
layout: single
categories: SQL
tag: [SQL]
toc: true
toc_sticky: true
author_profile: false
sidebar:
    nav: "docs"
---
# 파티셔닝이란
테이블을 더 작은 테이블이나 DB 내부 분할로 보면 된다.  
2가지로 나뉘는데  
**수직 분할(Vertical Partitioning)**과 **수평 분할(Horizontal Partitioning)**로 나뉘어진다.  
**수직 분할**은 대표적인 걸로 '정규화'가 있다.  
여기서는 **수평 분할**에 대해서 설명해보겠다.  
<br>
수평 분할은 row수가 매우 많아질 경우 index의 범위도 매우 늘어나고  
조회나 삽입 등에서도 부담이 된다.  
이러한 방법을 해결하기 위한 것이 수평 분할이다.  
수평 분할에서도 '해시 분할', '범위 분할'등 여러개가 있다.  

# 범위 분할
날짜, 시간 기준으로 나누는 것이 효과적이다.  
대표적으로 매 초마다 들어오는 데이터나 로그 사용에 쓰인다.  

<br>
예시 코드로 확인해보자.(MySQL)
```sql
CREATE TABLE table_name (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `create_date` datetime(6) NOT NULL, # 2
  `create_id` varchar(20) DEFAULT NULL,
  `content` varchar(255) NULL,
  PRIMARY KEY (`id`, `create_date`) # 1
)
PARTITION BY RANGE (TO_DAYS(create_date)) (
    PARTITION p0 VALUES LESS THAN (TO_DAYS('2025-01-01')), # 3
    PARTITION p1 VALUES LESS THAN (TO_DAYS('2025-02-01')) # 4
);
```
테이블 생성시 파티셔닝 생성과 함께 만들어야한다.  

### 참고
MySQL은 테이블만 만들고나서 나중에 파티셔닝을 따로 추가할 수 없다.  
테이블을 만들 때 함께 파티셔닝 추가를 해야한다.  
Oracle이나 PostgreSQL은 테이블만 만들어 놓고 나중에 따로 파티셔닝을 추가할 수 있다.  

파티셔닝을 할 때 파티션 키를 지정해야하는데 create_date를 파티션 키로 지정  
1. 파티션 키를 지정할 때는 해당 키가 UNIQUE 또는 PRIMARY키 이어야한다.
2. 복합키로 설정됐기 때문에 NULL이어서는 안된다.
3. p1은 임의로 이름을 지정하는 것이고, '2025-01-01' 이전 날짜까지 범위 지정하는 것이다.
4. '2025-01-01'부터 '2025-02-01' 이전까지 범위를 지정하는 것이다.

* 만약 파티션을 추가하고 싶으면
```sql
ALTER TABLE table_name ADD PARTITION (PARTITION p2 VALUES LESS THAN (TO_DAYS('2025-03-01')));
```
이렇게 추가하면 된다.  
* 파티션 날짜가 '2025-02-01' 이전까지 설정되어 있으면 데이터 삽입시 '2025-02-01'부터 이후의 데이터는 삽입할 수 없다. 파티션을 더 추가해야한다.
* 만일 파티션을 더 추가하지 않고 끝내고 싶으면 'PARTITION p_max VALUES LESS THAN MAXVALUE' 추가하여 최대치 파티션을 생성한다.
* 파티션은 최대 1024개까지 추가가 된다.

```sql
'SELECT *  
FROM table_name  
WHERE create_date BETWEEN '2025-01-07 00:00:00' and '2025-01-08 00:00:00';
```
데이터를 찾을 시 'PARTITION p1 VALUES LESS THAN (TO_DAYS('2025-02-01'))' 의 범위 안에 들어므로 해당 파티션 기준으로 데이터를 빨리 찾게 된다.  
