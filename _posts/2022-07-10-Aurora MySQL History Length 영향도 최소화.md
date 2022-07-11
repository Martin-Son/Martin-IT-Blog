---
toc: true
title: Aurora MySQL History Length 상승을 예방하는 방법
layout: post
comments: true
author: Martin.S
date: '2022-07-10 18:00:00'
published: true
categories: [Aurora, MySQL]
---

### History Length란?
Aurora MySQL은 On-premise MySQL 환경과 동일하게 **REPEATABLE READ** isolation level이 Default로 설정되어 있습니다.
REPEATABLE READ 환경은 쿼리 실행 시점의 결과와, 실행 완료 시점의 결과가 동일해야 하기 때문에, 스냅샷 형태로 해당 시점의 결과를 저장하게 됩니다.
동시성이 높은 환경에서, 롱쿼리가 발생하게 되어 오랜 시간 시점데이터가 유지될수록, **History Legnth** 수치가 상승하게 됩니다.

History Length가 높아질수록, 오래된 데이터 스냅샷으로 인해 다른 조회 쿼리들도 시점 데이터를 통해 걸러내는 과정이 추가되면서,
**검색 비효율**이 발생하고 지연이 발생할 수 있습니다. 극단적으로 트래픽이 높은 서비스 환경에서는 약간의 지연으로 고객들의 불편함이 커지고, 
서비스 장애의 원인이 될 수 있기 때문에, 모니터링해야 하는 필수적인 지표 중 하나입니다.

### 옵션을 통한 History Length 상승 관리
위에서 설명드린 바와 같이, History Length가 서비스에 영향을 줄 수 있는 중요한 요소 중 하나이기 때문에,
기존 on-premise MySQL에서는 이를 관리하기 위해 아래의 옵션을 통해 긴 조회쿼리를 실행해야 하는 상황에서 session isolation level을 통해 관리가 가능했습니다.

```sql
-- Isolation level 변경 (REPATABLE READ -> READ COMMITTED)
mysql> set session transaction isolation level read committed;
```

하지만, Aurora MySQL의 공유 스토리지 환경은 위 옵션만으로는 History Length 관리가 되지 않는 문제점이 있었습니다.
공식 문서인 [Aurora MySQL 격리수준](https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.html#AuroraMySQL.Reference.IsolationLevels)을 통해 기존처럼 관리가 되지 않는 문제를 아래와 같이 표현하고 있습니다.

> **리더 인스턴스의 REPEATABLE READ 격리 수준**
> * 기본적으로 읽기 전용 Aurora 복제본으로 구성된 Aurora MySQL DB 인스턴스에서는 항상 REPEATABLE READ 격리 수준을 사용합니다. 이 DB 인스턴스에서는 SET TRANSACTION ISOLATION LEVEL 문은 모두 무시하고 계속해서 REPEATABLE READ 격리 수준을 사용합니다.
> * 이 설정을 사용하기 전에 READ COMMITTED 격리의 특정 Aurora MySQL 동작을 이해하는 것이 좋습니다. Aurora 복제본 READ COMMITTED 동작은 ANSI SQL 표준을 준수합니다. 그러나 격리는 사용자에게 익숙한 일반적인 MySQL READ COMMITTED 동작보다 덜 엄격합니다.

Aurora MySQL의 공유스토리지 환경에서도, History Length 관리를 돕기 위해 `aurora_read_replica_read_committed` 옵션이 제공됩니다.
```sql
-- Isolation level 변경 (REPATABLE READ -> READ COMMITTED)
-- SET 명령문을 이용한 세션 옵션 적용
set session aurora_read_replica_read_committed = ON;
set session transaction isolation level read committed;
```
해당 옵션은 Aurora MySQL 엔진 버전 **1.21.x 이상 / 2.07.x 이상** 버전부터 적용이 가능하며, 
위 두개 구문 모두를 적용해야 Reader 장비 조회 쿼리로 인한 History Length 상승을 방지할 수 있습니다.


## 옵션 성능 테스트
### 테스트 환경
해당 옵션이 History Length 관리에 효과가 있는지 아래 환경에서 테스트를 진행했습니다.

* 테스트 장비 스펙
   * RDS 버전: Aurora MySQL 2.09.2
   * RDS 인스턴스 타입: r6g.large
   * sysbench(EC2) 타입: c5.large
  

* 테스트 테이블 구성
```bash
# sysbench를 이용하여 1억 건 테스트 테이블 구성
 /usr/local/sysbench/bin/sysbench \
        /usr/local/sysbench/share/sysbench/oltp_read_write.lua \
        --mysql-host='호스트명' \
        --mysql-port=6025 \
        --mysql-user=master --mysql-password='[비밀번호]' \
        --mysql-db=test --db-driver=mysql --tables=1 \
        --table-size=100000000 --threads=128 prepare
```

* 테이블 스키마
```sql
-- 테스트 테이블 스키마 확인 (1억 건 테이블 / 59GB) 
mysql> show create table sbtest1;
 
| sbtest1 | CREATE TABLE `sbtest1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `k` int(11) NOT NULL DEFAULT '0',
  `c` char(120) NOT NULL DEFAULT '',
  `pad` char(60) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  KEY `ix_test` (`pad`,`c`),
  KEY `ix_test2` (`c`,`k`)
) ENGINE=InnoDB AUTO_INCREMENT=100048996 DEFAULT CHARSET=utf8mb4 |
```

### 테스트 시나리오
1. sysbench 툴을 이용한 트래픽 환경 조성
```bash
# 32개 스레드
/usr/local/sysbench/bin/sysbench \
        /usr/local/sysbench/share/sysbench/oltp_read_write.lua \
        --mysql-host='[호스트]' \
        --mysql-port=6025 \
        --mysql-user=master --mysql-password='[비밀번호]' \
        --mysql-db=test --db-driver=mysql \
        --delete_inserts=30 --index_updates=10 \
        --non_index_updates=30 --threads=32 --time=300 --forced-shutdown --report-interval=1 run
