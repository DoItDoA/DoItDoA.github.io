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
예시 코드로 확인해보자.  
```sql
CREATE TABLE table_name (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `create_date` datetime(6) NOT NULL,
  `create_id` varchar(20) DEFAULT NULL,
  `content` varchar(255) NULL,
  PRIMARY KEY (`id`, `create_date`)
)
PARTITION BY RANGE (TO_DAYS(create_date)) (
    PARTITION p1 VALUES LESS THAN (TO_DAYS('2025-01-01')),
    PARTITION p2 VALUES LESS THAN (TO_DAYS('2025-02-01'))
);
```
테이블 생성시 파티셔닝 생성과 함께 만들어야한다.  

### 참고
MySQL은 테이블만 만들고나서 나중에 파티셔닝을 따로 추가할 수 없다.  
테이블을 만들 때 함께 파티셔닝 추가를 해야한다.  

