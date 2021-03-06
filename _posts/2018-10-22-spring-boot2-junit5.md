---
layout: post
title: "[java] jUnit5 적용기"
category: [tech]
tags: [java,junit5]
---

-   예제 프로젝트:
    [spring-boot2-junit5-spock](https://github.com/ihoneymon/spring-boot2-junit5-spock)

JUnit5가 세상에 모습을 드러내놓은지는 제법 됐다. 새로운 프로젝트를
시작하면서 JUnit5 와 Spock 을 기반으로 한 테스트를 작성하고자 한다.

스프링 부트 2에서 JUnit5 에 대한 의존성을 추가하고 테스트를 작성하는
방법을 설명한다.

```
buildscript {
    ext {
        springBootVersion = '2.0.6.RELEASE'
    }
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    }
}

apply plugin: 'java'
apply plugin: 'eclipse'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'

group = 'io.honeymon.study'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = 10

repositories {
    mavenCentral()
}

dependencies {
    implementation('org.springframework.boot:spring-boot-starter-data-jpa')
    implementation('org.springframework.boot:spring-boot-starter-web')

    runtimeOnly('com.h2database:h2')
    compileOnly('org.projectlombok:lombok')

    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude module: 'junit'
    }
    testImplementation('org.junit.jupiter:junit-jupiter-api:5.2.0')
    testCompile('org.junit.jupiter:junit-jupiter-params:5.2.0')
    testRuntime('org.junit.jupiter:junit-jupiter-engine:5.2.0')
}

test {
    useJUnitPlatform()
}
```

## junit4 제외

스프링 부트 스타터
`org.springframework.boot:spring-boot-starter-test`는 junit4에
대한 의존성을 가지고 있다. 그래서 junit5를 사용하기 위해서는
`spring-boot-starter-test`에 추가되어 있는 `junit4`를
제외해야 한다.

```
testImplementation('org.springframework.boot:spring-boot-starter-test') {
    exclude module: 'junit'
}
```

> **Note**
>
> 그레이들에서 모듈에 대한 의존성 정의는 3개 부분으로 나뉜다.
>
> -   예: \`\`org.springframework.boot:spring-boot-starter-test\`\`
>
>     -   group: \`\`org.springframework.boot\`\`
>
>     -   module: \`\`spring-boot-starter-test\`\`
>
>     -   version: 생략
>
> 이런 구분을 이해하고 나면 위에서 설명한 제외(\`\`exclude\`\`)방법을
> 활용할 수 있을 것이다.
>
> -   그룹과 모듈을 정의하는 경우: \`\`exclude group: *junit*, module:
>     *junit*\`\` 으로 정의할 수 있다.
>
이어서 junit5에 대한 의존성을 추가한다.

```
dependencies {
    // 생략
    testImplementation('org.junit.jupiter:junit-jupiter-api:5.2.0')
    testCompile('org.junit.jupiter:junit-jupiter-params:5.2.0')
    testRuntime('org.junit.jupiter:junit-jupiter-engine:5.2.0')
    // 생략
}
```

이어서 \`\`test\`\` 태스크에서 \`\`useJUnitPlatform()\`\`를 선언한다.
\`\`useJUnitPlatform\`\`는 테스트 실행시 JUnit 플랫폼(JUnit 5)이라는
것을 정의한다.

    /**
     * Specifies that JUnit Platform (a.k.a. JUnit 5) should be used to execute the tests. <p> To configure JUnit platform specific options, see {@link #useJUnitPlatform(Action)}.
     *
     * @since 4.6
     */
    @Incubating
    public void useJUnitPlatform() {
        useJUnitPlatform(Actions.<JUnitPlatformOptions>doNothing());
    }

이렇게 해서 \`\`build.gradle\`\`에서 JUnit5를 사용하기 위한 위존성
정리를 마쳤다.

## 테스트 작성
```

    package io.honeymon.study.junit5;

    import org.junit.jupiter.api.Test;
    import org.junit.jupiter.api.extension.ExtendWith;
    import org.springframework.boot.test.context.SpringBootTest;
    import org.springframework.test.context.junit.jupiter.SpringExtension;

    @ExtendWith(SpringExtension.class)  // 
    @SpringBootTest
    public class SpringBoot2Junit5ApplicationTests {

        @Test // 
        public void contextLoads() {
        }

    }
```

-   [\`\`ExtendWith\`\`](https://junit.org/junit5/docs/5.0.3/api/org/junit/jupiter/api/extension/ExtendWith.html)는
    JUnit5 에서 반복적으로 실행되는 클래스나 메서드에 선언한다.
    [\`\`SpringExtension\`\`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/context/junit/jupiter/SpringExtension.html)는
    스프링 5에 추가된 JUnit 5의 주피터(Jupiter) 모델에서 스프링
    테스트컨텍스트(TestContext)를 사용할 수 있도록 해준다.

-   \`\`\`@Test\`\`의 경로도 변경(\`\`org.junit.Test\`\` →
    \`\`org.junit.jupiter.api.Test\`\`)되었다.

이제 JUnit5를 기반으로 통합테스트를 위한 준비를 마쳤다.

JUnit 5 = JUnit Platform + JUnit Jupiter + JUnit Vintage

> **Note**
>
> junit5는 람다를 기반으로 한 선언(assertion)을 지원한다. junit4에서
> 지원했던 기능이 부족하여 assertJ 의존성을 추가해야 했던 불편함을
> 해소할 수 있다.

참고문서
========

-   [Managing Transitive Dependencies -
    Gradle](https://docs.gradle.org/current/userguide/managing_transitive_dependencies.html)

-   [JUnit5](https://junit.org/junit5/)

    -   [JUnit5 User
        Guide](https://junit.org/junit5/docs/current/user-guide/)


