---
toc: true
title: Aurora MySQL Major 버전 업그레이드
layout: post
comments: true
author: Martin.S
date: '2022-08-13 18:00:00'
published: true
categories: [Aurora, MySQL, RDS, Upgrade]
---

## AWS Major 버전 업그레이드
Aurora Major 버전을 업그레이드는 두 가지 방법(콘솔, cli)을 통해 진행할 수 있습니다.
이번 Post에서는 cli 명령어를 통해 **업그레이드하는 방법**과, 운영 상 **순단시간 체크**를 하는 방법에 대해 정리해보려 합니다.

업그레이드 시, 기존 파라미터 그룹과 동일하게 파라미터 그룹 설정이 필요하지만, 아래 테스트 과정에서는 설명상 편의를 위해 생략했습니다.

### 테스트 구성
* 업그레이드 버전: `5.6.mysql_aurora.1.19.6` -> `5.7.mysql_aurora.2.07.7`
* 인스턴스 타입: r5.xlarge
* 구성: Writer 1대 / Reader 1대

### 업그레이드 순서
1. 신규 메이저 버전을 위한 파라미터 그룹 생성
    ```bash
    ### 클러스터 파라미터 그룹 생성
    aws rds create-db-cluster-parameter-group \ 
    --db-cluster-parameter-group-name [신규 클러스터 파라미터그룹명] \ # (ex) aurora-5.7-mysql-upgrade-cluster-pg
    --db-parameter-group-family [버전별 파라미터 그룹 family] \ # (ex) aurora-mysql5.7
    --description "5.7 업그레이드 클러스터 파라미터 그룹"

    ### 인스턴스 파라미터 그룹 생성
    aws rds create-db-parameter-group
    --db-parameter-group-name [신규 인스턴스 파라미터그룹명] \ # (ex) aurora-5.7-mysql-upgrade-pg
    --db-parmaeter-group-family [버전별 파라미터 그룹 family] \ # (ex) 5.7 이상 aurora-mysql -> aurora mysql
    --description "5.7 업그레이드 인스턴스 파라미터 그룹"
    ```
2. 버전 업그레이드를 위한 cli 준비
    ```bash
    $ vi upgrade_cli.sh

    aws rds modify-db-cluster \ 
    --db-cluster-identifier [클러스터명] \ 
    --engine-version 5.7.mysql_aurora_2.07.7 \ 
    --allow-major-version-upgrade \ 
    --db-cluster-parameter-group-name [신규 메이저 버전을 위한 클러스터 파라미터그룹] \ 
    --db-instance-parameter-group-name [신규 메이저 버전을 위한 인스턴스 파라미터그룹] \ 
    --apply-immediately
    ```
3. Writer / Reader별 순단 체크
    ```bash
    $ vi check_connection.sh

    log_writer=[경로]/cluster_writer.log
    log_reader=[경로]/cluster_reader.log
    no=0

    # 1초마다 순단 체크 루프문 구성
    while [ $no -le 10 ]
    do

    mysql -u[마스터계정] -p[비밀번호] -P[포트] -h[클러스터 Writer 엔드포인트] \ 
    -e "SELECT 'Cluster_Writer' as role, @@AURORA_SERVER_ID as aws_nm, aurora_version() as aws_version, CONCAT(VARIABLE_VALUE DIV 3600, '시간 ', MOD
    (VARIABLE_VALUE DIV 60, 60), '분 ', MOD(VARIABLE_VALUE, 60),'초') AS Uptime, now() as dt FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Uptime';" >> ${log_writer}"

    mysql -u[마스터계정] -p[비밀번호] -P[포트] -h[클러스터 Reader 엔드포인트] \ 
    -e "SELECT 'Cluster_Reader' as role, @@AURORA_SERVER_ID as aws_nm, aurora_version() as aws_version, CONCAT(VARIABLE_VALUE DIV 3600, '시간 ', MOD(VARIABLE_VALUE DIV 60, 60), '분 ', MOD(VARIABLE_VALUE, 60),'초') AS Uptime, now() as dt FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Uptime';" >> ${log_reader}"
    ```

4. 업그레이드 스크립트 실행 & 백그라운드로 순단 체크 진행
    ```bash
    # 업그레이드 스크립트 실행
    $ sh upgrade_cli.sh

    # 백그라운드에서, 순단체크 스크립트 실행
    $ nohup sh check_connercion.sh &

    # 순단체크 로깅 확인
    $ tail -f cluster_writer.log # session A
    $ tail -f cluster_reader.log # session B
    ```
5. 업그레이드 상태 체크 & 완료 후 파라미터 그룹 적용을 위한 재부팅
    ```bash
    aws rds describe-db-instances --db-instance-identifier [DB 인스턴스명] \ 
    --query '*[].{DBInstanceIdentifier:DBInstanceIdentifier,DBInstanceStatus:DBInstanceStatus} | [0]'

    # (ex) 업그레이드 완료 시
    #       {
    #           "DBInstanceIDentifier": "sgy-test-220811-upgrade-1-instance-1",
    #           "DBInstanceStatus": "available"
    #       }

    # 업그레이드 완료 후, 파라미터 그룹 적용을 위한 재부팅
    aws rds reboot-db-instance --db-instance-identifier [Writer 인스턴스명]
    ```    

### 테스트 결과
cli를 통한 업그레이드 진행 시, 순단 시간은 아래와 같은 결과를 얻을 수 있었습니다.
* 총 작업 시간: 25분 소요
* 순단 횟수 : **총 2회 발생**
  * 업그레이드 도중 순단 발생: 약 11분 40초 소요
  * 파라미터 그룹 적용을 위한 재부팅: 약 8초 소요

결과적으로 약 12분 가량 순단시간이 발생되며, 서비스 영향도가 가장 적과 시간에 메이저 버전 업그레이드를 진행하면 됩니다.
위 순단 시간에도 크리티컬한 서비스 환경이라면, binlog를 이용한 복제를 사용하여 업그레이드를 진행하는 방법도 존재합니다.
해당 방법은 다음 게시글에서 정리해보도록 하겠습니다.

