# Testing Spring Boot Application with REST-assured library
[//]: # "TODO add a photo with laptop with opened rest assured page"
[//]: # "TODO introduction: rest assured tests for application securred with jwt token"

![alt text](Header_photo.jpg "Testing Spring Boot Application with JWT and REST-assured library
")

#### TL;DR  
Our application has the following requirements:
* it's based on Spring Boot
* there are few public endpoints  
* a user is authenticated and authorized with JSON Web Token (JWT)
* build tool is gradle
* e2e tests are implemented with:  
  * TestNG
  * REST assured library

## Create a simple application
### 1. Scaffolding
Requirements are provided, so we have to create a repository and initiate the project.  
We can do it with our favourite IDE or with a command line:
```groovy
mkdir ~/j-labs-blog-springboot-restassured-jwt
cd ~/j-labs-blog-springboot-restassured-jwt
gradle init --type java-application --dsl groovy --test-framework testng --project-name j-labs-blog-springboot-restassured-jwt --package jlabsblog.jwt  
```  
Next step is to add Spring Boot. App class needs a valid annotation and run method:  
##### App.class
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
and necessary dependencies:
##### build.gradle
```groovy
implementation 'org.springframework.boot:spring-boot:2.1.8.RELEASE'
implementation 'org.springframework.boot:spring-boot-autoconfigure:2.1.8.RELEASE'
implementation 'org.springframework.boot:spring-boot-starter-web-services'
```
To make life easier for us, we use gradle plugin which contains [bootRun task that can be used to run application in an exploded form](https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-running-your-application.html)  
##### build.gradle
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

Thanks to the plugin, we can execute bootRun:
##### console log
```text
2019-09-19 20:59:27.904  INFO 10956 --- [main] jlabsblog.jwt.App: Started App in 2.654 seconds (JVM running for 3.076)
```
### 2. Tasks endpoints
Add a new package 'task' under jlabsblog.jwt with the following classes and interface:
##### Task.class
JPA entity, it represents a table stored in database. One instance is one row in the table.
```java
@Entity
public class Task {
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String description;

  protected Task() {}

  public Task(String description) {
    this.description = description;
  }

  public Long getId() {
    return id;
  }

  public String getDescription() {
    return description;
  }

  public void setDescription(String description) {
    this.description = description;
  }
}
```
##### TaskRepository.interface
JPA repository, provides default implementation for CRUD.
```java
public interface TaskRepository extends JpaRepository<Task, Long> {

}
```
##### TaskController.class
```java
@RestController
@RequestMapping("/tasks")
public class TaskController {
  private TaskRepository taskRepository;

  public TaskController(TaskRepository taskRepository) {
    this.taskRepository = taskRepository;
  }

  @GetMapping
  public List<Task> getTask() {
    return taskRepository.findAll();
  }

  @PostMapping
  public void addTask(@RequestBody Task task) {
    taskRepository.save(task);
  }

  @PutMapping("/{id}")
  public void editTask(@PathVariable long id, @RequestBody Task task) {
    Task existingTask = taskRepository.findById(id).get();
    existingTask.setDescription(task.getDescription());
    taskRepository.save(existingTask);
  }

  @DeleteMapping("/{id}")
  public void deleteTask(@PathVariable long id) {
    Task existingTask = taskRepository.findById(id).get();
    taskRepository.delete(existingTask);
  }
}
```
But when we try to run the application:
#### console log
```text
Failed to configure a DataSource: 'url' attribute is not specified and no embedded datasource could be configured.
```
We can easily fix it with adding:
#### build.gradle
```groovy
dependencies {
    ...
    implementation "com.h2database:h2"
    ...
}
```

## Add few basic tests
TaskControllerTests class contains our tests, e.g.: addTest:
#### TaskControllerTests.class
```java
  @Test
  public void addTask() {
    Task retrievedTask = retrieveTask();

    assertTask(retrievedTask, task);
  }
```
assertTask verifies two things:
```java
  private void assertTask(Task actual, Task expected) {
    SoftAssertions assertions = new SoftAssertions();
    assertions.assertThat(actual.getDescription()).isEqualTo(expected.getDescription());
    assertions.assertThat(actual.getId()).isGreaterThan(0);
    assertions.assertAll();
  }
```
value of description variable and if id is greater than zero. We also use SoftAssertion to be sure that no assertions have failed.  

Before each test we create new Task:
```java
  @BeforeMethod
  public void createTask() {
    task = new Task("initialValue");
    given().basePath("/tasks").contentType("application/json").body(task).when().post();
  }
```

and after test we clean it up
```java
  @AfterMethod
  public void cleanUp() {
    deleteTask(id);
  }

  private void deleteTask(Long id) {
    if (id != null) {
      given().basePath("/tasks").when().delete(String.format("%s", id)).then().statusCode(200);
    }
  }
```
editTask and deleteTask, have similar structure.

When tests are added, we start the application and run the tests:
#### console log
```text

===============================================
Default Suite
Total tests run: 3, Failures: 0, Skips: 0
===============================================

```

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