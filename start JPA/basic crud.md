### basic crud
``` java
  package jpapractice;

  import org.hibernate.annotations.common.reflection.XMember;

  import javax.persistence.EntityManager;
  import javax.persistence.EntityManagerFactory;
  import javax.persistence.EntityTransaction;
  import javax.persistence.Persistence;

  public class JpaMain {
      public static void main(String[] args) {
          // 프로젝트 런 시점에 딱 한번만 생성
          EntityManagerFactory emf =  Persistence.createEntityManagerFactory("hgb");

          // therad가 생성될 때 마다(매 요청마다) EntityManagerFactory에서 EntityManager 생성하고 트랙잭션이 끝나면 꼭 닫아야 함
          // EntityManager는 내부적으로 DB connection pool을 사용해서 DB에 붙음
          // thread간 공유 금지
          EntityManager entityManager = emf.createEntityManager();

          // 데이터를 변경하는 모든 작업은 JPA에서는 트랜잭션 안에서 해야 함
          EntityTransaction tx = entityManager.getTransaction();

          tx.begin();

          try {
            // create member  code
            Member member = new Member();
            member.setId(2L);
            member.setName("gyubin");

            entityManager.persist(member);

            // find member, remove memeber code
            Member member = entityManager.find(Member.class, 1L);
            System.out.println(member.getId());
            System.out.println(member.getName());

            entityManager.remove(member);
            
            /* 
              update member code
              JPA를 통해 entity를 가져오 JPA가 entity를 관리하고
              entity의 데이터가 변경이 됐는지 안 됐는지 transaction commit 시점에 다 체크를해서
              변경된 데이터가 있으면 DB data를 업데이트 한다
            */
            
            Member member = entityManager.find(Member.class, 1L);
            member.setName("hgb");
          
            tx.commit();
          } catch (Exception e) {
              tx.rollback();
          } finally {
              entityManager.close();
          }

          emf.close();
      }
  }
```
