### Q-Type
  - Q-Type class는 Querydsl 설정을 성공적으로 마치면 @Entity가 붙은 클래스를 찾아 컴파일 시점에 자동으로 생성
- 예를 들어 @Entity가 붙은 Member.java class가 있다면 QMember.java 파일이 자동으로 생성
- Q-type class는 Querydsl을 사용하여 메소드 기반으로 쿼리를 작성할 때 우리가 만든 domain class의 구조를 설명해주는 메타데이터 역할을 하며 쿼리의 조건을 설정할 때 사용
- Q-Type 인스턴스를 사용하는 방법
```java
  QMember qMember1 = new QMember("m") // 별칭 직접 사용
  
  // 컴파일 시점에 자동으로 생성된 QMember.java class에 static final로 이미 기본 인스턴스가 있음 
  // 기본 인스턴스를 static import해주면 더 짧은 문법으로 사용 가능
  // public static final QMember member = new QMember("member1");
  QMember qMember2 = QMember.member // 기본 인스턴스 사용
```
