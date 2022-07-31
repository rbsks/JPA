### Paging
  - Spring Data JPA가 제공하는 JpaRepository 인터페이스가 PagingAndSortingRepository를 확장하고 있기 때문에 간편하게 페이징 처리 가능
  - Pagable의 구현체인 PageRequest를 파라미터로 넘겨 줌으로써 페이징 구현이 가능
  - PageRequest에는 page(0부터 시작), offset, sort 조건을 넣을 수 있는데 sort 조건이 까다로워 지면 @Query 부분에 직접 sort 조건을 적어주는게 좋음
  - 실무에선 연관관계를 전부 fatch lazy를 쓰는데 ManytoOne은 N : 1이므로 데이터가 뻥튀기 돼서 오지 않으므로 fetch join으로 N + 1 이슈를 해결 할 수 있음
  - 반대로 oneToMany는 1 : N 이므로 데이터가 뻥튀기가 돼서 원하는 결과 값이 안나올 수 있는데 이 부분은 yml 또는 properties에 BatchSize를 지정해서 N + 1의 이슈를 해결 할 수 있음. 그리고 데이터가 뻥튀기 되는 부분은 JPQL에 distinct를 사용해줘야 
  ```java
    // controller
     /**
     * GET / account/{user_id}?page={page}&offset={offset}
     * 파라미터 : 사용자 아이디, 페이지, 오프셋
     * 정책 : 사용자가 없을 경우 실패응답, 계좌가 없을 경우 빈 데이터
     * 성공 응답 : List<계좌번호, 잔액> 구조로 응답
     */
    @GetMapping("/account/{user_id}")
    public PagingAccountInfo getAccountPaging(
            @PathVariable("user_id") Long userId,
            @RequestParam("page") int page,
            @RequestParam("offset") int offset
    ) {
        return accountService.getAccountsByUserIdPaging(userId, page, offset);
    }
    
    // service
    @Transactional
    public PagingAccountInfo getAccountsByUserIdPaging(Long userId, int page, int offset) {
        AccountUser accountUser = getAccountUser(userId);

        // Pageable의 구현체
        PageRequest pageRequest = PageRequest.of(page, offset);

        Page<Account> paging_accounts =
                accountRepository.findByAccountUser(accountUser, pageRequest);

        PagingAccountInfo pagingAccountInfo = PagingAccountInfo.builder()
                .totalCount(paging_accounts.getTotalElements()) // 총 개수
                .totalPage(paging_accounts.getTotalPages()) // 총 페이징 수
                .currentPage(paging_accounts.getNumber()) // 현제 페이지
                .hasNext(paging_accounts.hasNext()) // 다음 페이지 여부
                .accountInfos(paging_accounts.stream()
                        .map(account -> AccountInfo.builder()
                                .accountNumber(account.getAccountNumber())
                                .balance(account.getBalance())
                                .accountStatus(account.getAccountStatus())
                                .build())
                        .collect(Collectors.toList()))
                .build();

        return pagingAccountInfo;
    }
    
    // repository
    // 실무에서는 total count 쿼리로 인한 성능 저하가 나올 수 있음
    // 쿼리가 복잡해지고 성능 저하가 나오면 @Query에 countQuery를 이용해 카운트 쿼리도 따로 분리해서 사용할 수 있으니 분리해서 직접 작성하는게 성능적으로 좋음
    @Query(value = "select a from Account a where a.accountUser = :accountUser", countQuery = "select count(a) from Account a")
    Page<Account> findByAccountUser(@Param("accountUser") AccountUser accountUser, Pageable pageable);
  ```
  
  - 유저 계좌에 대한 페이징 응답 값
  - 총 7개의 계좌가 있고 데이터를 5개씩 불러온 결과임
  - 총 2페이지가 있고 현재 0페이지(페이지는 0부터 시작)이고 hasNext가 true 이면 다음 페이지가 있다는 것
  ![스크린샷 2022-07-31 오후 4 44 53](https://user-images.githubusercontent.com/67041069/182015546-291a8bdf-2818-44b0-b517-1285d65eed75.png)
