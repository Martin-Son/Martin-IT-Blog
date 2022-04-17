---
toc: true
title: Aurora MySQL Quorum 모델
layout: post
comments: true
author: Martin.S
date: '2022-04-01 18:00:00'
published: true
categories: [Aurora, Stoarge, Quorum, AWS]
---

## Aurora Storage 모델
AWS에서 분산 시스템을 구성하는 쿼럼(=Quorum)에 대해 정리해보려 합니다.

![]({{ site.baseurl }}/images/aurora_storage_architecture.jpeg "Aurora Storage 아키텍처")

Aurora Storage는 위 그림과 같이 총 6개의 복제본으로 구성되어 있으며 특징은 아래와 같습니다.
1. 3개의 Zone에 대해 각각 2개의 스토리지 영역으로 구성
2. 스토리지 데이터 정합성을 보장하기 위해 쿼럼 모델을 사용
3. 쓰기 작업의 완료 최소 노드 수: 6개중 **4개** / 읽기 작업 완료 최소 노드 수: 6개중 **3개**

### 쿼럼(=Quorum) 모델
쿼럼 모델은 여러 노드가 존재하는 클러스터 내부에서 데이터의 정합성을 보장하면서,
특정 노드에 장애가 발생했을 경우에도 문제없는 서비스 환경을 제공하기 위한 모델입니다.

데이터 정합성에 대한 보장과 동시에 **특정 작업의 완료 여부를 판단**을 위해 제약조건이 존재합니다.
특정 작업의 완료를 판단하기 위한 최소 노드 수는 아래 표와 같습니다.

| 총 노드 수 (Nt) | 최소 Read 노드 수 (=Nr) | 최소 Write 노드 수 (=Nw) |
| -------- | -------- | -------- |
| 1 | 1 | 1 |
| 2 | 1 | 2 | 
| 3 | 2 | 2 |
| 4 | 2 | 3 |
| 5 | 3 | 3 |
| **6** | **3** | **4** |
| 7 | 4 | 4 |

위의 최소 노드 수 기준은 아래 쿼럼모델의 제약조건을 만족하는 완료조건입니다.
1. Nt < Nr + Nw (총 노드 수 < 최소 읽기 노드 수 + 최소 쓰기 노드 수)
2. Nw > Nt / 2  (최소 쓰기 노드 수 > 총 노드 수의 절반)

이론상, 현재 Aurora Storage 모델을 구성하는 6개 쿼럼모델에서 
**읽기 작업**은 3개 노드의 장애까지 정상적인 서비스가 가능하며 **쓰기 작업**은 2개 노드 장애에 대비가 되어 있습니다.

또한 Zone을 세개로 분산시켜, 하나의 지역(=Zone)에 정전이나 재해가 발생하게 되더라도
남은 2개 지역(=4개 노드)을 통해 정상적인 서비스를 할 수 있도록 구성되어 있습니다.


### 쿼럼모델의 단점 개선을 위한 Aurora 아키텍처
쿼럼 구조로 정합성과 가용성 두가지 모두를 챙길 수 있지만, 이러한 구조적 특성으로
버퍼풀에 없는 데이터에 대한 조회 요청을 받게되면 쿼럼 모델의 제약조건에 따라
최소 3개 이상의 노드에서 완료(=ACK) 신호를 받아야 하므로, **Read Latency가 발생**하게 됩니다.

![]({{ site.baseurl }}/images/Aurora_Quorum_Read.png "Aurora 쿼럼 Read Latency 최소화")

이런 Read Latency를 최소화 하기 위해 Aurora에서는 LSN을 이용해 노드별 우선순위를 결정합니다.
예를 들어 특정 쓰기가 발생하면 최소 4개의 노드에서 쓰기 완료(=ACK) 응답을 받으면 커밋으로 인지하게 되는데요.
해당 데이터에 대한 읽기 요청이 들어왔을 때 먼저 완료된 4개 노드에 대해서 우선적으로 읽기 요청을 보내는데, 
이를 판별하는 수단이 **노드별 LSN**를 통해 가장 최신 상태인 노드를 판단합니다.
이를 통해 데이터가 있는 노드에 **우선적으로 접근**하여 불필요한 접근을 줄여 Read Latency를 최소화합니다.






#### sysbench를 이용한 File I/O 성능 비교

성능 비교 환경 구성을 위해 두 가지 EBS(HDD, SSD)를 추가 후 마운트합니다.

```shell
## 마운트할 디스크 확인.
$ lsblk -f
NAME    FSTYPE LABEL UUID                                 MOUNTPOINT
xvda
└─xvda1 xfs    /     518d317e-9b1a-43aa-8b7c-850dd3510341 /
xvdf    xfs          7ab58089-47e5-4667-b68c-45a6f3d2262f
xvdg    xfs          c49943b6-68ba-47d5-ad57-ba56a6852ef2
xvdh    xfs          60a2fae0-05d5-4690-96d4-10b9f8784543
 
 
## 디스크 Mount
$ sudo mount dev/xvdg /tmp_hdd
$ sudo mount dev/xvdh /tmp_ssd_gp3
 
## Mount 확인
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        2.0G     0  2.0G   0% /dev
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           2.0G  428K  2.0G   1% /run
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/xvda1       32G   17G   16G  51% /
tmpfs           395M     0  395M   0% /run/user/1001
/dev/xvdg       300G  339M  300G   1% /tmp_hdd
/dev/xvdh       300G  339M  300G   1% /tmp_ssd_gp3
```

