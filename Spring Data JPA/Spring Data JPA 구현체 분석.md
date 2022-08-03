### Spring Data JPA 구현체 SimpleJpaRepository 분석
  - Spring Data JPA가 제공하는 공통 인터페이스의 구현체임
  - @Repository는 스프링 Bean의 component scan의 대상이 되고 JPA 예외를 스프링이 추상화한 예외로 변환
  - @Taransaction 적용
    - JPA의 모든 변경은 Transaction 안에서 동작
    - Spring Data JPA는 변경(등록, 수정, 삭제)메서드를 트랜잭션 처리
    - Service 계층에서 Transaction을 시작하지 않으면 Repository에서 Transaction 시작하고 Service 계층에서 Transaction을 시작하면 Repository는 해당 Transaction을 전파 받아서 사용
    - SimpleJpaRepository에 Transaction이 걸려있기 때문에 @Transaction을 따로 적어주지 않아도 등록, 수정, 삭제가 가능
    - 하지만 Service계층에서 Transaction을 전파 받지 않고 사용하면 Persistence Context가 유지 되지 않기 때문에 Persistence Context를 유지해야 한다면 Service 계층에서 Transaction을 전파받아서 사용해야 함
    - SimpleRespository는 기본적으로 Transaction readOnly true이지만 등록, 수정, 삭제에 관한 메서드는 readOnly false이다 (@Transaction은 readOnly false가 default)
  ```java
    @Repository
    @Transactional(readOnly = true)
    public class SimpleJpaRepository<T, ID> implements JpaRepositoryImplementation<T, ID> {
      // ...생략

      // 새로운 엔티티면 저장(persist)
      // 새로운 엔티티가 아니면 병합(merge)
      @Transactional
      @Override
      public <S extends T> S save(S entity) {

        Assert.notNull(entity, "Entity must not be null.");

        if (entityInformation.isNew(entity)) {
          em.persist(entity);
          return entity;
        } else {
          // merge는 select query가 한번 더 나감 (단점)
          // 만약 데이터가 없으면 새로운 객체로 가정을하고 persist 함
          // 데이터가 있으면 가져온 데이터를 지금 등록하려는 데이터로 덮어 씀
          // 데이터 변경은 dirty checking(변경 감지)을 통해 해야함 (가급적이면 merge를 사용 안 하는게 낫다)
          // merge는 영속상태의 엔티티가 영속상태를 벗어났다가 다시 영속상태가 되어야 할 때 사용하는 것
          return em.merge(entity);
        }
      }
    }
  ```
