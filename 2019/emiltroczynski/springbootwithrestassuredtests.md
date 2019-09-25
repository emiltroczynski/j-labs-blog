# Testing Spring Boot Application secured with JSON Web Tokens using REST-assured library

![alt text](Header_photo.jpg "Testing Spring Boot Application secured with JSON Web Tokens using REST-assured library
")

#### TL;DR  
REST-assured library is very well crafted library making test effort a lot simpler and more efficient. 
Combined with Spring Boot, gradle and TestNG, it allows implement complicated application with ease.  

We're going to go over building an application with the following requirements:
* we use Spring Boot as framework
* there are few public endpoints  
* an user is authenticated and authorized with the JSON Web Token (JWT)
* build tool is gradle
* tests are implemented with:  
  * TestNG
  * REST assured library

## 1. Create a simple application
### 1.1. The scaffolding
To create a repository and initiate the project, we can use our favourite IDE or a command line:
##### command line
```groovy
mkdir ~/j-labs-blog-springboot-restassured-jwt
cd ~/j-labs-blog-springboot-restassured-jwt
gradle init --type java-application --dsl groovy --test-framework testng --project-name j-labs-blog-springboot-restassured-jwt --package jlabsblog.jwt  
```  
The next step is add Spring Boot. App class needs a valid annotation and run method:  
##### App
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
To make life easier for us, we use gradle plugin which contains [bootRun task that can be used to run application in an exploded form](https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-running-your-application.html#using-boot-running-with-the-gradle-plugin)  
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
### 1.2. Tasks endpoints
Our application is running so we can proceed with adding endpoints.  
Add a new package 'task' under jlabsblog.jwt with the following classes and interface:
##### Task
JPA entity, it represents a table stored in a database. One instance is one row in the table.
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
##### TaskRepository
JPA repository, provides default implementation for CRUD.
```java
public interface TaskRepository extends JpaRepository<Task, Long> {

}
```
##### TaskController
Class responsible for the endpoints.
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
We can run the application with bootRun, but that will fail with the message:
##### console log
```text
Failed to configure a DataSource: 'url' attribute is not specified and no embedded datasource could be configured.
```
We can easily fix that with adding dependency for JPA, for example in memory database H2:
##### build.gradle
```groovy
dependencies {
    ...
    implementation "com.h2database:h2"
    ...
}
```

## 2. Add a few basic tests
We add new package jlabsblog.jwt.task in src/test.  
##### TaskControllerTests
TaskControllerTests class contains our tests, below are method from that class.

addTask method is the test of, how the name implies, adding a new task.
```java
  @Test
  public void addTask() {
    Task retrievedTask = retrieveTask();

    assertTask(retrievedTask, task);
  }
```
assertTask verifies value of a description and if id is greater than zero.  
We also use SoftAssertion to be sure that no assertions have failed.
```java
  private void assertTask(Task actual, Task expected) {
    SoftAssertions assertions = new SoftAssertions();
    assertions.assertThat(actual.getDescription()).isEqualTo(expected.getDescription());
    assertions.assertThat(actual.getId()).isGreaterThan(0);
    assertions.assertAll();
  }
```

Before each test we create new Task:
```java
  @BeforeMethod
  public void createTask() {
    task = new Task("initialValue");
    given().basePath("/tasks").contentType("application/json").body(task).when().post();
  }
```

and after the test we clean it up:
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
editTask() and deleteTask(), tests of editing and deleting task, have similar structure.

The tests are added, so we start the application and run them:
##### console log
```text

===============================================
Default Suite
Total tests run: 3, Failures: 0, Skips: 0
===============================================

```
There was no failure, so let's secure the application.

## 3. Secure the endpoints
This section is heavily based on: [Secure endpoint with JWT library](https://auth0.com/blog/implementing-jwt-authentication-on-spring-boot/#Securing-RESTful-APIs-with-JWTs)  
We need two services: one for managing users and second for authentication and authorization. 

### 3.1. Users
In package jlabsblog.jwt.user we added two classes and one interface. 
Structure is very similar to package with the tasks.
We have JwtUser which is JPA entity, JwtUserRepository which is implementation for CRUD and JwtUserController which is responsible for endpoints.

##### JwtUser
```java
@Entity
public class JwtUser {
	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;

	private String username;
	private String password;

	public Long getId() {
		return id;
	}

	public String getUsername() {
		return username;
	}

	public void setUsername(String username) {
		this.username = username;
	}

	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}
}
``` 

##### JwtUserRepository

```java
public interface JwtUserRepository extends JpaRepository<JwtUser, Long> {
	JwtUser findByUsername(String username);
}
```

##### JwtUserController
```java
public class JwtUserController {
	private JwtUserRepository jwtUserRepository;
	private BCryptPasswordEncoder bCryptPasswordEncoder;

	public JwtUserController(
			JwtUserRepository jwtUserRepository,
			BCryptPasswordEncoder bCryptPasswordEncoder) {
		this.jwtUserRepository = jwtUserRepository;
		this.bCryptPasswordEncoder = bCryptPasswordEncoder;
	}

	@PostMapping("/sign-up")
	public void signUp(@RequestBody JwtUserController user) {
		user.setPassword(bCryptPasswordEncoder.encode(user.getPassword()));
		jwtUserRepository.save(user);
	}
}
```
BCryptPasswordEncoder requires spring-boot-starter-security, when that starter is on the classpath, our application is secured by default. But if we run TaskControllerTests, all of them fail with message:
##### console log
```text
java.lang.AssertionError: 1 expectation failed.
Expected status code <200> but was <401>.
```
which is expected behaviour.  
We can use default user: 'user' and password printed at INFO level when application starts to authenticate:

##### console log
```text
Using generated security password: 8775a7ac-8ac2-45ca-9945-e18aa518c97c
```
 but because we want to use the JSON Web Token, we skip that and go straight to the implementation.  

### 3.2. Authentication and authorization
In the new package jlabsblog.jwt.security we add the class which implements UserDetailsService, it allows to load user data into the framework.
##### JwtUserDetailsServiceImpl
```java
public class JwtUserDetailsServiceImpl implements UserDetailsService {
  private JwtUserRepository jwtUserRepository;

  public JwtUserDetailsServiceImpl(JwtUserRepository applicationUserRepository) {
    this.jwtUserRepository = applicationUserRepository;
  }

  @Override
  public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
    JwtUser applicationUser = jwtUserRepository.findByUsername(username);
    if (applicationUser == null) {
      throw new UsernameNotFoundException(username);
    }
    return new User(applicationUser.getUsername(), applicationUser.getPassword(), emptyList());
  }
}
```
There are also three other classes: JwtWebSecurity, JwtAuthenticationFilter and JwtAuthorizationFilter.  
The most important part of JwtWebSecurity is configure method: 
##### JwtWebSecurity
```java
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.cors()
				.and()
				.csrf()
				.disable()
				.authorizeRequests()
				.antMatchers(HttpMethod.POST, SecurityConstants.SIGN_UP_URL)
				.permitAll()
				.anyRequest()
				.authenticated()
				.and()
				.addFilter(new JwtAuthenticationFilter(authenticationManager()))
				.addFilter(new JwtAuthorizationFilter(authenticationManager()))
				.sessionManagement()
				.sessionCreationPolicy(SessionCreationPolicy.STATELESS);
	}
```
It configures public and secured endpoints, CORS and a custom security filter.

##### JwtAuthenticationFilter
in that class, method attemptAuthentication tries to authenticate the user:
```java
public Authentication attemptAuthentication(
      HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
    try {
      JwtUser credentials = new ObjectMapper().readValue(request.getInputStream(), JwtUser.class);
      return authenticationManager.authenticate(
          new UsernamePasswordAuthenticationToken(
              credentials.getUsername(), credentials.getPassword(), new ArrayList<>()));
    } catch (IOException e) {
      throw new RuntimeException();
    }
  }
```
and if the user was authenticated, successfulAuthentication method returns token:
```java
  protected void successfulAuthentication(
      HttpServletRequest request,
      HttpServletResponse response,
      FilterChain chain,
      Authentication authentication)
      throws IOException, ServletException {
    String token =
        Jwts.builder()
            .setSubject(authentication.getName())
            .claim(
                "authorities",
                authentication.getAuthorities().stream()
                    .map(GrantedAuthority::getAuthority)
                    .collect(Collectors.toList()))
            .setIssuedAt(new Date(System.currentTimeMillis()))
            .setExpiration(new Date(System.currentTimeMillis() + JwtSecurityConstants.EXPIRATION_TIME))
            .signWith(SignatureAlgorithm.HS512, JwtSecurityConstants.SECRET.getBytes())
            .compact();
    response.addHeader(JwtSecurityConstants.HEADER_STRING, JwtSecurityConstants.TOKEN_PREFIX + token);
  }
```

##### JwtAuthorizationFilter
doFilterInternal overrides BasicAuthenticationFilter, thanks to that, Spring Boot replaces in the filter chain default implementation with our own.

```java
 protected void doFilterInternal(
      HttpServletRequest request, HttpServletResponse response, FilterChain chain)
      throws IOException, ServletException {
    String header = request.getHeader(JwtSecurityConstants.HEADER_STRING);

    if (header == null || !header.startsWith(JwtSecurityConstants.TOKEN_PREFIX)) {
      chain.doFilter(request, response);
      return;
    }

    UsernamePasswordAuthenticationToken authenticationToken = getAuthentication(request);
    SecurityContextHolder.getContext().setAuthentication(authenticationToken);
    chain.doFilter(request, response);
  }
```
getAuthentication reads and validates a JWT. When it's valid, sets the user in the Security Context and allows the request to proceed. 
```java
private UsernamePasswordAuthenticationToken getAuthentication(HttpServletRequest request) {
    String token = request.getHeader(JwtSecurityConstants.HEADER_STRING);
    if (token != null) {
      String user =
          Jwts.parser()
              .setSigningKey(JwtSecurityConstants.SECRET.getBytes())
              .parseClaimsJws(token.replace(JwtSecurityConstants.TOKEN_PREFIX, ""))
              .getBody()
              .getSubject();
      if (user != null) {
        return new UsernamePasswordAuthenticationToken(user, null, new ArrayList<>());
      }
      return null;
    }
    return null;
```

We also need the small class with constants:
##### JwtSecurityConstants
```java
public class JwtSecurityConstants {
	public static final String SECRET = "SecretKeyToGenJWTs";
	public static final long EXPIRATION_TIME = 86_400_000;
	public static final String TOKEN_PREFIX = "Bearer ";
	public static final String HEADER_STRING = "Authorization";
	public static final String SIGN_UP_URL = "/users/sign-up";
}
```
At the end, two missing parts to make a successful build:
##### App:
```java
    @Bean
    public BCryptPasswordEncoder bCryptPasswordEncoder() {
        return new BCryptPasswordEncoder();
    }
```
##### build.gradle
```groovy
implementation "io.jsonwebtoken:jjwt:0.9.1"
```
Now we can run the application and if everyting is working as expected we can go to the tests.
There we will check how application handles authorization.

##  4. Update the tests
To use the JWT, we add @BeforeClass method:
```java
  @BeforeClass
  public void authorization() {
    JwtUser user = new JwtUser();
    user.setUsername("username");
    user.setPassword("password");

    given()
        .basePath("/users/sign-up")
        .contentType("application/json")
        .body(user)
        .when()
        .post()
        .then()
        .statusCode(200);

    String token =
        given()
            .basePath("/login")
            .contentType("application/json")
            .body(user)
            .when()
            .post()
            .then()
            .statusCode(200)
            .extract()
            .header(JwtSecurityConstants.HEADER_STRING);

    specification =
        new RequestSpecBuilder()
            .addHeader(JwtSecurityConstants.HEADER_STRING, token)
            .setBasePath("/tasks")
            .build();
  }
```
Together with BeforeClass, we swap basePath("/tasks") with spec(specification).  
Tests execution returns:

##### console log
```text

===============================================
Default Suite
Total tests run: 3, Failures: 0, Skips: 0
===============================================

```

## 5. Logs and properties of application
The application works as expected, of course is very simple and if something is wrong is relatively easy to figure out the cause.
To deal with more complicated scenarios, we should take care about logs from the application and from the tests. 
Also we should have a mean to adjust the test, it is shown with reading application properties during the tests.  

### 5.1. Logs
In resources we add application.properties with two lines:
##### application.properties
```properties
server.port=8099
logging.level.root=OFF
``` 
All logs are now switched off, but it doesn't apply for logs from tests.  
To change that we have to add file logback.xml into tests' resources:
##### logback.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <include resource="org/springframework/boot/logging/logback/base.xml" />
  <logger name="org.springframework.core " level="ERROR" />
  <logger name="org.springframework.beans" level="ERROR" />
  <logger name="org.springframework.context" level="ERROR" />
  <logger name="org.springframework.transaction" level="ERROR" />
  <logger name="org.springframework.web" level="ERROR" />
  <logger name="org.springframework.test" level="ERROR" />
  <logger name="org.hibernate" level="ERROR" />
</configuration>
```
We got rid of all logs, and now we can easily control them with RequestSpecBuilder:
```java
addFilter(new RequestLoggingFilter(LogDetail.ALL)
addFilter(new ResponseLoggingFilter(LogDetail.ALL)
```
these two lines provides logs with requests and responses, in case there is any problem, we have place, where we can start to work out:
##### console log
```text
Request method:	POST
Request URI:	http://localhost:8099/tasks
Proxy:			<none>
Request params:	<none>
Query params:	<none>
Form params:	<none>
Path params:	<none>
Headers:		Authorization=Bearer eyJhbGciOiJIUzUxMiJ9.eyJzdWIiOiJ1c2VybmFtZSIsImF1dGhvcml0aWVzIjpbXSwiaWF0IjoxNTY5MzI4OTk5LCJleHAiOjE1Njk0MTUzOTl9.9oquWLKG5bAXvktUrsUcFuOh3iQKsIQErVffVzXMhTSGoW-9jNuRdrna5EofMr05_LImukp83Rk0RayPX7e1_g
				Accept=*/*
				Content-Type=application/json; charset=UTF-8
Cookies:		<none>
Multiparts:		<none>
Body:
{
    "id": null,
    "description": "initialValue"
}
HTTP/1.1 200 
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Pragma: no-cache
Expires: 0
X-Frame-Options: DENY
Content-Length: 0
Date: Tue, 24 Sep 2019 12:43:19 GMT
```

### 5.2. Application properties
In application.properties we changed default port.  
To read the value of server.port:
- test class has to have annotation @TestPropertySource
- test class has to extend AbstractTestNGSpringContextTests
- port can be assigned with @Value

## 6. Conclusion
Testing an application with rest-assured library is efficient, and with use of RequestLoggingFilter any inaccuracy is conveniently track down.
In our application the most complicated in implementation was JWT, but adding token to tests, required only a few minor changes.
Spring Boot, gradle and TestNG offer a lot more, but we have a great place to start.

### Useful links:
[Link to the repository](https://github.com/emiltroczynski/j-labs-blog-springboot-restassured-jwt)  
[Spring Boot documentation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)    
[Gradle plugin](https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-running-your-application.html#using-boot-running-with-the-gradle-plugin)  
[Secure endpoint with JWT library](https://auth0.com/blog/implementing-jwt-authentication-on-spring-boot/#Securing-RESTful-APIs-with-JWTs)  
[Gradle](https://gradle.org/)  
[TestNG](https://testng.org)
[REST assured](http://rest-assured.io/)