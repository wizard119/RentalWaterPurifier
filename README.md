# RentalWaterPurifier
Cloud App. Eng._AWS_Personal Project

서비스시나리오

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
   - 고객이 수시로 렌탈/배송현황을 MyPage에서 확인할 수 있어야 한다 -> CQRS

분석/설계
Event Storming 결과
MSAEz 로 모델링한 이벤트스토밍 결과: http://labs.msaez.io/#/storming/fMnS97KBhKdTR73T20ep9sUs6kE2/69c4cd8c92b1f6f3b5d35ea823ab4921

 이벤트 도출
https://user-images.githubusercontent.com/87048633/129510838-93083903-ff02-40aa-ac7d-2baaa87b8e57.png
![image](https://user-images.githubusercontent.com/87048633/129510838-93083903-ff02-40aa-ac7d-2baaa87b8e57.png)

 부적격 이벤트 탈락
 https://user-images.githubusercontent.com/87048633/129510865-2c6a3cfe-f293-4f2f-99ac-c18b86d7f7cf.png
 
 액터,커맨드 부착하여 읽기 좋게
 https://user-images.githubusercontent.com/87048633/129510881-88f8567f-dc12-4671-87c1-644e2308630f.png
