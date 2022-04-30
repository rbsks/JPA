### xToOne을 조회하는 방법 (엔티티 연관관계 fetch = FetchTyp.LAZY로 설정한 경우)
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
  
### 엔티티를 DTO로 변환
  - 엔티티를 DTO로 변환하는 가장 일반적인 방법
  - 이 방법의 치명적인 단점은 Order를 불러오고 fetch lazy로 설정 되어있는 연관관계를 디비에서 다시 조회를 하게 됨
  - 그 결과 쿼리는 총 1 + N번이 실행이 된다 엔티티를 외부에 직접 노출하지 않을 때의 장점들이 있지만 성능면에서는 엔티티를 직접 반환하는 경우랑 똑같음
  ```java
    // controller
    /**
     * XToOne
     * Order
     * Order -> Member
     * Order -> Delivery
     * 간단한 주문 조회 V2: 엔티티를 DTO로 변환
     * lazy loding으로 인해 order를 찾고 그에 연관된 객체들에 대한 쿼리가 N번 발생
     */
    @GetMapping("/api/v2/orders")
    public ResponseEntity<?> ordersV2() {
        HashMap<String, Object> result = new HashMap<>();

        List<OrderCollectionResponseDto> orders = orderService.findOrdersCollectionV2(new OrderSearch());

        result.put("list", orders);
        return new ResponseEntity<HashMap<String, Object>>(result, HttpStatus.OK);
    }
    
    // service
    public List<OrderResponseDto> findOrdersV2(OrderSearch orderSearch) {

        return orderRepository.findSearchOrder(orderSearch).stream()
                .map(order -> OrderResponseDto.builder()
                        .id(order.getId())
                        .orderDate(order.getOrderDate())
                        .orderStatus(order.getStatus())
                        .name(order.getMember().getName()) // lazy 초기화
                        .address(order.getDelivery().getAddress()) // lazy 초기화
                        .build())
                .collect(Collectors.toList());
    }
    
    // repository
    public List<Order> findSearchOrder(OrderSearch orderSearch){
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Order> cq = cb.createQuery(Order.class);
        Root<Order> o = cq.from(Order.class);
        Join<Object, Object> m = o.join("member", JoinType.INNER);

        List<Predicate> criteria = new ArrayList<>();

        // 주문 상태 검색
        if(orderSearch.getOrderStatus() != null) {
            Predicate status = cb.equal(o.get("status"), orderSearch.getOrderStatus());
            criteria.add(status);
        }

        // 회원 이름 검색
        if(StringUtils.hasText(orderSearch.getMemberName())) {
            Predicate name = cb.like(m.<String>get("name"), "%" + orderSearch.getMemberName() + "%");
            criteria.add(name);
        }

        cq.where(cb.and(criteria.toArray(new Predicate[criteria.size()])));
        List<Order> resultList = em.createQuery(cq).getResultList();

        return resultList;
    }
  ```
  - query 실행 결과
  
  ```sql
    -- 최초 order 조회 총 2
    select
        order0_.order_id as order_id1_6_,
        order0_.delivery_id as delivery4_6_,
        order0_.member_id as member_i5_6_,
        order0_.order_date as order_da2_6_,
        order0_.status as status3_6_ 
    from
        orders order0_ 
    inner join
        member member1_ 
            on order0_.member_id=member1_.member_id 
    where
        1=1
        
    --order entity에 지연 로딩 연관관계로 설정된 member 조회
    select
      member0_.member_id as member_i1_4_0_,
      member0_.city as city2_4_0_,
      member0_.street as street3_4_0_,
      member0_.zipcode as zipcode4_4_0_,
      member0_.name as name5_4_0_ 
    from
        member member0_ 
    where
        member0_.member_id=?
        
    -- order entity에 지연 로딩 연관관계로 설정된 delivery 조회
    select
      delivery0_.delivery_id as delivery1_2_0_,
      delivery0_.city as city2_2_0_,
      delivery0_.street as street3_2_0_,
      delivery0_.zipcode as zipcode4_2_0_,
      delivery0_.status as status5_2_0_ 
    from
        delivery delivery0_ 
    where
        delivery0_.delivery_id=?
        
    -- order의 결과가 2개이기 때문에 2번째 order에 지연 로딩 member 조회
    select
      member0_.member_id as member_i1_4_0_,
      member0_.city as city2_4_0_,
      member0_.street as street3_4_0_,
      member0_.zipcode as zipcode4_4_0_,
      member0_.name as name5_4_0_ 
    from
        member member0_ 
    where
        member0_.member_id=?
        
    -- order의 결과가 2개이기 때문에 2번째 order에 지연 로딩 delivery 조회
    select
      delivery0_.delivery_id as delivery1_2_0_,
      delivery0_.city as city2_2_0_,
      delivery0_.street as street3_2_0_,
      delivery0_.zipcode as zipcode4_2_0_,
      delivery0_.status as status5_2_0_ 
    from
        delivery delivery0_ 
    where
        delivery0_.delivery_id=?
  ```
  
