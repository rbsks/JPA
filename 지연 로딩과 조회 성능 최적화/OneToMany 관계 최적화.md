### OneToMany 조회하는 방법
  - OneToMany 조회하는 방법 6가지
    - 엔티티 직접 노출(권장하지 않으므로 설명하지 않음)
    - 엔티티를 DTO로 변환
    - 페치 조인 후 엔티티를 DTO로 변환
    - 페치 조인 후 엔티티를 DTO로 변환과 페이징
    - JPA에서 DTO로 직접 조회
    - JPA에서 DTO로 직접 조회와 컬렉션 조회 최적화
    - JPA에서 DTO로 직접 저회와 플랫 데이터 최적화

### DTO 추가
  ```java
    @Getter @Setter
    public class OrderCollectionResponseDto {

        private Long id;
        private String name;
        private List<OrderItemDto> orderItems;
        private DeliveryStatus deliveryStatus;
        private LocalDateTime orderDate;
        private OrderStatus orderStatus;
        private Address address;

        @Builder
        public OrderCollectionResponseDto(Long id, String name, List<OrderItem> orderItems, DeliveryStatus deliveryStatus, LocalDateTime orderDate, OrderStatus orderStatus, Address address) {
            this.id = id;
            this.name = name;
            this.orderItems = orderItems.stream()
                    .map(orderItem -> OrderItemDto.builder()
                            .itemName(orderItem.getItem().getName())
                            .orderPrice(orderItem.getOrderPrice())
                            .count(orderItem.getCount())
                            .build())
                    .collect(Collectors.toList());
            this.deliveryStatus = deliveryStatus;
            this.orderDate = orderDate;
            this.orderStatus = orderStatus;
            this.address = address;
        }
    }
    
    @Getter @Setter
    public class OrderItemDto {

        private String itemName;
        private int orderPrice;
        private int count;

        @Builder
        public OrderItemDto(String itemName, int orderPrice, int count) {
            this.itemName = itemName;
            this.orderPrice = orderPrice;
            this.count = count;
        }
    }
  ```
  
### 엔티티를 DTO로 변환
  - xToMany에서 처럼 지연 로딩으로 너무 많은 SQL이 실행
  ```java
    // controller
    @GetMapping("/api/v2/odersCollectionV2")
    public ResponseEntity<?> ordersCollectionV2() {
        HashMap<String, Object> result = new HashMap<>();

        List<OrderCollectionResponseDto> orders = orderService.findOrdersCollectionV2(new OrderSearch());

        result.put("list", orders);
        return new ResponseEntity<HashMap<String, Object>>(result, HttpStatus.OK);
    }
    
    // service
    public List<OrderCollectionResponseDto> findOrdersCollectionV2(OrderSearch orderSearch) {

        return orderRepository.findSearchOrder(orderSearch).stream()
                .map(order -> OrderCollectionResponseDto.builder()
                        .id(order.getId())
                        .orderDate(order.getOrderDate())
                        .orderStatus(order.getStatus())
                        .name(order.getMember().getName()) // lazy 초기화
                        .address(order.getDelivery().getAddress()) // lazy 초기화
                        .orderItems(order.getOrderItems())
                        .build())
                .collect(Collectors.toList());
    }
    
    // repository
    findSearchOrder(OrderSearch orderSearch){
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Order> cq = cb.createQuery(Order.class);
        Root<Order> o = cq.from(Order.class);

        List<Predicate> criteria = new ArrayList<>();

        // 주문 상태 검색
        if(orderSearch.getOrderStatus() != null) {
            Predicate status = cb.equal(o.get("status"), orderSearch.getOrderStatus());
            criteria.add(status);
        }



        // 회원 이름 검색
        if(StringUtils.hasText(orderSearch.getMemberName())) {
            Predicate name = cb.like(o.get("member").get("name"), "%" + orderSearch.getMemberName() + "%");
            criteria.add(name);
        }

        cq.where(cb.and(criteria.toArray(new Predicate[criteria.size()])));
        List<Order> resultList = em.createQuery(cq).getResultList();

        return resultList;
    }
  ```
  - 너무 많은 query로 실행결과 생략
  