이후 디스크별 File I/O 테스트를 진행하기 위해, sysbench 파일을 복사합니다.

```shell
## sysbench 파일 복사
$ cp -R /usr/share/sysbench /tmp_ssd_gp3/
$ cp -R /usr/share/sysbench /tmp_hdd/

# 각 디스크에서 File I/O 테스트 환경 구축
$ sysbench --test=fileio --file-total-size=8G prepare
 
-rw------- 1 root root 67108864 Aug  3 05:07 test_file.57
-rw------- 1 root root 67108864 Aug  3 05:07 test_file.58
-rw------- 1 root root 67108864 Aug  3 05:07 test_file.59
-rw------- 1 root root 67108864 Aug  2 09:25 test_file.6
-rw------- 1 root root 67108864 Aug  3 05:07 test_file.60

# 각 디스크에서 file I/O 테스트 실행
$ sysbench fileio --file-total-size=8G --file-test-mode=rndrw  --max-time=30 --report-interval=1
```

file-test-mode=rndrw 옵션을 사용해, Random I/O (읽기, 쓰기) 성능 차이를 비교한 결과는 아래와 같습니다.
![]({{ site.baseurl }}/images/EBS_Perf_vs.png "EBS Read Write 성능 비교")


#### 디스크 타입별 mysql 임시테이블 Insert 성능 비교

위 테스트는 sysbench 툴을 이용하여 간접적으로 성능 차이를 확인했습니다. 실제 mysql을 사용하면서 디스크 타입에 따라 성능 차이가
어떻게 나는지 확인하기 위해 임시테이블에 단건 Insert 1000만 건 하는 SP를 생성하여,
완료 시간 기준으로 성능을 비교해봤습니다.

```sql
-- 소요시간 기록 테이블 생성
 
CREATE TABLE `time_table` (
  `duration_sec` int(11) DEFAULT NULL,
  `cnt` int(11) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
 
-- SP 스크립트
CREATE
    DEFINER = `root`@`%` PROCEDURE `diskio_test`()
BEGIN
    DECLARE cnt int DEFAULT 1;
    DECLARE begin_time datetime;
    DECLARE end_time datetime;
 
    -- MyISAM 임시테이블 (디스크 directory 사용) 생성
    -- Engine=MyISAM 명시하지 않을 시 InnoDB 엔진을 사용하며, ibtmp1 (데이터파일 위치)에 생성되어 디스크 성능 테스트 실패
 
    CREATE TEMPORARY TABLE disk_tmp
    (
        id       int AUTO_INCREMENT PRIMARY KEY,
        calendar date,
        num      int,
        num_2    int,
        Index idx_calendar (Calendar),
        index idx_num (num)
    ) ENGINE = MyISAM;
 
    SELECT NOW() INTO begin_time;
       WHILE (cnt <= 10000000)  -- 단건 Insert 1000만 건.
        DO
               INSERT disk_tmp(calendar, num, num_2)
               VALUES((CURRENT_DATE - INTERVAL FLOOR(RAND() * 14) DAY),
                       cast(rand()*100 as unsigned), cast(rand()*100 as unsigned))
                ;
       
            SET cnt = cnt + 1;
 
        end while;
     SELECT NOW() INTO end_time;
 
            INSERT INTO time_table
            SELECT TIMESTAMPDIFF(SECOND, begin_time, end_time), cnt;
    DROP TEMPORARY TABLE disk_tmp;
END;


call diskio_test();
```

위 SP를 만든 후, 실제 운영환경과 같이 병렬로 Insert가 발생하는 환경을 재현하기 위해
백그라운드 프로세스로 위 SP를 10개 세션에서 병렬로 실행했습니다

```shell
$ vi sp_insert.sh
# sp 실행 구문
mysql -umaster -p[비밀번호ㅑi -h[호스트] -P[포트] -e"call test.diskio_test();"
```

```shell
$ vi disk_io.sh
# 병렬 sp 실행
for ((i=0; i<10; i++)); do

nohup sh sp_insert.sh &

sleep 1
done
```

위 스크립트를 실행하여 얻은 결과를 그래프로 비교한 결과는 아래와 같습니다.
![]({{ site.baseurl }}/images/disk_insert_perf.png "디스크 타입별 mysql 임시테이블 성능 비교")


두 가지 테스트를 통해 아래와 같은 사실을 확인할 수 있었습니다.

1. SSD (gp3) vs Cold HDD 8GB 기준 File I/O 성능 비교 시, 약 45배 정도의 성능 차이가 발생.
2. File I/O 차이만큼 디스크 I/O로 인한 Sorting 쿼리 성능 차이가 나진 않았지만,  (1.6배) 병렬 세션 수에 비례해서 성능차이가 증가.
3. **Random I/O 성능면에서 SSD가 HDD보다 훨씬 뛰어남*면