### 패치 조인으로 엔티티를 조회 후 DTO로 변환
  - 패치 조인을 이용해 fetch lazy로 설정된 연관관계를 미리 조회 하기 때문에 지연 로딩일 발생하지 않아 쿼리가 1번 실행
  ```java
    // controller
    @GetMapping("/api/v3/orders")
    public ResponseEntity<?> ordersV3() {
        HashMap<String, Object> result = new HashMap<>();

        List<OrderResponseDto> orders = orderService.findOrdersV3(new OrderSearch());

        result.put("list", orders);
        return new ResponseEntity<HashMap<String, Object>>(result, HttpStatus.OK);
    }
    
    // service
    public List<OrderResponseDto> findOrdersV3(OrderSearch orderSearch) {

        return orderRepository.findOrderV3(orderSearch).stream()
                .map(order -> OrderResponseDto.builder()
                        .id(order.getId())
                        .orderDate(order.getOrderDate())
                        .orderStatus(order.getStatus())
                        .name(order.getMember().getName()) // lazy 초기화
                        .address(order.getDelivery().getAddress()) // lazy 초기화
                        .build())
                .collect(Collectors.toList());
    }
    
    // repository
    public List<Order> findOrderV3(OrderSearch orderSearch) {
        return em.createQuery("select o from Order o" +
                " join fetch o.member m" +
                " join fetch o.delivery d", Order.class)
                .getResultList();
    }
  ```
  
  - query 실행 결과
  ```sql
    select
        order0_.order_id as order_id1_6_0_,
        member1_.member_id as member_i1_4_1_,
        delivery2_.delivery_id as delivery1_2_2_,
        order0_.delivery_id as delivery4_6_0_,
        order0_.member_id as member_i5_6_0_,
        order0_.order_date as order_da2_6_0_,
        order0_.status as status3_6_0_,
        member1_.city as city2_4_1_,
        member1_.street as street3_4_1_,
        member1_.zipcode as zipcode4_4_1_,
        member1_.name as name5_4_1_,
        delivery2_.city as city2_2_2_,
        delivery2_.street as street3_2_2_,
        delivery2_.zipcode as zipcode4_2_2_,
        delivery2_.status as status5_2_2_ 
    from
        orders order0_ 
    inner join
        member member1_ 
            on order0_.member_id=member1_.member_id 
    inner join
        delivery delivery2_ 
            on order0_.delivery_id=delivery2_.delivery_id
  ```
   
### DTO로 바로 조회
  - 일반적인 SQL을 사용할 때 처럼 원하는 값을 선택해서 조회할 수 있음
  - new 명령어를 사용해서 JPQL의 결과를 DTO로 즉시 변환
  - select 절에 원하는 데이터를 직접 선택하므로 DB -> Application 네트워크 용량 최적화 (생각보다 미비함)
  - 레포지토리의 재사용성이 떨어지고 API 스펙에 맞춘 코드가 레포지토리에 들어가는 단점과 new 뒤에 패키지명을 풀로 써줘야해서 코드의 가독성이 떨어짐
  ```java
  public List<OrderQueryResponseDto> findOrderV4(OrderSearch orderSearch) {
        return em.createQuery("select new com.jpabook.jpashop.controller.order.OrderQueryResponseDto(o.id, m.name, o.orderDate, o.status, d.address) " +
                        " from Order o" +
                        " join o.member m" +
                        " join o.delivery d", OrderQueryResponseDto.class)
                .getResultList();
    }
  ```
