### JPA 구동 방식
  - JPA에는 Persistance라는 class에서 persistance.xml을 읽고 EntityManagerFactory class를 생성
  - EntityManagerFactory는 applcation loading 시점에 DB당 하나씩 생성하고, 종료 시점에 닫아줘야 내부적으로 connection pooling에 대한. resource가 release 됨
  - EntityManagerFactory는 thread가 생성될 때 마다 (매 요청마다) EntityManager를 생성
  - EntityManager는 내부적으로 DB connection pool을 사용하여 DB를 사용
  - EntityManager는 thread간 공유를 하지 않고 사용 후 반드시 닫아줘야 한다
  
<img width="846" alt="스크린샷 2022-04-17 오후 3 51 12" src="https://user-images.githubusercontent.com/67041069/163704072-0d081622-7b2c-4dad-8bde-394c4d51afa1.png">
