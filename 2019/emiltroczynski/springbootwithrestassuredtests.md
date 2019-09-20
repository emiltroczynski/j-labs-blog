# Testing Spring Boot Application with REST-assured library
[//]: # "TODO add a photo with laptop with opened rest assured page"
[//]: # "TODO introduction: rest assured tests for application securred with jwt token"

### TL;DR  
Our application has the following requirements:
* it's based on Spring Boot
* there are a few public endpoints  
* an user is authenticated and authorized with JSON Web Token
* build tool is gradle
* create e2e tests which use:  
  * TestNG
  * REST assured library

## Create a simple application
### 1. Scaffolding
Requirements are provided so we have to create a repository and initiate the project.  
We can do it with our favourite IDE or with a command line:
```groovy
mkdir ~/j-labs-blog-springboot-restassured-jwt
cd ~/j-labs-blog-springboot-restassured-jwt
gradle init --type java-application --dsl groovy --test-framework testng --project-name j-labs-blog-springboot-restassured-jwt --package jlabsblog.jwt  
```  
Next step is to add Spring Boot. App class needs a valid annotation and run method:  
```java
package jlabsblog.jwt;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class);
    }
}
``` 
To accomplish that build.gradle requires:
```groovy
implementation 'org.springframework.boot:spring-boot:2.1.8.RELEASE'
implementation 'org.springframework.boot:spring-boot-autoconfigure:2.1.8.RELEASE'
implementation 'org.springframework.boot:spring-boot-starter-web-services'
```
and to make life easier, we use gradle plugin which contains [bootRun task that can be used to run application in an exploded form](https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-running-your-application.html)  
```groovy
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin")
    }
}

plugins {
    id 'org.springframework.boot'  version '2.1.8.RELEASE'
    id 'io.spring.dependency-management' version '1.0.8.RELEASE'
}
```

Execution of bootRun should return:
```text
2019-09-19 20:59:27.904  INFO 10956 --- [main] jlabsblog.jwt.App: Started App in 2.654 seconds (JVM running for 3.076)
```
### 2. Tasks endpoint


## Add few basic tests
[//]: # "TODO debugging"
[//]: # "TODO postman"

## Secure an endpoints

## Update the tests
[//]: # "TODO read application.properties"
[//]: # "TODO logging level"

## Conclusion

### Useful links:
[//]: # "TODO [Link to repo](???)"
[Spring Boot documentation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)  
[Secure endpoint with Jwt library](https://auth0.com/blog/implementing-jwt-authentication-on-spring-boot/#Securing-RESTful-APIs-with-JWTs)  
[Gradle](https://gradle.org/)  
[TestNG](https://testng.org)
[REST assured](http://rest-assured.io/)