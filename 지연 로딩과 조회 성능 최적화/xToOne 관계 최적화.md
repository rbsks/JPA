### xToOne을 조회하는 방법 (엔티티의 모든 필드가 fetch = FetchTyp.LAZY로 설정한 경우)
  - ManyToOne, OneToOne을 조회하는 방법 4가지
    - 엔티티를 직접노출(권장하지 않음)
      - 엔티티를 직접 노출하면 발생하는 문제점
        - 엔티티에 프레젠테이션 계층을 위한 로직이 추가됨
        - 기본적으로 엔티티의 모든 값이 노출 됨
        - 응답 스펙을 맞추기 위해 로직이 추가 됨(@JsonIgnore, 별도의 뷰 로직 등등)
        - 실무에서는 같은 엔티티에 대해 API 용도에 따라 다양하게 만들어지는데, 한 엔티티에 각각의 API를 위한 프레젠테이션 응답 로직을 담기는 어려움
        - 엔티티가 변하면 API 스펙이 변함
        - 추가로 컬렉션을 직접 반환하면 향후 API 스펙을 변경하기 어려움
    - 엔티티를 DTO로 변환
    - 페치 조인으로 엔티티를 조회 후 DTO로 변환
    - DTO로 바로 조회
    
### 엔티티를 직접 노출
  ```java
    @GetMapping("/api/v1/simple-orders")
    public List<Order> ordersV1() {
      List<Order> all = orderRepository.findAllByString(new OrderSearch());
      for (Order order : all) {
        order.getMember().getName(); //Lazy 강제 초기화
        order.getDelivery().getAddress(); //Lazy 강제 초기환 
      }
      return all; 
    }
  ```
  
  - 엔티티를 직접 노출하는 경우에는 양방향 연관관계가 걸린 곳에 순한 참조가 일어나기 때문에 한 곳에 꼭 @JsonIgnore를 추가해야 함
  - jackson 라이브러리는 프록시 객체를 json으로 어떻게 생성해야하는지 몰라서 예외가 발생하는데 이 문제를 Hibernate5Module을 사용하여 해결 할 수 있지만 권장하지 않음
  - build.gradle에 implementation 'com.fasterxml.jackson.datatype:jackson-datatype-hibernate5'를 추가 후 Application에 밑 코드를 추가
  ```java
    @SpringBootApplication
    public class JpashopApplication {
    
      public static void main(String[] args) {
        SpringApplication.run(JpashopApplication.class, args);
      }
    
      @Bean
      Hibernate5Module hibernate5Module() {
        Hibernate5Module hibernate5Module = new Hibernate5Module();
        //강제 지연 로딩 설정 
        hibernate5Module.configure(Hibernate5Module.Feature.FORCE_LAZY_LOADING, true);
        return hibernate5Module;
      }
    }
  ```
