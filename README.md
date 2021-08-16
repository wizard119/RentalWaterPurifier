Cloud App. Eng._AWS_Personal Project
# Rent the WaterPurifier

## Table of contents
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
  - [구현](#구현)
    - [DDD의 적용](#ddd-의-적용)
    - [Polyglot Persistent](#Polyglot-Persistent)
    - [Polyglot Programming](#Polyglot-Programming)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출과 Eventual Consistency](#비동기식-호출과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#CI/CD-설정)
    - [동기식 호출/서킷 브레이킹/장애격리](#동기식-호출/서킷-브레이킹/장애격리)
    - [AutoScale Out](#AutoScale-Out)
    - [무정지 재배포](#무정지-재배포)
    - [개발 운영 환경 분리](#개발-운영-환경-분리)
    - [모니터링](#모니터링)

## 서비스 시나리오
    기능적 요구사항
        1. 관리자는 정수기 정보를 등록한다.
        2. 고객은 관리자가 등록한 정수기를 조회한 후 원하는 정수기를 렌탈 신청한다.
        3. 고객은 신청한 정수기의 렌탈비를 결제한다
        4. 결제가 완료되면 정수기 재고 수량을 조정한다. 
        5. 렌탈 신청이 완료되면 정수기를 배송 시작한다
        6. 고객은 렌탈 신청을 취소할 수 있다.
        7. 렌탈 신청이 취소되면 결제가 취소된다.
        8. 결제 취소되면 정수기 재고 수량을 조정한다. 
        9. 결제 취소되면 배송이 취소된다.
        10. 고객은 언제든지 렌탈/배송 현황을 조회한다. (CQRS-View)
        11. 렌탈 신청 상태,배송상태가 바뀔때마다 해당 고객에게 문자 메세지가 발송된다. 

    비기능적 요구사항
        1. 트랜잭션 
            - 렌탈비 결제가 완료되어야만 렌탈 신청을 완료할 수 있다. -> Sync 호출
        2. 장애격리
            - 상품관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다 -> Async (event-driven), Eventual Consistency
            - 결제시스템이 과중되면 렌탈 신청자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 -> Circuit breaker, fallback
         3. 성능
            고객이 수시로 렌탈/배송현황을 MyPage에서 확인할 수 있어야 한다 -> CQRS

## 분석/설계
### Event Storming 결과
    MSAEz 로 모델링한 이벤트스토밍 결과: http://labs.msaez.io/#/storming/fMnS97KBhKdTR73T20ep9sUs6kE2/69c4cd8c92b1f6f3b5d35ea823ab4921

#### 1.이벤트 도출
 ![image](https://user-images.githubusercontent.com/87048633/129510838-93083903-ff02-40aa-ac7d-2baaa87b8e57.png)

#### 2.부적격 이벤트 탈락
 ![image](https://user-images.githubusercontent.com/87048633/129510865-2c6a3cfe-f293-4f2f-99ac-c18b86d7f7cf.png)
 
#### 3.Actor, Command 부착하여 읽기 좋게
 ![image](https://user-images.githubusercontent.com/87048633/129510881-88f8567f-dc12-4671-87c1-644e2308630f.png)
 
#### 4.Aggregate으로 묶기
 ![image](https://user-images.githubusercontent.com/87048633/129510894-2935b76f-b7f0-4289-99e0-7d8c847e72a1.png)
 
#### 5.Bounded Context로 묶기
 ![image](https://user-images.githubusercontent.com/87048633/129510902-38b255f4-ed81-4de3-a96c-f45dd553bfcc.png)

#### 6.Policy 부착
 ![image](https://user-images.githubusercontent.com/87048633/129510915-15e35f37-1535-46f9-b74f-41f990cd9695.png)

#### 7.Policy의 이동과 Context 매핑
 ![image](https://user-images.githubusercontent.com/87048633/129510929-3ec576d0-5941-4a31-be75-c1bad607a2b7.png)
 
#### 8.완성된 1차 모형
 ![image](https://user-images.githubusercontent.com/87048633/129510939-ba685a74-0be2-4143-aa9e-c704ab1f0fe9.png)
 
#### 9.1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증
 ![image](https://user-images.githubusercontent.com/87048633/129510952-f71928e9-9c33-4d05-8c68-ab3b8af7752f.png)

#### 10.모델수정
 ![image](https://user-images.githubusercontent.com/87048633/129510961-92b5eb54-dea4-4c75-a2d8-67946700b63a.png)

#### 11.비기능적 요구사항에 대한 검증
 ![image](https://user-images.githubusercontent.com/87048633/129510971-3eb107a0-8fc7-4d8e-92ea-028c79b7d1c8.png)
 
#### 12.헥사고날 아키텍처 다이어그램 도출
 ![image](https://user-images.githubusercontent.com/87048633/129513867-8df97e8b-e44f-4e00-81ce-84d0f5f9d7cb.png)

 
## 구현
### DDD의 적용
### Polyglot Persistent
### Polyglot Programming
### 동기식 호출과 Fallback 처리
### 비동기식 호출과 Eventual Consistency



## 운영
### CI/CD 설정
### 동기식 호출/서킷 브레이킹/장애격리
### AutoScale Out
### 무정지 재배포
### 개발 운영 환경 분리
### 모니터링