### 페치 조인 후 엔티티를 DTO로 변환
  - 페치 조인으로 인해 SQL이 한번만 실행
  - 1 : N 조인이 있으므로 조회 결과 수가 증가하므로 distinct를 써서 중복 제거를 해줘야 함
  - DB의 distinct는 모든 컬럼의 내용이 같아야지만 중복제거를 해주지만 JPA는 Root의 같은 id값인 컬럼이 있으면 Root의 중복을 제거 해줌
  - 1 : N 관계에서 페치 조인을 사용하면 페이징 불가. 페이징 쿼리가 안나가고 모든 데이터를 가져온 후 메모리에서 페이징 처리(매우 위험)
  - 1 : N의 페치 조인은 한개만 사용하자. 여러개를 사용 할 경우 1 : N : N... 이 되는 상황이 발생해서 데이터의 정합성이 안 맞음
  ```java
    // controller
    @GetMapping("/api/v2/odersCollectionV3")
    public ResponseEntity<?> ordersCollectionV3() {
        HashMap<String, Object> result = new HashMap<>();

        List<OrderCollectionResponseDto> orders = orderService.findOrdersCollectionV3(new OrderSearch(), offset, limit);

        result.put("list", orders);
        return new ResponseEntity<HashMap<String, Object>>(result, HttpStatus.OK);
    }
    
    // service
    public List<OrderCollectionResponseDto> findOrdersCollectionV3(OrderSearch orderSearch, int offset, int limit) {

        return orderRepository.findOrdersCollectionV3(orderSearch).stream()
                .map(order -> OrderCollectionResponseDto.builder()
                        .id(order.getId())
                        .orderDate(order.getOrderDate())
                        .orderStatus(order.getStatus())
                        .name(order.getMember().getName()) // lazy 초기화
                        .address(order.getDelivery().getAddress()) // lazy 초기화
                        .orderItems(order.getOrderItems())
                        .build())
    }
    
    // repository 
    public List<Order> findOrdersCollectionV3(OrderSearch orderSearch) {
      return em.createQuery("select distinct o from Order o" +
                        " join fetch o.member m" +
                        " join fetch o.delivery d" +
                        " join fetch o.orderItems oi" +
                        " join fetch oi.item i" , Order.class)
                .getResultList();
    }
  ```
  -  query 실행 결과
  ```sql
    select
        distinct order0_.order_id as order_id1_6_0_,
        member1_.member_id as member_i1_4_1_,
        delivery2_.delivery_id as delivery1_2_2_,
        orderitems3_.order_item_id as order_it1_5_3_,
        item4_.item_id as item_id2_3_4_,
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
        delivery2_.status as status5_2_2_,
        orderitems3_.count as count2_5_3_,
        orderitems3_.item_id as item_id4_5_3_,
        orderitems3_.order_id as order_id5_5_3_,
        orderitems3_.order_price as order_pr3_5_3_,
        orderitems3_.order_id as order_id5_5_0__,
        orderitems3_.order_item_id as order_it1_5_0__,
        item4_.name as name3_3_4_,
        item4_.price as price4_3_4_,
        item4_.stock_quantity as stock_qu5_3_4_,
        item4_.artist as artist6_3_4_,
        item4_.etc as etc7_3_4_,
        item4_.author as author8_3_4_,
        item4_.isbn as isbn9_3_4_,
        item4_.actor as actor10_3_4_,
        item4_.director as directo11_3_4_,
        item4_.dtype as dtype1_3_4_ 
    from
        orders order0_ 
    inner join
        member member1_ 
            on order0_.member_id=member1_.member_id 
    inner join
        delivery delivery2_ 
            on order0_.delivery_id=delivery2_.delivery_id 
    inner join
        order_item orderitems3_ 
            on order0_.order_id=orderitems3_.order_id 
    inner join
        item item4_ 
            on orderitems3_.item_id=item4_.item_id
  ```
  
