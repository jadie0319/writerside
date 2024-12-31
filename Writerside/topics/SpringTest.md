# Spring Test


## spring-test-3.0.5 (Spring 3.0.5 기준)
- Spring 3.0.5 에서 Ibatis -> Mybatis 로 변경하는 작업을 하다 알게된 내용을 기록
- `@ActiveProfiles` 어노테이션이 없음
- `System.setProperty` 를 사용해 profile 지정 가능

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(
        locations = {
                "classpath:sigong/resource/spring/test-config.xml",
                "classpath:sigong/resource/spring/context-datasource.xml"
        }
)
public class RoadmapTest {
    @BeforeClass
    public static void setUp() {
        System.setProperty("test-component-scan", "dao.myroadmap");
        System.setProperty("spring.profiles.active", "local");
    }
}
```

## spring-test-4.3.7 (Spring Boot 1.5.2)
- 기능 리팩터링 전에 테스트 코드를 만들어 리팩터링 후에도 기능이 정상적으로 동작하는지 확인하기 위해 구현 
- `@Sql` 원하는 쿼리를 sql 파일로 만들어 테스트 실행시 쿼리를 함께 실행할 수 있다.
- `@ActiveProfile` 로 Profile 지정 가능하다.
- `@ContextConfiguration`, `@TestPropertySource` 를 사용하면 특정 클래스, xml, properties 등 
  테스트에 필요한 파일을 주입해 원하는 테스트 환경을 만들어 줄 수 있다.

```java
@Documented
@Inherited
@Retention(RUNTIME)
@Target({TYPE, METHOD})
@Sql(scripts = {
        "classpath:test-h2-auth-member-schema.sql"
},
        config = @SqlConfig(encoding = "UTF-8"))
public @interface MemberH2InitSql {
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@ActiveProfiles("test-member-h2")
@ContextConfiguration(classes = {MemberH2JpaConfig.class})
@TestPropertySource(locations = "classpath:/application-test-member-h2.properties")
@MemberH2InitSql
public @interface MemberH2JpaIntTest {
    String[] componentScanPackages() default {};
}

@Configuration
@Import({MemberTestMasterAuthDatabaseConfig.class, MemberTestJpaConfiguration.class})
@ComponentScan(basePackages = "com.sigongedu.oauth2.member.*",
        useDefaultFilters = false,
        includeFilters = {
            @ComponentScan.Filter(
                    type = FilterType.REGEX,
                    pattern = ".*GivenHelper.*")
        },
        excludeFilters = {
                @ComponentScan.Filter(type = FilterType.ANNOTATION,
                        classes = {Configuration.class})
        })
public class MemberH2JpaConfig {
    @Bean
    public static PropertySourcesPlaceholderConfigurer placeholderConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }
}
``` 

