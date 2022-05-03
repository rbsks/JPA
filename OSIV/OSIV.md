### OSIV
  - Hibernate에서는 Open Session In View라고 부름
  - JPA에서는 Open Entity In View라고 부름
  - spring jpa OSIV는 default true
  - OSIV 전략은 트랜잭션 시작처럼 최초 데이터베이스 커넥션 시작 시점부터 API 응답이 끝날 때 까지 영속성 컨텍스트와 데이터베이스 커넥션을 유지함
  - OSIV ON
    - OSIV를 키면 API 응답이 끝날 때 까지 영속성 컨텍스트가 유지되므로 컨트롤러에서도 지연 로딩이 가능함. 이것이 큰 장점
    - 하지만 오랜시간동안 데이터베이스 커넥션 리소스를 사용하기 때문에, 실시간 트래픽이 중요한 애플리케이션에서는 커넥션이 모자랄 수 있음
    <img width="558" alt="스크린샷 2022-05-03 오후 6 10 38" src="https://user-images.githubusercontent.com/67041069/166429253-87a15ee2-c25f-4270-8819-8c4fa5089ef3.png">
  
  - OSIV OFF
    - OSIV를 끄면 트랙잭션을 종료할 떄 영속성 컨텍스트를 닫고, 데이터베이스 커넥션도 반환함. 따라서 커넥션 리소스를 낭비하지 않음
    - 단점은 지연 로딩을 트랙잭션 안에서 처리해야 함. OSIV를 켰을 때 처럼 컨트롤러 단에서 지연 로딩이 사용 불가하고 서비스단에서 지연 로딩을 끝내야 함
    <img width="556" alt="스크린샷 2022-05-03 오후 6 10 46" src="https://user-images.githubusercontent.com/67041069/166429265-e1bce7ca-4194-4df7-bdc1-a0c08fb80447.png">

