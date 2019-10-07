## spring.factories ?
 - spring boot application이 실행될때 classpath 안의 spring.factories을 본다.
 - META-INF 폴더에 위치한다.

#### SpringFactoriesLoader가 spring.factories를 읽는다.
```java
package org.springframework.core.io.support;

import java.io.IOException;
import java.net.URL;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.Enumeration;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Properties;
import java.util.Map.Entry;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.core.annotation.AnnotationAwareOrderComparator;
import org.springframework.core.io.UrlResource;
import org.springframework.lang.Nullable;
import org.springframework.util.Assert;
import org.springframework.util.ClassUtils;
import org.springframework.util.ConcurrentReferenceHashMap;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.util.ReflectionUtils;
import org.springframework.util.StringUtils;

public abstract class SpringFactoriesLoader {
    public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
    private static final Log logger = LogFactory.getLog(SpringFactoriesLoader.class);
    private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap();

    public SpringFactoriesLoader() {
    }

    public static <T> List<T> loadFactories(Class<T> factoryClass, @Nullable ClassLoader classLoader) {
        Assert.notNull(factoryClass, "'factoryClass' must not be null");
        ClassLoader classLoaderToUse = classLoader;
        if (classLoader == null) {
            classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
        }

        List<String> factoryNames = loadFactoryNames(factoryClass, classLoaderToUse);
        if (logger.isTraceEnabled()) {
            logger.trace("Loaded [" + factoryClass.getName() + "] names: " + factoryNames);
        }

        List<T> result = new ArrayList(factoryNames.size());
        Iterator var5 = factoryNames.iterator();

        while(var5.hasNext()) {
            String factoryName = (String)var5.next();
            result.add(instantiateFactory(factoryName, factoryClass, classLoaderToUse));
        }

        AnnotationAwareOrderComparator.sort(result);
        return result;
    }

    public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
        String factoryClassName = factoryClass.getName();
        return (List)loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
    }

    private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
        MultiValueMap<String, String> result = (MultiValueMap)cache.get(classLoader);
        if (result != null) {
            return result;
        } else {
            try {
                Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories") : ClassLoader.getSystemResources("META-INF/spring.factories");
                LinkedMultiValueMap result = new LinkedMultiValueMap();

                while(urls.hasMoreElements()) {
                    URL url = (URL)urls.nextElement();
                    UrlResource resource = new UrlResource(url);
                    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                    Iterator var6 = properties.entrySet().iterator();

                    while(var6.hasNext()) {
                        Entry<?, ?> entry = (Entry)var6.next();
                        List<String> factoryClassNames = Arrays.asList(StringUtils.commaDelimitedListToStringArray((String)entry.getValue()));
                        result.addAll((String)entry.getKey(), factoryClassNames);
                    }
                }

                cache.put(classLoader, result);
                return result;
            } catch (IOException var9) {
                throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var9);
            }
        }
    }

    private static <T> T instantiateFactory(String instanceClassName, Class<T> factoryClass, ClassLoader classLoader) {
        try {
            Class<?> instanceClass = ClassUtils.forName(instanceClassName, classLoader);
            if (!factoryClass.isAssignableFrom(instanceClass)) {
                throw new IllegalArgumentException("Class [" + instanceClassName + "] is not assignable to [" + factoryClass.getName() + "]");
            } else {
                return ReflectionUtils.accessibleConstructor(instanceClass, new Class[0]).newInstance();
            }
        } catch (Throwable var4) {
            throw new IllegalArgumentException("Unable to instantiate factory class: " + factoryClass.getName(), var4);
        }
    }
}
```


### Q1. spring factories는 append될까? overwrite될까?

 - 정답은 append됩니다..
pom.xml을 아래 정도만 의존이 걸리게합니다.
```xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.0.RC2</version>
    </parent>

    <!-- Add typical dependencies for a web application -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
```
그런뒤 spring.factories를 찾아봅니다.<br/>
![spring.factories-1.png](/image/spring.factories-1.png)<br/>
![spring.factories-2.png](/image/spring.factories-2.png)<br/>
![spring.factories-3.png](/image/spring.factories-3.png)<br/>
추가로  day03예제에 spring.factories를 작성해봅니다.<br/> 
![spring.factories-4.png](/image/spring.factories-4.png)<br/>
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.navercorp.configuration.ServiceConfig
```

jar파일이 어떤 path로 구성되는지 확인해봅니다.
>jar tf day03-1.0-SNAPSHOT.jar
```
META-INF/spring.factories
...
BOOT-INF/lib/spring-boot-autoconfigure-2.0.0.RC2.jar
...
BOOT-INF/lib/spring-beans-5.0.4.RELEASE.jar
...
BOOT-INF/lib/spring-boot-2.0.0.RC2.jar
```
jar 파일에 다 따로따로 존재하는것 같네요. 그럼 application을 디버깅해서 확인해봅니다.
디버깅 포인트는 SpringFactoriesLoader의 loadFactoryNames메소드입니다.
```java
    public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
        String factoryClassName = factoryClass.getName();
        return (List)loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
    }
```

맨 처음으로 spring-boot-2.0.0.RC2.jar에 있는 spring.factories를 맨처음으로 읽네요.<br />
![spring.factories-5.png](/image/spring.factories-5.png)<br/>
![spring.factories-6.png](/image/spring.factories-6.png)<br/>
![spring.factories-7.png](/image/spring.factories-7.png)<br/>
...

그 다음 쭉 진행하던 순간 아래 값을 읽을 때를 확인해봅니다.
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration
```
<br />
![spring.factories-8.png](/image/spring.factories-8.png)<br />
제가 설정한 `com.navercorp.configuration.ServiceConfig`가 spring-boot-autoconfigure-2.0.0.RC2.jar에 있는 spring.factories의 설정과 함께 load된게 확인이 되었습니다.

이로써 append가 됨을 확인했습니다. 스프링부트 load하는것부터 신기하네요. ㅎㅎ