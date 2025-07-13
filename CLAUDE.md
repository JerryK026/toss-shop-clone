# Spring Boot 프로젝트 개발 가이드

Spring Boot 공식 프로젝트의 아키텍처 패턴과 개발 관례를 기반으로 한 포괄적인 개발 가이드입니다.

## 1. 프로젝트 개요

### 아키텍처 원칙
- **모듈화**: 기능별로 명확히 분리된 멀티모듈 구조
- **Auto-Configuration**: 조건부 설정을 통한 유연한 구성
- **Starter 패턴**: 의존성 묶음을 통한 개발 생산성 향상
- **테스트 우선**: 포괄적인 테스트 전략으로 품질 보장

### 핵심 설계 원칙
```java
// Spring Boot Configuration 패턴
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { 
    @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
    @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) 
})
public @interface SpringBootApplication {
    // ...
}
```

## 2. 디렉토리 구조

### 표준 멀티모듈 레이아웃
```
project-root/
├── build.gradle                     # 루트 빌드 설정
├── settings.gradle                  # 모듈 정의
├── gradle.properties               # 버전 관리
├── buildSrc/                       # 커스텀 빌드 로직
│   ├── build.gradle
│   └── src/main/groovy/
├── project-core/                   # 핵심 모듈
│   ├── build.gradle
│   └── src/
│       ├── main/java/
│       ├── main/resources/
│       ├── test/java/
│       └── testFixtures/java/
├── project-starters/              # Starter 모듈들
│   ├── project-starter-web/
│   └── project-starter-data/
├── project-tests/                 # 테스트 모듈들
│   ├── project-integration-tests/
│   └── project-smoke-tests/
└── project-tools/                 # 도구 모듈들
    ├── project-gradle-plugin/
    └── project-maven-plugin/
```

### 패키지 구조
```java
// 기본 패키지 구조
com.example.project/
├── ProjectApplication.java         # 메인 클래스
├── config/                        # 설정 클래스들
│   ├── WebConfig.java
│   └── DatabaseConfig.java
├── domain/                        # 도메인 객체
│   ├── User.java
│   └── Order.java
├── repository/                    # 데이터 액세스
│   ├── UserRepository.java
│   └── OrderRepository.java
├── service/                       # 비즈니스 로직
│   ├── UserService.java
│   └── OrderService.java
├── web/                          # 웹 계층
│   ├── UserController.java
│   └── OrderController.java
└── util/                         # 유틸리티
    └── DateUtils.java
```

## 3. 코딩 규칙

### Java 코딩 스타일

#### 1. 네이밍 컨벤션
```java
// 클래스명: PascalCase
public class UserServiceImpl implements UserService {
    
    // 상수: UPPER_SNAKE_CASE
    private static final String DEFAULT_ENCODING = "UTF-8";
    
    // 변수/메서드: camelCase
    private final UserRepository userRepository;
    
    public User findUserById(Long userId) {
        return userRepository.findById(userId);
    }
}
```

#### 2. 메서드 시그니처 패턴
```java
// Builder 패턴 활용
public class ApiResponse<T> {
    
    private final boolean success;
    private final T data;
    private final String message;
    
    private ApiResponse(Builder<T> builder) {
        this.success = builder.success;
        this.data = builder.data;
        this.message = builder.message;
    }
    
    public static <T> Builder<T> builder() {
        return new Builder<>();
    }
    
    public static class Builder<T> {
        private boolean success = true;
        private T data;
        private String message;
        
        public Builder<T> success(boolean success) {
            this.success = success;
            return this;
        }
        
        public ApiResponse<T> build() {
            return new ApiResponse<>(this);
        }
    }
}
```

#### 3. 애노테이션 사용 패턴
```java
// Configuration 클래스
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({DataSource.class, JdbcTemplate.class})
@ConditionalOnProperty(prefix = "spring.datasource", name = "url")
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        return createDataSource(properties);
    }
}
```

### 코드 포맷팅 규칙
```gradle
// build.gradle에서 Spring Java Format 적용
plugins {
    id "io.spring.javaformat" version "${javaFormatVersion}"
    id "checkstyle"
}

checkstyle {
    toolVersion = "${checkstyleToolVersion}"
    configFile = file("${rootDir}/src/checkstyle/checkstyle.xml")
}
```

## 4. 테스트 가이드라인

