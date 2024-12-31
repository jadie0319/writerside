# mybatis전환

Spring 3.0.5, JDK 7 을 사용하는 프로젝트의 버전을 올려야 하는 일이 생겼다. Spring 버전업과 관련하여 조사를 하던 중
Spring 4 부터 JDK 8 이 완벽하게 동작한 다는 것을 알게 되어 최소 Spring 4 이상, JDK 8  이상으로 버전업을 목표로 조사를 진행했다.
조사 중 단순히 Spring 라이브러리 의존성(pom.xml) 을 변경하는 것 만으로는 작업을 할 수 없다는 것을 알게 되었다.
왜냐하면 [Spring 4 부터는 Ibatis 를 지원하지 않기 때문이다.](https://blog.mybatis.org/2015/11/spring-4-got-you-down-with-no-ibatis.html) 
그래서 Spring, JDK 버전을 올리기 전에 Ibatis 를 MyBatis 로 변경하기 위한 작업을 진행하기로 했다.

## mybatis 의존성 추가 {id="mybatis_1"}

Spring 3.0.5 버전업 기준이다.

```xml
<!-- MyBatis -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>3.5.9</version>
        </dependency>

        <!-- MyBatis-Spring -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis-spring</artifactId>
            <version>1.3.3</version>
        </dependency>
```

의존성을 추가하고 mybatis 설정까지 했지만 실행이 안되는 경우엔 라이브러리 버전도 의심을 해봐야 한다.
나는 아래 라이브러리 버전과의 호환성 문제로 실행이 되지 않았다.

```xml
<dependency>
            <groupId>commons-dbcp</groupId>
            <artifactId>commons-dbcp</artifactId>
            <version>1.4</version>     <!-- 1.3 -> 1.4 로 UP -->
        </dependency>
        <dependency>
            <groupId>commons-pool</groupId>
            <artifactId>commons-pool</artifactId>
            <version>1.6</version>  <!-- 1.5.2 -> 1.6 으로 UP -->
        </dependency>
```

## mybatis 설정

mybatis 설정 파일을 만들고 Spring 에서 mybatis 설정 파일을 읽을 수 있게 web.xml 에 설정해준다.
설정에 대한 자세한 내용은 생략한다. 요점은 `SqlSessionTemplate` 객체를 Spring Bean 으로 등록하는 것이다.

```xml

<!-- web.xml -->

<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
        classpath*:resource/mybatis/context-sqlSession.xml
    </param-value>
</context-param>

<!-- context-sqlSession.xml -->
        <?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">

<bean id="sessionMasterTemplate" class="org.mybatis.spring.SqlSessionTemplate">
    <constructor-arg ref="sessionMasterFactory"/>
</bean>
<bean id="sessionSlaveTemplate" class="org.mybatis.spring.SqlSessionTemplate">
    <constructor-arg ref="sessionSlaveFactory"/>
</bean>

<bean id="sessionMasterFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSourceMaster"/>
    <property name="configLocation" value="classpath:resource/mybatis/mybatis-config.xml" />
    <property name="mapperLocations" value="classpath:resource/mybatis/mapper/**/*Mapper.xml" />
</bean>

<bean id="sessionSlaveFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSourceSlave"/>
    <property name="configLocation" value="classpath:resource/mybatis/mybatis-config.xml"/>
    <property name="mapperLocations" value="classpath:resource/mybatis/mapper/**/*Mapper.xml" />
</bean>

</beans>

<!-- mybatis-config.xml -->

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>

    <!-- DTO 의 카멜케이스 방식과 DB의 스네이크 표기 방식 연동을 위해 설정  -->
    <settings>
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>

    <typeAliases>
        <typeAlias alias="hashMap" type="java.util.HashMap"/>
    </typeAliases>

</configuration>
```

## mybatis 실행 {id="mybatis_2"}

Ibatis 에선 `SqlMapClient` Bean 객체를 통해 쿼리를 실행했지만 mybatis 에서는 `SqlSession` Bean 객체를 통해 쿼리를 실행한다.

```java
@Resource(name = "sessionMasterTemplate")
protected SqlSession masterSession;

@Resource(name = "sessionSlaveTemplate")
protected SqlSession slaveSession;

public String insertItemTest(Map<String, Object> map) throws Exception {
    masterSession.insert("MyRoadMapItemTestMapper.insertItemTest", map);
    return dataMap.get("itemTestUid").toString();
}
    
```

SqlSession 을 주입받고 실행시키고자 하는 쿼리가 있는 mapper의 id 를 인자로 넘겨 쿼리를 실행하면 된다.

## Ibatis vs MyBatis

ibatis 와 mybatis 는 문법간 약간의 차이가 있다. 여기서 모든 차이점을 얘기하진 못하겠지만 경험했던 차이에 대해서 기록하고자 한다.

<table>
<tr><td>aa</td><td>aa</td></tr>
<tr>
<td>

- 선언 
  - DOCTYPE 변경 
  - sqlMap -> mapper 변경
- Select문 
  - parameterClass -> parameterType 
  - resultClass -> resultType 
  - remapResults 사라짐
- 변수
  - `$test$` -> `${test}` 
  - `#test#` -> `#{test}`

</td>
<td>
<p>

```plain text
<!-- Ibatis Sample-->

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE sqlMap PUBLIC "-//iBATIS.com//DTD SQL Map 2.0//EN" "http://ibatis.apache.org/dtd/sql-map-2.dtd">

<sqlMap namespace="parentDAO">
    <select id="selectList" parameterClass="hashMap" resultClass=“hashMap" remapResults="true">
        …
        where ste.f_exam_id in ($test_id$) and ste.f_user_cd = #student_no#
        …
    </select>
</sqlMap>

<!-- MyBatis 변환 -->

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN“ 
		"http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="service.impl.ParentMapper">
    <select id="selectList" parameterType="hashMap" resultType= “hashMap" >
        …
        where ste.f_exam_id in (${test_id}) and ste.f_user_cd = #{student_no}
        …
    </select>
</mapper>
```

</p>
</td>
</tr>
<tr>
<td>

- is**형식 메서드는 삭제됨
    - isNotEmpty -> if test
    - iterate -> foreach
    - isEmpty -> if test
- 반복문
    - iterate -> foreach
    - property -> collection
    - prepend + open -> open
    - conjunction -> separator
- selectKey
    - resultClass -> resultType

</td>
<td>
<p>

```plain text
<!-- Ibatis Sample-->

<isNotEmpty property="status">
    AND ste.f_status = #status#
</isNotEmpty>

<iterate property="problemId" prepend="where p.id in" open="(" close=")" conjunction=",">
    #problemId[]#
</iterate>

<isEmpty property="serviceId">
    <isEmpty property="seq">
        and (service_id is null or service_id = ‘’)
    </isEmpty>
</isEmpty>

<selectKey keyProperty="stuId" resultClass="long">
    SELECT LAST_INSERT_ID()
</selectKey>


<!-- MyBatis 변환 -->

<if test=" status != null and !status.equals('') ">
    AND ste.f_status = #{status}
</if>

<foreach collection="problemIdList" item="problemId" open=" where p.id in (" separator="," close=")" >
    #{problemId}
</foreach>

<if test = " serviceId == null or serviceId.equals('') ">
    <if test = " seq == null or seq.equals('') ">
        and (service_id is null or service_id = ‘’)
    </if>
</if>

<selectKey keyProperty="stuId" resultType="string">
    SELECT LAST_INSERT_ID()
</selectKey>

```

</p>
</td>
</tr>
</table>

### SelectKey

Ibatis, MyBatis 둘다 SelectKey 를 사용할 수 있으나 차이점이 있다. 차이점을 인지하고 있지 않은 채로 ibatis 를 mybatis 로 전환하면 
잘못구현된 로직을 찾기 어려울 수 있으니 기억하고 있으면 좋을 것 같다.

Ibatis 에서 아래와 같이 selectKey 를 사용하면 쿼리의 결과 값으로 **LAST_INSERT_ID** 가 반환된다.

```XML
<insert id="insertItem" parameterClass="hashMap">
    INSERT INTO items (
        
    )
    <selectKey keyProperty="itemId" resultClass="long">
        SELECT LAST_INSERT_ID()
    </selectKey>
</insert>
```
```JAVA
public String insertItem(Map<String,Object> dataMap) throws Exception {
	return String.valueOf(sqlMapClient.insert(“itemTestDAO.insertItem", dataMap));
}
```
위 자바 코드처럼 이를 사용하는 코드에서는 쿼리의 결과 값을 받아서 그대로 사용하고 있다.

그런데 위와 같은 쿼리를 mybatis 로 전환작업을 할 때, 문법만 변경하고 호출 부분 코드를 변경하지 않으면 어떨게 될까?
MyBatis 에서는 insert 쿼리의 결과 값으로 **insert 한 레코드 값을 반환**한다. 그리고 selectKey 의 값은 파라미터로 받은 Map 에 저장한다.
위의 쿼리의 경우에는 `LAST_INSERT_ID()` 의 값은 파라미터로 받은 Map 의 itemId 키에 저장되는 것이다. 그리고  
쿼리의 결과 값은 **insert 한 레코드 수를 반환**한다. 기존과 다른 값을 반환했기 때문에 전체적인 로직에 문제가 발생한다.
그러니 아래와 같은 형태로 클라이언트 코드를 변경해 기존과 같은 값을 반환하게 바꿔야 문제가 발생하지 않는다.

```JAVA
public String insertItem(Map<String,Object> dataMap) throws Exception {
    sqlSession.insert(“itemTestDAO.insertItem", dataMap);    
	return dataMap.get("itemId");
}
```

이렇게 한눈에 보이지 않는 문제는 나중에 발견될 가능성이 높고, 이미 잘못된 데이터가 축적되었을 수도 있다.
그러니 이러한 전환작업을 할 땐 테스트 코드를 작성 또는 확실하게 입력값, 출력값을 비교하며 전환작업을 하는 것을 추천한다.




