# compile and runtime dependencies의 분리
### Gradle 4.x 이하 버전
 - 소스 코드
 ```java
 package org.example.gradledemo;

import com.netflix.util.Pair;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        Pair pair;
        SpringApplication.run(Application.class, args);
    }
}
 ```
 
 - build.gradle
 ```gradle 
 plugins {
    id 'org.springframework.boot' version '1.5.18.RELEASE'
}

group 'org.example'
version '1.0-SNAPSHOT'

sourceCompatibility = 1.8

repositories {
    mavenCentral()
}

dependencies {
    compile("org.springframework.boot:spring-boot-starter-web")
    compile("org.springframework.cloud:spring-cloud-starter-zuul:1.4.7.RELEASE")
    testCompile group: 'junit', name: 'junit', version: '4.12'
}
 ```
 
 - 빌드 결과
```cmd
$ ./gradlew clean build

BUILD SUCCESSFUL in 36s
5 actionable tasks: 5 executed
```

### Gradle 5.x 이상으로 변경시
- 빌드 결과
```cmd
$ ./gradlew clean build
Starting a Gradle Daemon, 1 incompatible Daemon could not be reused, use --status for details

> Task :compileJava FAILED
D:\project\gradle-test\gradle-demo\src\main\java\org\example\gradledemo\Application.java:3: error: package com.netflix.util does not exist
import com.netflix.util.Pair;
                       ^
D:\project\gradle-test\gradle-demo\src\main\java\org\example\gradledemo\Application.java:10: error: cannot find symbol
        Pair pair;
        ^
  symbol:   class Pair
  location: class Application
2 errors

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':compileJava'.
> Compilation failed; see the compiler error output for details.

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.

* Get more help at https://help.gradle.org

BUILD FAILED in 27s
2 actionable tasks: 2 executed
```

### 원인
https://docs.gradle.org/current/userguide/upgrading_version_4.html#rel5.0:pom_compile_runtime_separation
위 이슈는 4.x 이하 버전에 대해 `compile("org.springframework.cloud:spring-cloud-starter-zuul:1.4.7.RELEASE")` compile로  선언하면 하위 모든 의존성에 대해서 runtime으로 선언하더라도 compile로 의존성을 가져오던 java plugin 동작 방식이 완전히 compile과 runtime으로 분리되어 발생한 이슈입니다.
> 해당 Pair class가 있던 라이브러리에 대한 [POM](https://repo1.maven.org/maven2/com/netflix/zuul/zuul-core/1.3.1/zuul-core-1.3.1.pom)
>```xml
><dependency>
>    <groupId>com.netflix.netflix-commons</groupId>
>    <artifactId>netflix-commons-util</artifactId>
>    <version>0.1.1</version>
>    <scope>runtime</scope>
></dependency>
>```

### 해결
너무나 당연하지만 compile시에 사용하기 위해 해당 라이브러리의 compile 의존을 추가합니다.
`compile("com.netflix.netflix-commons:netflix-commons-util:0.1.1")`


# `compile` configuration has been deprecated
The compile configuration has been deprecated for dependency declaration. This will fail with an error in Gradle 7.0. Please use the implementation configuration instead.

`compile` dependency는 deprecated 될 예정입니다. 이에 Gradle에서는 `implementation`이나 `api`를 사용해야합니다. `implementation`, `api`는 아래와 차이가 있습니다.

### 기존 `compile` configuration 소스 코드
 - 아래와 같이 project1, project2의 multi project 환경입니다.
 ![image-20200214-111959-932.png](/files/0a7056be-6f8e-1e78-8170-417ee8116de5)
-----
 - project1 `build.gradle`
   ```gradle
   plugins {
        id 'java'
   }

    group 'com.example.project1'
    version '1.0-SNAPSHOT'

    dependencies {
        compile("com.netflix.netflix-commons:netflix-commons-util:0.1.1")
        compile("org.apache.poi:poi:4.1.1")
    }
   ```
   
  - project1의 자바 코드
     ```java
     package com.example.project1;

     import com.netflix.util.Pair;

     public class ServiceA {
        public static Pair<String, String> test1() {
            return new Pair<>("a", "b");
        }

        public static void test2() {
            new HSSFWorkbook();
        }
     }
     ```
     
 --------
  - project2의 `build.gradle`
      ```gradle
       plugins {
         id 'org.springframework.boot' version '2.0.5.RELEASE'
         id 'io.spring.dependency-management' version '1.0.7.RELEASE'
         id 'java'
    }

    group 'com.example.project2'
    version '1.0-SNAPSHOT'

    dependencies {
        compile project(":project1")
        compile("org.springframework.boot:spring-boot-starter-web")
        compile("org.springframework.cloud:spring-cloud-starter-zuul:1.4.7.RELEASE")
        testCompile group: 'junit', name: 'junit', version: '4.12'
    }
     ```
  - proejct2의 소스코드
      ```java
      package com.example.project2;

    import com.example.project1.ServiceA;
    import com.netflix.util.Pair;
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;

    @SpringBootApplication
    public class Application {
        public static void main(String[] args) {
            ServiceA.test2();
            Pair<String, String> pair = ServiceA.test1();
            SpringApplication.run(Application.class, args);
        }
    }
     ```
--------     
 
  - 빌드 결과
      ```cmd
      $ ./gradlew clean build

      BUILD SUCCESSFUL in 11s
      8 actionable tasks: 8 executed

      ```
      

### `implementation`, `api` configuration로 전환
위 프로젝트는 project2에서 project1을 사용하고 있습니다. 
여기서 `Pair`는 proejct2에서 사용되고 있고 `HSSFWorkbook`는 project2에서만 사용이 되고있다,
이처럼 해당 의존성이 다른 프로젝트에서 사용이 되게하기 위해서는 `api`, 해당 프로젝트에서만 사용이 된다면 `implementation`으로 하면 됩니다. 단, `api`는 `java-library` 플러그인에 있어 해당 플러그인을 사용해야합니다.
> publishing system of Gradle을 사용해보면 `api`는 compile scope로 `implementation`은 `runtime` scope로 publishing 됩니다.

 - project1 `build.gradle`
   ```gradle
   plugins {
        id 'java-library'
   }

    group 'com.example.project1'
    version '1.0-SNAPSHOT'

    dependencies {
        api("com.netflix.netflix-commons:netflix-commons-util:0.1.1")
        implementation("org.apache.poi:poi:4.1.1")
    }
   ```
   
   - project2 `build.gradle`
   ```gradle
   plugins {
       id 'org.springframework.boot' version '2.0.5.RELEASE'
       id 'io.spring.dependency-management' version '1.0.7.RELEASE'
       id 'java'
    }

    group 'com.example.project2'
   version '1.0-SNAPSHOT'

    dependencies {
        implementation project(":project1")
        implementation("org.springframework.boot:spring-boot-starter-web")
        implementation("org.springframework.cloud:spring-cloud-starter-zuul:1.4.7.RELEASE")
        testImplementation group: 'junit', name: 'junit', version: '4.12'
    }
   ```