### 테스트 아키텍처

#### 1. 테스트 분류
```java
// 단위 테스트
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @InjectMocks
    private UserServiceImpl userService;
    
    @Test
    void shouldFindUserById() {
        // given
        Long userId = 1L;
        User expectedUser = new User(userId, "john");
        given(userRepository.findById(userId)).willReturn(Optional.of(expectedUser));
        
        // when
        User actualUser = userService.findById(userId);
        
        // then
        assertThat(actualUser).isEqualTo(expectedUser);
    }
}
```

#### 2. 통합 테스트
```java
@SpringBootTest
@TestPropertySource(properties = {
    "spring.datasource.url=jdbc:h2:mem:testdb",
    "spring.jpa.hibernate.ddl-auto=create-drop"
})
class UserIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    void shouldCreateUser() {
        // given
        User newUser = new User("jane");
        
        // when
        ResponseEntity<User> response = restTemplate.postForEntity(
            "/api/users", newUser, User.class);
        
        // then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getName()).isEqualTo("jane");
    }
}
```

#### 3. 웹 계층 테스트
```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void shouldReturnUser() throws Exception {
        // given
        User user = new User(1L, "john");
        given(userService.findById(1L)).willReturn(user);
        
        // when & then
        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("john"));
    }
}
```

### 테스트 명명 규칙
```java
// 패턴: should[ExpectedBehavior]When[StateUnderTest]
@Test
void shouldReturnNotFoundWhenUserDoesNotExist() {
    // 테스트 구현
}

@Test
void shouldThrowExceptionWhenInvalidDataProvided() {
    // 테스트 구현
}
```

## 5. 빌드 및 도구

### Gradle 설정

#### 1. 루트 build.gradle
```gradle
plugins {
    id "base"
    id "org.springframework.boot.conventions"
}

allprojects {
    group = "com.example"
    version = "1.0.0-SNAPSHOT"
}

subprojects {
    apply plugin: "java-library"
    apply plugin: "org.springframework.boot.conventions"
    
    repositories {
        mavenCentral()
    }
    
    dependencies {
        implementation platform("org.springframework.boot:spring-boot-dependencies:${springBootVersion}")
        testImplementation("org.springframework.boot:spring-boot-starter-test")
    }
}
```

#### 2. 모듈별 build.gradle
```gradle
plugins {
    id "org.springframework.boot.starter"
}

description = "Project Web Starter"

dependencies {
    api(project(":project-core"))
    api("org.springframework.boot:spring-boot-starter-web")
    api("org.springframework.boot:spring-boot-starter-validation")
    
    optional("org.springframework.boot:spring-boot-starter-security")
    
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

#### 3. gradle.properties
```properties
# 버전 관리
version=1.0.0-SNAPSHOT
springBootVersion=3.2.0
springFrameworkVersion=6.1.0

# 빌드 최적화
org.gradle.caching=true
org.gradle.parallel=true
org.gradle.jvmargs=-Xmx2g -Dfile.encoding=UTF-8

# 의존성 버전
junitJupiterVersion=5.10.0
assertjVersion=3.24.2
mockitoVersion=5.5.0
```

### 품질 관리 도구

#### 1. Checkstyle 설정
```xml
<!-- checkstyle.xml -->
<module name="Checker">
    <module name="io.spring.javaformat.checkstyle.SpringChecks" />
    <module name="TreeWalker">
        <module name="ImportControl">
            <property name="file" value="${config_loc}/import-control.xml"/>
        </module>
        <module name="RegexpSinglelineJava">
            <property name="format" value="org\.junit\.Assert"/>
            <property name="message" value="Please use AssertJ imports."/>
        </module>
    </module>
</module>
```

#### 2. 코드 품질 검사 자동화
```gradle
// 빌드 시 품질 검사 포함
check.dependsOn checkstyleMain, checkstyleTest

// 테스트 실행
test {
    useJUnitPlatform()
    testLogging {
        events "passed", "skipped", "failed"
    }
}
```

## 6. 개발 워크플로

### 코드 리뷰 체크리스트
- [ ] 코드 스타일이 프로젝트 규칙을 준수하는가?
- [ ] 테스트 커버리지가 충분한가?
- [ ] API 문서가 업데이트되었는가?
- [ ] 성능에 영향을 주는 변경사항이 있는가?
- [ ] 보안 취약점이 없는가?

### Git 커밋 메시지 규칙
```
type(scope): subject

