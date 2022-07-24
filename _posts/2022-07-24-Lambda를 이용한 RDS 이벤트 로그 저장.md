---
toc: true
title: Lambda를 이용한 RDS 백업 로그 저장
layout: post
comments: true
author: Martin.S
date: '2022-07-24 18:00:00'
published: true
categories: [Aurora, MySQL, RDS, Lambda, Slack]
---

### RDS 백업 로그 저장
Aurora RDS의 경우, 백업 로그가 \[RDS\]-\[Event\] 페이지에 기록됩니다. 해당 페이지에 기록되는 로그는 약 1~3일 이후 없어지는 **휘발성 로그**입니다. 
필요에 따라, 특정 기간의 백업 로그를 저장하기 위해 lambda와 slack을 이용하여 로그 저장 프로세스를 구축해보게 됐습니다.

기본적인 라이브러리 설치나, 권한 관련 내용은 해당 글에 생략됐습니다.

특정 슬랙 채널에 메세지를 보내기 위해선, slack에서 재공하는 api 앱 중 하나인 `incoming-webhook`을 사용합니다.
해당 앱 설치는 slack에서 제공하는 [설치 가이드](https://slack.com/intl/ko-kr/help/articles/115005265063-Slack%EC%9A%A9-%EC%88%98%EC%8B%A0-%EC%9B%B9%ED%9B%84%ED%81%AC)를 참고했습니다.

### 기본 Lambda 생성
백업 이벤트 로그가 발생 했을 때, slack 채널에 해당 로그 전달을 위한 Lambda를 생성해줍니다.
Lambda란, 별도의 서버 구성 없이 특정 상황(=트리거)에서 코드를 실행하여 필요한 프로세스를 구축할 수 있는 AWS 서비스입니다.
자세한 내용은 [AWS 공식 페이지](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/welcome.html)를 참고해주시면 됩니다.

Lambda에서 지원하는 언어 중 하나인 python을 이용해 이벤트를 받아 slack에 전달하는 스크립트를 작성했습니다.

```python
import os
import json
import urllib.request
import boto3

from dateutil import tz
from datetime import datetime

def post_slack(argStr):                            # 특정 slack 채널에 포스팅하는 함수 선언
    message = argStr                      
    send_data = {
        "text": message,
    }
    send_text = json.dumps(send_data)              # 채널에 보낼 메시지 json형식으로 전환
    request = urllib.request.Request(
        "slack Hook 주소",                          # incoming-webhook에서 제공한 url
        data=send_text.encode('utf-8')             # 한글 관련 encoding 
    )

    with urllib.request.urlopen(request) as response:
        slack_message = response.read()

def save_backup_log(event, context):                # 이벤트를 전달받아 백업 로그를 slack 채널에 전달할 메인 함수 선언
    
    event_snapshot_nm = event['detail']['SourceIdentifier'] # 스냅샷명
    evnet_source_type = event['detail']['SourceType']       # 이벤트 타입
    event_message = event['detail']['Message']              # 이벤트 로그
    event_arn = event['detail']['SourceArn']                # 이벤트 arn
    event_account_id = event['account']                     # 이벤트 계정 ID
    
    

        
    # 로그시간 출력
    seoul_zone = tz.gettz('Asia/Seoul')
    utc_zone = tz.gettz('US/Western')
    
    utc = datetime.now()
    kst = utc.replace(tzinfo=utc_zone).astimezone(seoul_zone)
    kst_time = kst.strftime("%Y-%m-%d %H:%M:%S")
    
    print(event_message)                                    # 이벤트 메세지 확인용 print
    
    if 'created' in event_message:                          # 백업 성공 로그일 경우에만, slack 채널에 전달

        split_nm = event_snapshot_nm.split(':')
        split_nm2 = split_nm[1]
        cluster_nm = split_nm2[0:-17]                       # 스냅샷명에서, cluster명 필터링
        
    
        
        print_text = kst_time + ', ' + cluster_nm + ', ' + event_message + ', ' + event_snapshot_nm   # 출력할 메세지 포맷 결정
        post_slack(print_text)
    
    else:
        print ("This is Not Target Event")                  # 그외 이벤트는 람다 로그로만 남도록 설정.
```

### AWS 이벤트 브릿지 생성
AWS RDS의 백업 이벤트 로그가 발생하는 시점에, 기록할 수 있도록 트리거인 이벤트 브릿지를 생성했습니다.

* 이벤트 브릿지명, Description, 기본 동작 정의
![]({{ site.baseurl }}/images/slack_alarm_lambda/event-brige-1.png "이벤트브릿지명, Description 설정")

* 수집할 이벤트 패턴 정의
![]({{ site.baseurl }}/images/slack_alarm_lambda/event-brige-2.png "수집 이벤트 패턴 정의(RDS backup Event)")

* 설정한 Lambda에 해당 이벤트를 전달
![]({{ site.baseurl }}/images/slack_alarm_lambda/event-brige-3.png "이벤트를 전달한 Lambda 설정")

사용자가 필요한 태그를 붙이는 과정은 생략했습니다. 위 과정을 거쳐, 이벤트 브릿지를 생성했다면 필요한 구성요소는 모두 준비됐습니다.


### 결론
위와 같은 간단한 파이썬 스크립트를 통해 특정 slack 채널에 이벤트 로그를 남기는 프로세스 구축이 완료됐습니다.
간단하지만, 필요할 때 유용하게 사용하면 좋은 기능이라고 생각됩니다.