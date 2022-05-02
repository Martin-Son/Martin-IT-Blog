---
toc: true
title: Aurora Quorum 모델
layout: post
comments: true
author: Martin.S
date: '2022-04-30 18:00:00'
published: true
categories: [Aurora, Stoarge, Quorum, AWS]
---

## Aurora Storage 모델
AWS에서 분산 시스템을 구성하는 쿼럼(=Quorum)에 대해 정리해보려 합니다.

![]({{ site.baseurl }}/images/Aurora_Quorum/aurora_storage_architecture.jpeg "Aurora Storage 아키텍처")

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


### 쿼럼모델 Latency 개선을 위한 Aurora 아키텍처
쿼럼 구조로 정합성과 가용성 두가지 모두를 챙길 수 있지만, 이러한 구조적 특성으로
버퍼풀에 없는 데이터에 대한 조회 요청을 받게되면 쿼럼 모델의 제약조건에 따라
최소 3개 이상의 노드에서 완료(=ACK) 신호를 받기 위해 6개 노드를 무작위로 찾게 되면, 
**Read Latency가 발생**하게 됩니다.

![]({{ site.baseurl }}/images/Aurora_Quorum/Aurora_Quorum_Read.png "Aurora 쿼럼 Read Latency 최소화")

이런 Read Latency를 최소화 하기 위해 Aurora에서는 LSN을 이용해 노드별 우선순위를 결정합니다.
예를 들어 특정 쓰기가 발생하면 최소 4개의 노드에서 쓰기 완료(=ACK) 응답을 받으면 커밋으로 인지하게 되는데요.
해당 데이터에 대한 읽기 요청이 들어왔을 때 먼저 완료된 4개 노드에 대해서 우선적으로 읽기 요청을 보내는데, 
**노드별 LSN**의 크기를 이용하여 가장 최신 상태인 노드를 판단합니다.
이를 통해 데이터가 있는 노드에 **우선적으로 접근**하여 불필요한 접근을 줄여 **Read Latency를 최소화**합니다.

Latency를 최소화하더라도 여전히 쿼럼 구조는 Latency가 발생하므로
쿼럼 읽기 작업이 항상 발생하지 않으며 아래와 같은 상황에서만 발생합니다.
1. 메모리에 없는(=캐싱되지 않은) 데이터에 대한 조회 요청
2. Master 인스턴스 재시작
3. Reader 인스턴스가 Master로 승격


### Aurora 쿼럼모델 비용 개선 아키텍처
Aurora 쿼럼모델은 6개의 노드를 사용하며, 만약 모든 노드가 전체 데이터를 소유한다면
일반적인 스토리지 구조에 비해 6배 크기의 데이터 베이스를 소유해야 합니다.
그만큼 비용도 크게 증가(=**약 6배**)하게 되므로 이런 구조를 개선하기 위해 Aurora에서는 
**풀 세그먼트 + 테일 세그먼트** 구조를 이용합니다.

> * 풀 세그먼트(=Full Segment) : 데이터 페이지 + Redo Log
> * 테일 세그먼트(=Tail Segment): Redo Log

Aurora 데이터베이스 볼륨은 **10GB 데이터 세그먼트 단위**로 구성됩니다.
데이터 세그먼트는 6개의 노드로 복제되어 보관되며, 같은 6개의 세그먼트 Set을 보호그룹(=Security Group)이라 합니다.
6개의 세그먼트 Set은 3개의 풀 세그먼트, 3개의 테일 세그먼트로 구성됩니다.

![]({{ site.baseurl }}/images/Aurora_Quorum/aurora_quorum_model.png "Aurora 쿼럼모델 아키텍처")

위 그림과 같이 각 Zone에는 두 개의 노드가 포함되며, Zone별로 풀 세그먼트 1 + 테일 세그먼트 1로 구성되어 있습니다.
이를 통해 기존 예상 증가 금액 대비 50% 절감이 가능해집니다. (6배 -> 약 3배)