```
2. 롱쿼리를 통한 롱 트랜잭션 환경 구성 (풀테이블 스캔)
```sql
mysql> explain select A.* FROM test.sbtest1 as A CROSS JOIN test.sbtest1 AS B WHERE B.k < 50000 ORDER BY A.k DESC LIMIT 20000 ;
+----+-------------+-------+------------+-------+---------------+----------+---------+------+----------+----------+-----------------------------------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows     | filtered | Extra                                                           |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+----------+----------+-----------------------------------------------------------------+
|  1 | SIMPLE      | A     | NULL       | ALL   | NULL          | NULL     | NULL    | NULL | 98631208 |   100.00 | Using temporary; Using filesort                                 |
|  1 | SIMPLE      | B     | NULL       | index | NULL          | ix_test2 | 484     | NULL | 98631208 |    33.33 | Using where; Using index; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+----------+----------+-----------------------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)
``` 

3. 옵션 적용 여부에 따른, History Length 상승 비교
   * **옵션 미적용 시나리오**
   ```sql
   mysql> select A.* FROM test.sbtest1 as A CROSS JOIN test.sbtest1 AS B WHERE B.k < 50000 ORDER BY A.k DESC LIMIT 20000;
   ```
   * **옵션 적용 시나리오**
   ```sql
   mysql> set session aurora_read_replica_read_committed = ON;
   Query OK, 0 rows affected (0.01 sec)
   
   mysql> set session transaction isolation level read committed;
   Query OK, 0 rows affected (0.02 sec)
   
   mysql> select A.* FROM test.sbtest1 as A CROSS JOIN test.sbtest1 AS B WHERE B.k < 50000 ORDER BY A.k DESC LIMIT 20000;
   ```

![]({{ site.baseurl }}/images/History_Length_cloudwatch.png "옵션 적용 여부에 따른 History Length 상승 비교")


### 결론
* 운영환경에서, 위 옵션 적용을 통해 Reader 장비에서 조회쿼리로 인한 History Length 상승을 예방할 수 있다는 걸 확인할 수 있었습니다.
History Length로 인해 크리티컬한 서비스 환경이라면, 조회 쿼리에 대해 해당 옵션 적용이 유용할 것으로 보입니다.