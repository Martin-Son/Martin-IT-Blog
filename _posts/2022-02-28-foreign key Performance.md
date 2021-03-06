---
toc: true
title: 외래키, 성능에 어떤 영향을 미칠까?
layout: post
comments: true
date: '2022-02-28 18:00:00'
author: Martin.S
categories: [MySQL, foreign key, performance]
published: true
---
## 외래키 (Foreign Key)란?
외래키는 MySQL에서 기본 엔진으로 사용되는 InnoDB 스토리지 엔진에서만 사용 가능한 제약조건입니다.
외래키는 서로 다른 테이블 간 부모-자식 관계를 정의하여, 데이터 변경이 발생하면 정의한 제약조건에 따라
함께 영향을 받는 특징을 가지고 있습니다.
또한, 부모-자식 관계로 정의된 컬럼에 대해 자동으로 인덱스가 생성됩니다.

## 외래키의 특징
데이터베이스에서 DML을 통해 데이터 변경이 발생하면, Lock을 소유하게 됩니다.
일반적으로 **외래키 제약조건이 없는** 테이블이라면, 다른 테이블에 대한 DML에 대해서는 Lock으로 인해 대기하지 않습니다.

하지만, **외래키 제약조건이 있는** 테이블의 경우, 부모-자식 관계로 정의된 컬럼에 대해서 두 테이블 데이터가 일치해야 하기 때문에,
외래키로 정의된 동일 데이터에 대해 DML 작업이 발생하게 되면, Lock으로 인해 대기해야 하는 상황이 발생합니다.
위 상황을 간단한 아래 테스트를 통해 확인해 볼 수 있습니다.

```sql
-- 외래키 제약조건을 가진 테스트 테이블 A, B 생성

mysql> CREATE TABLE parent_fk
(
    id   int AUTO_INCREMENT PRIMARY KEY,
    name varchar(10),
    age  int NOT NULL,
    INDEX ix_name_age (name, age)
);
Query OK, 0 rows affected (0.04 sec)

mysql> CREATE TABLE child_fk
(
    id      int AUTO_INCREMENT PRIMARY KEY,
    p_name  varchar(10),
    product varchar(15),
    p_age   int(11),
    FOREIGN KEY (p_name, p_age) REFERENCES parent_fk (name, age) ON DELETE CASCADE ON UPDATE CASCADE
);
Query OK, 0 rows affected (0.05 sec)


-- 테스트 데이터 Insert
mysql> INSERT parent_fk (name, age) VALUES ('Tom', 11), ('Martin', 29), ('David', 17), ('Grek', 40), ('Lucy', 26);
Query OK, 5 rows affected (0.02 sec)
Records: 5  Duplicates: 0  Warnings: 0

-- 외래키 제약조건 상 데이터 정합성을 유지하기 위해 부모 테이블에 없는 데이터 Insert 시 에러 발생.
mysql> INSERT child_fk (p_name, product, p_age) VALUES ('Tom', 'TV', 13);
ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`blog_test`.`child_fk`, CONSTRAINT `child_fk_ibfk_1` FOREIGN KEY (`p_name`, `p_age`) REFERENCES `parent_fk` (`name`, `age`) ON DELETE CASCADE ON UPDATE CASCADE)

mysql> INSERT child_fk (p_name, product, p_age) VALUES ('Tom', 'TV', 11);
Query OK, 1 row affected (0.03 sec)

-- 데이터 확인
mysql> SELECT * FROM parent_fk;
+----+--------+-----+
| id | name   | age |
+----+--------+-----+
|  3 | David  |  17 |
|  4 | Grek   |  40 |
|  5 | Lucy   |  26 |
|  2 | Martin |  29 |
|  1 | Tom    |  11 |
+----+--------+-----+
5 rows in set (0.01 sec)

mysql> SELECT * FROM child_fk;
+----+--------+---------+-------+
| id | p_name | product | p_age |
+----+--------+---------+-------+
|  2 | Tom    | TV      |    11 |
|  4 | David  | Desk    |    17 |
+----+--------+---------+-------+
2 rows in set (0.01 sec)

```

| 시간 | 세션 1 | 세션 2 | 비고 |
| -------- | -------- | -------- | -------- |
| 1 | ``START TRANSACTION;``   |    |    |
| 2 | ``UPDATE parent_fk SET age=80 WHERE name = 'Tom';``   |    |  name='Tom'인 부모테이블의 age 컬럼 업데이트 update  |
| 3 |   | ``START TRANSACTION;``    |    |
| 4 |   | ``UPDATE child_fk SET product = 'Notebook' WHERE p_name = 'Tom';``    |  자식 테이블의 name='Tom' product 컬럼 update  |
| 5 |   | ``ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction``    |  Lock으로 인해 wait_timeout 만큼 대기 후 에러 로그 발생  |


## 외래키의 성능

