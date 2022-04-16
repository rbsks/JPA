### Member Entity 생성
  ```java
    package jpapractice;

    import javax.persistence.Entity;
    import javax.persistence.Id;

    @Entity // 객체와 테이블을 맵핑하는 어노테이션
    public class Member {
        @Id // PK 맵핑하는 어노테이션
        private long id;
        private String name;

        public long getId() {
            return id;
        }

        public void setId(long id) {
            this.id = id;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }
    }
  ```
