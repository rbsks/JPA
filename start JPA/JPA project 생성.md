### h2 database 설치
  - h2 database 사이트에서 All Platforms 다운로드 후 압축해제
  - terminal에서 h2.sh 실행
  - Generic H2(Embedded)에서 JDBC URL을 jdbc:h2:~/test로 설정 후 test.mv.db 생성
  - Generic H2(Server), JDBC URL을 jdbc:h2:tcp://localhost/~/test 변경 후 연결을 클릭해 DB 접속

### gradle setting
  ```gradle
    plugins {
        id 'java'
    }

    group 'org.example'
    version '1.0-SNAPSHOT'

    repositories {
        mavenCentral()
    }

    dependencies {
        testImplementation 'org.junit.jupiter:junit-jupiter-api:5.8.1'
        testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.8.1'
        implementation group: 'org.hibernate', name: 'hibernate-entitymanager', version: '5.6.7.Final'
        implementation group: 'com.h2database', name: 'h2', version: '1.4.200'
    }

    test {
        useJUnitPlatform()
    }
  ```
### persistence.xml 설정
  ```xml
    <?xml version="1.0" encoding="UTF-8"?>

    <persistence version="3.0"
            xmlns="https://jakarta.ee/xml/ns/persistence"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence http://xmlns.jcp.org/xml/ns/persistence/persistence_2_2.xsd">
        <persistence-unit name="hgb">
            <class>jpapractice.Member</class>
            <properties>
                <!-- required -->
                <property name="javax.persistence.jdbc.driver" value="org.h2.Driver"/>
                <property name="javax.persistence.jdbc.user" value="sa"/>
                <property name="javax.persistence.jdbc.password" value=""/>
                <property name="javax.persistence.jdbc.url" value="jdbc:h2:tcp://localhost/~/test"/>
                <!-- JPA는 특정 데이터베이스 종속된게 아님 -->
                <!-- 각각의 데이터베이스가 제공하는 SQL 문법과 함수는 조금씩 다름 hibernate.dialect를 설정해줌으로써 JPA가 특정 데이터베이스 문법에 맞게 쿼리를 날림 -->
                <property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect"/>

                <!-- option -->
                <property name="hibernate.show_sql" value="true"/> <!-- console query display -->
                <property name="hibernate.format_sql" value="true"/> <!-- console query 정렬해서 display -->
                <property name="hibernate.use_sql_comments" value="true"/> <!-- console query 정보 insert -->
            </properties>
        </persistence-unit>
    </persistence>
  ```