위의 테스트 결과에서 확인할 수 있듯이, 외래키 제약 조건이 있으면 Lock을 통해 제약 조건을 만족할 수 있도록 제어합니다.
외래키에 대한 DML이 다량으로 발생하게 되면 그만큼 확인 작업도 증가하게 될테니 성능에 어떤 영향을 주는지 의문이 들었습니다.


그래서 sysbench 툴을 이용해 2개의 부모-자식 관계 테이블을 생성하고 
외래키에 대한 update문을 실행하도록 하여 스레드별 성능 테스트를 진행했습니다.

테스트를 위한 데이터 준비는 아래 스크립트를 이용하여 만들었습니다.

```shell
# sysbench툴을 이용한 테스트 테이블 생성
## 1억건 테이블 2개 / 약 21GB
/usr/local/sysbench/share/sysbench/oltp_read_write.lua \
	--mysql-host='[호스트 정보]' \
	--mysql-port=[Port 번호] \
	--mysql-user=master --mysql-password='[비밀번호]' \
	--mysql-db=sbtest --db-driver=mysql --tables=2 \
	--table-size=100000000 --threads=128 prepare
```

위에서 준비한 테이블에 외래키 제약조건을 테스트할, 컬럼을 추가해주고 외래키 제약조건을 생성합니다.

```sql
-- 부모테이블에 외래키 제약조건 생성을 위한, 컬럼추가 & 인덱스 추가
mysql> ALTER TABLE sbtest1 ADD foreign_column char(100) DEFAULT 'test_for_foreign_key', ADD INDEX ix_foreign_column(foreign_column), ALGORITHM = INPLACE, LOCK=NONE;

-- 자식테이블에 컬럼 추가 & 외래키 제약조건 추가
mysql> ALTER TABLE sbtest2 ADD foreign_column char(100), ADD CONSTRAINT fk_test FOREIGN KEY (foreign_column) REFERENCES sbtest1 (foreign_column) ON UPDATE CASCADE;
```

외래키 성능 테스트를 위한 테이블 스키마 사전 준비는 위 과정을 통해 모두 끝냈습니다.
이후 기본 sysbench 테스트 툴에는 수동으로 생성한 외래키 컬럼에 대한 DML문이 실행되지 않기 때문에, 수정이 필요합니다.

```shell
# update index 옵션 사용 시, 실행되는 update문을 변경 ()
$ cp [경로]/oltp_common.lua [경로]/oltp_common_backup.txt

$ vi [경로]/oltp_common.lua

# 아래 구문 수정 
#index_updates = {
#      "UPDATE sbtest%u SET k=k+1 WHERE id=?",
#      t.INT},
index_updates = {
      "UPDATE sbtest1 SET foreign_column=(case floor(rand()*5) when 0 then 'abc' when 1 then 'czcz' when 2 then 'sotq' end) WHERE id=?",
      t.INT},
```

위와 같이 수정 후, 아래 스크립트문을 준비하여 동시 실행 스레드 수별 성능 차이를 테스트해봅니다.
테스트는 아래와 같은 MySQL 환경에서 진행했습니다.
- **버전**: 5.7.mysql_aurora.2.09.2
- **인스턴스 스펙**: db.r6g.xlarge

```shell
# $ vi test.sh
/usr/local/sysbench/bin/sysbench \
/usr/local/sysbench/share/sysbench/oltp_read_write.lua \
        --mysql-host='[접속 호스트명]'
        --mysql-port=[Port번호] \
        --mysql-user=master --mysql-password='[비밀번호]' \
        --index_updates=50 \
        --mysql-db=sbtest_foreign --db-driver=mysql --threads=[병렬 실행 Thread수]  --forced-shutdown \
        --tables=2
        --report-interval=1 --max-time=120 run
```

외래키 제약조건이 있는 상태에서 스레드별 결과를 확인 후, 제약조건을 제거합니다.

```sql
-- 외래키 제약조건 제거
mysql> ALTER TABLE sbtest2 DROP FOREIGN KEY fk_test;
Query OK, 0 rows affected (0.05 sec)
-- sysbench 스크립트 병렬 스레드 갯수를 조절해가며 이전 결과와 비교 진행.
```


sysbench를 이용하여, 외래키 관계에 있는 컬럼에 대한 update를 실행하도록 조절했습니다.
그에 대한 성능 결과는 아래와 같습니다. 

![]({{ site.baseurl }}/images/foreign_key_perf.png "외래키 유무에 따른 성능 차이 비교 ")

병렬 스레드 갯수와 무관하게, 전체적으로 외래키 제약조건이 존재할 경우, QPS는 감소하고 Latency는 증가하는 현상을 확인할 수 있습니다.
DBA로서 외래키 제약조건을 여러 이유로 최대한 지양 (성능 저하, DDL/DML시 관리포인트 증가)해야 하는 사실은 인지만 했었는데,
간단한 테스트를 통해 성능 저하를 확인할 수 있어 좋은 경험이었습니다.



