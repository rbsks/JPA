### Spring boot 2.6 이상 Querydsl 설정
  - Spring boot 2.6 부터는 Querydsl 5.0을 사용
  - build.gradle 
  ```gradle
    buildscript {
        ext {
            queryDslVersion = "5.0.0"
        }
    }

    plugins {
        id 'org.springframework.boot' version '2.7.2'
        id 'io.spring.dependency-management' version '1.0.12.RELEASE'
        //querydsl
        id "com.ewerk.gradle.plugins.querydsl" version "1.0.10"
        id 'java'
    }

    group = 'com.example'
    version = '0.0.1-SNAPSHOT'
    sourceCompatibility = '11'

    configurations {
        compileOnly {
            extendsFrom annotationProcessor
        }
    }

    repositories {
        mavenCentral()
    }

    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
        implementation 'org.springframework.boot:spring-boot-starter-web'

        // query parameter log
        implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.5.6'

        //querydsl 추가
        implementation "com.querydsl:querydsl-jpa:${queryDslVersion}" // Querydsl 라이브러리
        implementation "com.querydsl:querydsl-apt:${queryDslVersion}" // Querydsl 관련 코드 생성 기능 제공

        compileOnly 'org.projectlombok:lombok'
        runtimeOnly 'com.h2database:h2'
        annotationProcessor 'org.projectlombok:lombok'

        // test에서 lombok 사용
        testCompileOnly 'org.projectlombok:lombok'
        testAnnotationProcessor 'org.projectlombok:lombok'

        testImplementation 'org.springframework.boot:spring-boot-starter-test'
    }

    tasks.named('test') {
        useJUnitPlatform()
    }

    //querydsl 추가 시작
    def querydslDir = "$buildDir/generated/querydsl"

    querydsl {
        jpa = true
        querydslSourcesDir = querydslDir
    }

    sourceSets {
        main.java.srcDir querydslDir
    }

    compileQuerydsl {
        options.annotationProcessorPath = configurations.querydsl
    }

    configurations {
        compileOnly {
            extendsFrom annotationProcessor
        }
        querydsl.extendsFrom compileClasspath
    }

    // 추가 끝
  ```