일반적인 구조와 다르게 풀 + 테일 세그먼트가 혼합된 Aurora 아키텍처에서는 
쓰기/읽기 작업 완료를 위해 아래의 특정 조건을 만족해야 합니다.

1. 쓰기 작업 (6개 중 4개의 노드 ACK)
   - 1개 이상의 풀 세그먼트 + 테일 세그먼트
2. 읽기 작업 (6개 중 3개 노드 ACK)
   - 1개의 풀 세그먼트 + 테일 세그먼트

또한 조회 Latency를 최소화하기 위한 방법도 LSN을 통해 가장 최신의 **풀 세그먼트를 소유**한 
노드를 이용하여 요청을 처리하게 됩니다. 
만약 세그먼트에 Corrupt가 발생하면, 풀세그먼트와 테일 세그먼트를 이용하여 쉽게 복구도 가능합니다.


### 장애 관리를 위한 Aurora Quorum Membership 개념
Aurora는 쿼럼 집합(=Quorum Membership) 개념을 이용하여 장애 상황을 관리합니다.

세그먼트에 변화가 발생할 때마다, 독립적인 상태값인 **epoch** 라는 값이 증가합니다.
이 값을 통해 쿼럼 집합 구성원들의 상태를 판단합니다.

아래 간단한 예시를 통해 쿼럼 멤버십에 대해 설명드리겠습니다.
![]({{ site.baseurl }}/images/Aurora_Quorum/quorum_membership.jpeg "Aurora 쿼럼 집합")
기존 구성원(=ABCEDF) 중 F에 장애가 일어났을 때, 아래와 같은 과정을 거치게 됩니다.

1. F 구성원에 장애 발생
2. 신규 G 구성원 준비 / F 구성원 복구 프로세스 진행
3. 기존 구성원 집합(=ABCDE**F**), 신규 구성원 집합(=ABCDE**G**)에 요청(읽기/쓰기) 전달
4. 더 신속한 응답을 주는 집합을 선택 후, 나머지 예비 집합 제거

각 상태에 따라 Epoch 값이 1 ~ 3으로 증가하는 것을 위 그림에서 확인할 수 있습니다.
만약 위와 달리, F 구성원의 복구가 G 구성원의 준비보다 빠르게 되었다면 **ABCDEF** 쿼럼 집합이 선택됩니다.

위와 같이 장애에 대한 멤버십 교체가 진행 도중에도 **사용자 요청(읽기/쓰기)은 정상적으로 진행**됩니다.
Epoch를 이용한 쿼럼집합 개념을 통해 장애 내구성을 높이고, 사용자에게 불편을 최소화할 수 있는 아키텍처 구조를 지니고 있습니다.


### 정리를 마치며
Aurora 아키텍처에서 사용되는 쿼럼 모델에 대해 아래 참조링크에 자세하게 정리된 글이 있어, 해당 내용에 대해 정리해봤습니다.
현재 제가 글을 정리하면서도 이해가 부족한 부분이 있지만, 좀 더 쉽고 간편하게 Aurora 쿼럼 모델을 정리해보자! 라는 목적을 가지고
정리할 수 있어 좋은 경험이었다고 생각됩니다.

#### **참조 링크**
- [Amazon Aurora 내부 들여다보기(1) – 쿼럼 및 상관 오류 해결 방법](https://aws.amazon.com/ko/blogs/korea/amazon-aurora-under-the-hood-quorum-and-correlated-failure/)
- [Amazon Aurora 내부 들여다보기(2) – 쿼럼 읽기 및 상태 변경](https://aws.amazon.com/ko/blogs/korea/amazon-aurora-under-the-hood-quorum-reads-and-mutating-state/)
- [Amazon Aurora 내부 들여다보기(3) – 쿼럼 집합을 이용한 비용 절감 방법](https://aws.amazon.com/ko/blogs/korea/amazon-aurora-under-the-hood-reducing-costs-using-quorum-sets/)
- [Amazon Aurora 내부 들여다보기(4) – 쿼럼 구성원](https://aws.amazon.com/ko/blogs/korea/amazon-aurora-under-the-hood-quorum-membership/)