body

footer
```

예시:
```
feat(user): Add user profile management

Implement user profile CRUD operations with validation.
Add integration tests for user controller.

Closes #123
```

## 7. 의존성 관리

### 의존성 분류 기준

#### 1. API vs Implementation
```gradle
dependencies {
    // 외부로 노출되는 의존성
    api("org.springframework:spring-context")
    
    // 내부 구현에만 사용
    implementation("org.springframework:spring-web")
    
    // 선택적 의존성
    optional("org.springframework.boot:spring-boot-starter-security")
    
    // 컴파일 시에만 필요
    compileOnly("org.springframework.boot:spring-boot-configuration-processor")
}
```

#### 2. BOM 활용
```gradle
dependencies {
    // BOM을 통한 버전 관리
    implementation platform("org.springframework.boot:spring-boot-dependencies:${springBootVersion}")
    implementation platform("org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}")
    
    // 버전 명시 없이 사용
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.cloud:spring-cloud-starter-gateway")
}
```

### 버전 관리 전략
```gradle
// gradle/libs.versions.toml
[versions]
spring-boot = "3.2.0"
spring-framework = "6.1.0"
junit = "5.10.0"

[libraries]
spring-boot-starter-web = { group = "org.springframework.boot", name = "spring-boot-starter-web", version.ref = "spring-boot" }
junit-jupiter = { group = "org.junit.jupiter", name = "junit-jupiter", version.ref = "junit" }

[bundles]
spring-boot = ["spring-boot-starter-web", "spring-boot-starter-actuator"]
testing = ["junit-jupiter", "assertj-core", "mockito-core"]
```

## 8. 성능 및 보안

### 성능 모범 사례

#### 1. 지연 로딩 최적화
```java
@Configuration(proxyBeanMethods = false)
public class PerformanceConfig {
    
    // 지연 초기화 활성화
    @Bean
    @Lazy
    public ExpensiveService expensiveService() {
        return new ExpensiveService();
    }
}
```

#### 2. 캐싱 전략
```java
@Service
public class UserService {
    
    @Cacheable(value = "users", key = "#userId")
    public User findById(Long userId) {
        return userRepository.findById(userId);
    }
    
    @CacheEvict(value = "users", key = "#user.id")
    public User update(User user) {
        return userRepository.save(user);
    }
}
```

### 보안 설정

#### 1. 기본 보안 구성
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(OAuth2ResourceServerConfigurer::jwt)
            .csrf(csrf -> csrf.disable())
            .build();
    }
}
```

#### 2. 민감 정보 관리
```yaml
# application.yml
spring:
  datasource:
    url: ${DB_URL:jdbc:h2:mem:testdb}
    username: ${DB_USERNAME:sa}
    password: ${DB_PASSWORD:}

# 프로덕션에서는 환경 변수 사용
management:
  endpoint:
    env:
      show-values: when-authorized
```

### 모니터링 설정
```java
@Configuration
public class MonitoringConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config()
            .commonTags("application", "my-app");
    }
    
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}
```

## 개발 도구 설정

### IDE 설정 (IntelliJ IDEA)
```xml
<!-- .idea/codeStyles/Project.xml -->
<code_scheme name="Project" version="173">
  <option name="RIGHT_MARGIN" value="120" />
  <JavaCodeStyleSettings>
    <option name="CLASS_COUNT_TO_USE_IMPORT_ON_DEMAND" value="999" />
    <option name="NAMES_COUNT_TO_USE_IMPORT_ON_DEMAND" value="999" />
  </JavaCodeStyleSettings>
</code_scheme>
```

### 지속적 통합 설정
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
    - name: Run tests
      run: ./gradlew build
    - name: Publish test results
      uses: EnricoMi/publish-unit-test-result-action@v2
      if: always()
      with:
        files: '**/build/test-results/**/*.xml'
```

이 가이드는 Spring Boot 프로젝트의 검증된 패턴과 모범 사례를 기반으로 작성되었습니다. 프로젝트의 특성에 맞게 조정하여 사용하세요.

## 프로젝트 가이드

이 프로젝트는 코틀린을 사용하며, 코틀린스러운 테스트와 소스코드를 작성할 수 있도록 가이드합니다.