### 페치 조인 후 엔티티를 DTO로 변환과 페이징
  - 컬렉션을 페치 조인하면 데이터의 개수가 증가하므로 페이징이 불가
  - 이 경우 DB에서 모든 데이터를 가져온 후 메모리에서 페이징을 하고 경고 로그를남김
  - 데이터를 조회할 때 컬렉션을 같이 조회 후 페이징을 사용하고싶다면 이 방법을 사용하면 됨
  - xToOne관계를 모두 페치 조인을 사용하여 조회 후 컬렌셕은 지연 로딩으로 조회함
  - 지연 로딩 성능 최적화를 위해 yml, properties에 hibernate_batch_fetch_size 또는 엔티티에 직접 @BatchSize를 적용
  ```java
    // controller
    @GetMapping("/api/v2/odersCollectionV3")
    public ResponseEntity<?> ordersCollectionV3(@RequestParam(defaultValue = "0") int offset,
                                                @RequestParam(defaultValue = "10") int limit) {
        HashMap<String, Object> result = new HashMap<>();

        List<OrderCollectionResponseDto> orders = orderService.findOrdersCollectionV3(new OrderSearch(), offset, limit);

        result.put("list", orders);
        return new ResponseEntity<HashMap<String, Object>>(result, HttpStatus.OK);
    }
    
    // service
    public List<OrderCollectionResponseDto> findOrdersCollectionV3(OrderSearch orderSearch, int offset, int limit) {
    
        return orderRepository.findSearchOrderPaging(orderSearch, offset, limit).stream()
                .map(order -> OrderCollectionResponseDto.builder()
                        .id(order.getId())
                        .orderDate(order.getOrderDate())
                        .orderStatus(order.getStatus())
                        .name(order.getMember().getName()) // lazy 초기화
                        .address(order.getDelivery().getAddress()) // lazy 초기화
                        .orderItems(order.getOrderItems())
                        .deliveryStatus(order.getDelivery().getStatus())
                        .build())
                .collect(Collectors.toList());
    }
    
    // repository
    public List<Order> findSearchOrderPaging(OrderSearch orderSearch, int offset, int limit){
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Order> cq = cb.createQuery(Order.class);
        Root<Order> o = cq.from(Order.class);

        o.fetch("member", JoinType.INNER);
        o.fetch("delivery", JoinType.INNER);

        List<Predicate> criteria = new ArrayList<>();

        // 주문 상태 검색
        if(orderSearch.getOrderStatus() != null) {
            Predicate status = cb.equal(o.get("status"), orderSearch.getOrderStatus());
            criteria.add(status);
        }



        // 회원 이름 검색
        if(StringUtils.hasText(orderSearch.getMemberName())) {
            Predicate name = cb.like(o.get("member").get("name"), "%" + orderSearch.getMemberName() + "%");
            criteria.add(name);
        }

        cq.where(cb.and(criteria.toArray(new Predicate[criteria.size()])));
        List<Order> resultList = em.createQuery(cq)
                .setFirstResult(offset)
                .setMaxResults(limit)
                .getResultList();

        return resultList;
    }
  ```
  - query 실행 결과
  - batch size를 적용하면 in으로 데이터를 한번에 가져오기때문에 최적화 + 페이징을 사용할 수 있음
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
    where
        1=1 limit ?
        
    select
        orderitems0_.order_id as order_id5_5_1_,
        orderitems0_.order_item_id as order_it1_5_1_,
        orderitems0_.order_item_id as order_it1_5_0_,
        orderitems0_.count as count2_5_0_,
        orderitems0_.item_id as item_id4_5_0_,
        orderitems0_.order_id as order_id5_5_0_,
        orderitems0_.order_price as order_pr3_5_0_ 
    from
        order_item orderitems0_ 
    where
        orderitems0_.order_id in (
            ?, ?
        )
        
    select
        item0_.item_id as item_id2_3_0_,
        item0_.name as name3_3_0_,
        item0_.price as price4_3_0_,
        item0_.stock_quantity as stock_qu5_3_0_,
        item0_.artist as artist6_3_0_,
        item0_.etc as etc7_3_0_,
        item0_.author as author8_3_0_,
        item0_.isbn as isbn9_3_0_,
        item0_.actor as actor10_3_0_,
        item0_.director as directo11_3_0_,
        item0_.dtype as dtype1_3_0_ 
    from
        item item0_ 
    where
        item0_.item_id in (
            ?, ?, ?, ?
        )
  ```
