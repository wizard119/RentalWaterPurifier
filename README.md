Cloud App. Eng._AWS_Personal Project
# Rent the WaterPurifier

## Table of contents
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
  - [구현](#구현)
    - [DDD의 적용](#DDD-의-적용)
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
        4. 렌탈 신청이 완료되면 정수기를 배송 시작한다
        5. 고객은 렌탈 신청을 취소할 수 있다.
        6. 렌탈 신청이 취소되면 결제가 취소된다.
        7. 결제 취소되면 배송이 취소된다.
        8. 배송 시작/취소되면 정수기 재고 수량을 조정한다. 
        9. 고객은 언제든지 렌탈/배송 현황을 조회한다. (CQRS-View)
       10. 렌탈 신청 상태,배송상태가 바뀔때마다 해당 고객에게 문자 메세지가 발송된다. 

    비기능적 요구사항
        1. 트랜잭션 
            - 렌탈비 결제가 완료되어야만 렌탈 신청을 완료할 수 있다. -> Sync 호출
        2. 장애격리
            - 상품관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다 -> Async(event-driven),Eventual Consistency
            - 결제시스템이 과중되면 렌탈 신청자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 -> Circuit breaker,fallback
        3. 성능
            고객이 수시로 렌탈/배송현황을 MyPage에서 확인할 수 있어야 한다 -> CQRS

## 분석/설계
### Event Storming 결과
    MSAEz 로 모델링한 이벤트스토밍 결과
    http://labs.msaez.io/#/storming/fMnS97KBhKdTR73T20ep9sUs6kE2/69c4cd8c92b1f6f3b5d35ea823ab4921

#### 1.이벤트 도출
 ![image](https://user-images.githubusercontent.com/87048633/129510838-93083903-ff02-40aa-ac7d-2baaa87b8e57.png)

#### 2.부적격 이벤트 탈락
 ![image](https://user-images.githubusercontent.com/87048633/129510865-2c6a3cfe-f293-4f2f-99ac-c18b86d7f7cf.png)
  - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
   - '상품조회됨' :  UI 의 이벤트이지, 업무적인 의미의 이벤트가 아니라서 제외
   - '렌탈신청내역조회' : 별도 VIEW로 구현 예정이라 제외
   - '렌탈결제됨' : '결제승인됨'과 중복되어 제외
 
#### 3.Actor, Command 부착하여 읽기 좋게
 ![image](https://user-images.githubusercontent.com/87048633/129569046-6ba9b793-304d-4848-b20d-53d156c53c5c.png)
 
#### 4.Aggregate으로 묶기
 ![image](https://user-images.githubusercontent.com/87048633/129568675-d7c24316-bfb1-4f0f-a24a-bba7a2443f54.png)
 
#### 5.Bounded Context로 묶기
 ![image](https://user-images.githubusercontent.com/87048633/129569416-dd8adf83-8a7f-4ab0-8b79-8d02135c7fad.png)

#### 6.Policy 부착
 ![image](https://user-images.githubusercontent.com/87048633/129569892-3798600e-603f-42f6-be15-8dc1b0dc2103.png)
 
#### 7.Policy의 이동과 Context 매핑
 ![image](https://user-images.githubusercontent.com/87048633/129570393-c1cec300-5a84-406f-9c15-fecdd9866b3f.png)
 
#### 8.완성된 1차 모형
 ![image](https://user-images.githubusercontent.com/87048633/129510939-ba685a74-0be2-4143-aa9e-c704ab1f0fe9.png)
 
#### 9.1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증
 ![image](https://user-images.githubusercontent.com/87048633/129571570-a6d5aee5-3eaa-4b6e-90ab-c1addedec257.png)
 (O)  1. 관리자는 정수기 정보를 등록한다.</br>
 (O)  2. 고객은 관리자가 등록한 정수기를 조회한 후 원하는 정수기를 렌탈 신청한다.</br>
 (O)  3. 고객은 신청한 정수기의 렌탈비를 결제한다.</br>
 (O)  4. 렌탈 신청이 완료되면 정수기를 배송 시작한다.</br>
 (O)  5. 고객은 렌탈 신청을 취소할 수 있다.</br>
 (O)  6. 렌탈 신청이 취소되면 결제가 취소된다.</br>
 (O)  7. 결제 취소되면 배송이 취소된다.</br>
 (O)  8. 배송 시작/취소되면 정수기 재고 수량을 조정한다. </br>
 (O)  9. 고객은 언제든지 렌탈/배송 현황을 조회한다. (CQRS-View)</br>
 (O) 10. 렌탈 신청 상태,배송상태가 바뀔때마다 해당 고객에게 문자 메세지가 발송된다. </br>
 
#### 10.모델수정
 ![image](https://user-images.githubusercontent.com/87048633/129571664-25aa977a-cc1c-4a18-a127-b131a02a308f.png)

#### 11.비기능적 요구사항에 대한 검증
 ![image](https://user-images.githubusercontent.com/87048633/129572639-d7a2731b-afbe-4a58-a5a5-32c569e96714.png)
 (O) 1. 트랜잭션</br>
       - 렌탈비 결제가 완료되어야만 렌탈 신청을 완료할 수 있다. -> Sync 호출</br>
 (O) 2. 장애격리</br>
       - 상품관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다 -> Async(event-driven),Eventual Consistency</br>
       - 결제시스템이 과중되면 렌탈 신청자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 -> Circuit breaker,fallback</br>
 (O) 3. 성능</br>
       - 고객이 수시로 렌탈/배송현황을 MyPage에서 확인할 수 있어야 한다 -> CQRS</br>
 
#### 12.Hexagonal Architecture Diagram 도출
 ![image](https://user-images.githubusercontent.com/87048633/129574072-aa40e763-316d-4d00-aebb-f08c4654642d.png)
 
 
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
