# SSM(Spring+SpringMVC+MyBatis整合)

## 整合思路

```sql
1、SqlSessionFactory对象应该放到spring容器中作为单例存在。
2、传统dao的开发方式中，应该从spring容器中获得sqlsession对象。
3、Mapper代理形式中，应该从spring容器中直接获得mapper的代理对象。
4、数据库的连接以及数据库连接池事务管理都交给spring容器来完成.
```

 1.创建项目，导入jar(spring(15个)＋hibernate(11个)+上传包(2个)………….)

 2.导入页面资源jsp放入到WEB-INF/jsp目录下

3.以分层思想进行分包

![1.项目分层思想分包目录图](../ssh框架整合导图/1.项目分层思想分包目录图.png)

4.实体类User.java

```java
public class User {
    private Integer user_id;//用户编号
    private String user_name;//用户名称
    private String user_pwd;//用户密码
    private String user_headimg;//用户头像地址
    set/get......  
}
```

## 第一章：以传统dao层实现方式进行整合

#### 1.1 dao层相关：

#####     	1.1.1 dao层接口及映射类和实现类

```java
UserDao.java
public interface UserDao {
    Integer saveUser(User user);
    User queryUserByUserNameAndPwd(User user);
}

UserDao.xml
<!DOCTYPE mapper  PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="/test">
     <insert id="saveUser" parameterType="com.sky.domain.User">
         <selectKey keyProperty="user_id" resultType="java.lang.Integer" order="AFTER">
             SELECT LAST_INSERT_ID()
         </selectKey>
       insert into user (user_name,user_pwd,user_headimg) 
                  values(#{user_name},#{user_pwd},#{user_headimg})
     </insert>
     <select id="queryUserByUserNameAndPwd"  parameterType="user"
             resultType="com.sky.domain.User">
        select * from user where user_name = #{user_name} and user_pwd = #{user_pwd}
     </select>
</mapper>

UserDaoImpl.java
public class UserDaoImpl extends SqlSessionDaoSupport
        implements UserDao {
    @Override
    public Integer saveUser(User user) {
       int rowCount= getSqlSession().insert("saveUser",user);
        return rowCount;
    }
    @Override
    public User queryUserByUserNameAndPwd(User user) {
       User user1 =  getSqlSession().selectOne("queryUserByUserNameAndPwd",
                user);
        return user1;
    }
}
```

#####    	1.1.2 配置文件(applicationContext_dao.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
	   http://www.springframework.org/schema/context/spring-context-4.3.xsd">

    <!--1.加载数据库的属性配置文件 -->
    <context:property-placeholder location="classpath:dbinfo.properties">
    </context:property-placeholder>
  <!-- 配置数据库连接池 -->
    <!-- 方式一：使用DBCP数据库连接池 -->
   <!--<bean name="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
             <property name="driverClassName" value="${jdbc.driverClass}"></property>
             <property name="url" value="${jdbc.jdbcUrl}"></property>
             <property name="username" value="${jdbc.user}"></property>
             <property name="password" value="${jdbc.password}"></property>
   </bean>-->
    <!-- 方式二：使用C3P0数据库连接池 -->
   <!--<bean name="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
             <property name="driverClass" value="${jdbc.driverClass}"></property>
             <property name="jdbcUrl" value="${jdbc.jdbcUrl}"></property>
             <property name="user" value="${jdbc.user}"></property>
             <property name="password" value="${jdbc.password}"></property>
   </bean>-->
    <!-- 方式三：使用Druid数据库连接池 -->
	<bean name="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
              <property name="driverClassName" value="${jdbc.driverClass}"></property>
              <property name="url" value="${jdbc.jdbcUrl}"></property>
              <property name="username" value="${jdbc.user}"></property>
              <property name="password" value="${jdbc.password}"></property>
	</bean>
    <!-- 3.配置sqlSessionFactory -->
    <bean name="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="configLocation" value="classpath:SqlMapConfig.xml"></property>
        <property name="dataSource" ref="ds"></property>
    </bean>
    <!-- 4.配置UserDaoImpl -->
    <bean name="userDao" class="com.sky.dao.impl.UserDaoImpl">
        <property name="sqlSessionFactory" ref="sqlSessionFactory"></property>
    </bean>
</beans>
```

#### 1.2 service层相关：

##### 	1.2.1 service接口实现类

```java
UserService.java

public interface UserService {
      void register(User user)throws UserException;
      void loginUser(User user) throws UserException;
}

UserServiceImpl.java

@Transactional
public class UserServiceImpl implements UserService {
    private UserDao userDao;
    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
    @Override
    public void register(User user) throws UserException{
        Integer id = userDao.saveUser(user);
        if (id>0){

        }else{
            throw new  UserException("注册失败");
        }
    }
    @Override
    public void loginUser(User user) throws UserException {
      User user1 = userDao.queryUserByUserNameAndPwd(user);
      if (user1!=null){

      }else{
          throw new UserException("登录失败");
      }
    }
}
```

##### 	1.2.2 配置文件(applicationContext_service.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns="http://www.springframework.org/schema/beans"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
    http://www.springframework.org/schema/context
	http://www.springframework.org/schema/context/spring-context-4.3.xsd
	http://www.springframework.org/schema/tx
	http://www.springframework.org/schema/tx/spring-tx-4.3.xsd
	http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans-4.3.xsd">

    <!--配置事务，让Spring来管理 -->
    <bean name="transactionManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="ds"></property>
    </bean>

    <!--开启事务注解功能 -->
    <tx:annotation-driven transaction-manager="transactionManager"/>

    <bean name="userService" class="com.sky.service.impl.UserServiceImpl">
     <property name="userDao" ref="userDao"></property>
    </bean>
</beans>
```

#### 1.3 controller控制层类：

#####       1.2.1 UserController类

```java
package com.sky.controller;

import com.sky.domain.User;
import com.sky.exception.UserException;
import com.sky.service.UserService;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import javax.annotation.Resource;

@Controller
public class UserController {
    @Resource(name = "userService")
    private UserService userService;
    @RequestMapping("/register.action")
    public String  register(){
        return "register";
    }
    @RequestMapping("/doRegister.action")
    public String  toDoRegister(User user){
        try {
            userService.register(user);
        } catch (UserException e) {
            e.printStackTrace();
            return "fail";
        }
        return "success";
    }
    @RequestMapping("/login.action")
    public String  login(){
        return "login";
    }
    @RequestMapping("/dologin.action")
    public String  tologin(User user){
        try {
            userService.loginUser(user);
        } catch (UserException e) {
            e.printStackTrace();
            return "fail";
        }
        return "success";
    }
}
```

#### 1.4 springmvc的配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns="http://www.springframework.org/schema/beans"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="
	http://www.springframework.org/schema/mvc
	http://www.springframework.org/schema/mvc/spring-mvc-4.3.xsd
	http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
	http://www.springframework.org/schema/context
	http://www.springframework.org/schema/context/spring-context-4.3.xsd">
    <!--1.扫描SpringMVC的注解@Controller  -->
    <context:component-scan base-package="com.sky.controller">
    </context:component-scan>
    <!--2.配置处理器 (RequestMappingHandlerMapping,RequestMappingHandlerAdapter)-->
    <mvc:annotation-driven></mvc:annotation-driven>
    <!--3.放行静态资源-->
    <mvc:default-servlet-handler></mvc:default-servlet-handler>
    <!--配置视图解析器-->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/jsp/"></property>
        <property name="suffix" value=".jsp"></property>
    </bean>
</beans>
```

## 第二章：以Mapper动态代理实现dao层方式进行整合

#### 1.1 dao层相关：

#####     	1.1.1 dao层接口及类

```java
UserMapper.java

public interface UserMapper {
    Integer saveUser(User user);
    User queryUserByUserNameAndPwd(User user);
}
UserMapper.xml

<!DOCTYPE mapper  PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.sky.dao.UserMapper">
    <insert id="saveUser" parameterType="com.sky.domain.User">
        <selectKey keyProperty="user_id" resultType="java.lang.Integer" order="AFTER">
            SELECT LAST_INSERT_ID()
        </selectKey>
        insert into user (user_name,user_pwd,user_headimg) values(#{user_name},#{user_pwd},#{user_headimg})
    </insert>
    <select id="queryUserByUserNameAndPwd" parameterType="user"
            resultType="com.sky.domain.User">
        select * from user where user_name = #{user_name} and user_pwd = #{user_pwd}
     </select>
</mapper>

```

#####    	1.1.2 配置文件(applicationContext_dao.xml)

```xml
dbinfo.properties如下：
      driverClass=com.mysql.jdbc.Driver
      jdbcUrl=jdbc:mysql://localhost:3306/ideaBySSH
      user=root
      password=123456

applicationContext_dao.xml如下：
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
	   http://www.springframework.org/schema/context/spring-context-4.3.xsd">

    <!--1.加载数据库的属性配置文件 -->
    <context:property-placeholder location="classpath:dbinfo.properties">
    </context:property-placeholder>
    <!-- 2.配置c3p0数据源 -->
    <bean name="ds" class="com.mchange.v2.c3p0.ComboPooledDataSource">
        <property name="driverClass" value="${driverClass}"></property>
        <property name="jdbcUrl" value="${jdbcUrl}"></property>
        <property name="user" value="${user}"></property>
        <property name="password" value="${password}"></property>
    </bean>
    <!-- 3.配置sessionFactory -->
    <bean name="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="configLocation" value="classpath:SqlMapConfig.xml"></property>
        <property name="dataSource" ref="ds"></property>
    </bean>
    <!-- 方式一：Mapper动态代理开发：使用工厂生成mapper接口的实现类对象 -->
    <bean name="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
     <property name="sqlSessionFactory" ref="sqlSessionFactory"></property>
    <!-- 配置Mapper接口-->
     <property name="mapperInterface" value="com.sky.dao.UserMapper"></property>
    </bean>
    <!-- 方式二：Mapper动态带来开发进阶：以扫描包的方式配置代理 -->
    <!--<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.sky.dao"></property>
    </bean>-->
</beans>
```

#### 1.2 service层相关：

##### 	1.2.1 service接口实现类

```java
UserService.java
public interface UserService {
      void register(User user)throws UserException;
      void loginUser(User user) throws UserException;
}
UserServiceImpl.java
  @Transactional
  @Service
  public class UserServiceImpl implements UserService {
          @Autowired
          private UserMapper userMapper;
          @Override
          public void register(User user) throws UserException{
             userMapper.saveUser(user);
          }
          @Override
          public void loginUser(User user) throws UserException {
            userMapper.queryUserByUserNameAndPwd(user);
          }
      }
```

##### 	1.2.2 配置文件(applicationContext_service.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns="http://www.springframework.org/schema/beans"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
    http://www.springframework.org/schema/context
	http://www.springframework.org/schema/context/spring-context-4.3.xsd
	http://www.springframework.org/schema/tx
	http://www.springframework.org/schema/tx/spring-tx-4.3.xsd
	http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans-4.3.xsd">
    <!--配置事务，让Spring来管理 -->
    <bean name="transactionManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="ds"></property>
    </bean>
    <!--开启事务注解功能 -->
    <tx:annotation-driven transaction-manager="transactionManager"/>
    <!-- 配置扫描@Service -->
    <context:component-scan base-package="com.sky.service">
    </context:component-scan>
</beans>
```

#### 1.3 controller控制层类：

#####       1.2.1 UserController类

```java
package com.sky.controller;
@Controller
public class UserController {
    @Autowired
    private UserService userService;
    @RequestMapping("/register.action")
    public String  register(){
        return "register";
    }
    @RequestMapping("/doRegister.action")
    public String  toDoRegister(User user){
       try {
            userService.register(user);
        } catch (UserException e) {
            e.printStackTrace();
            return "fail";
        }
        return "success";
    }
    @RequestMapping("/login.action")
    public String  login(){
        return "login";
    }
    @RequestMapping("/dologin.action")
    public String  tologin(User user){
        try {
            userService.loginUser(user);
        } catch (UserException e) {
            e.printStackTrace();
            return "fail";
        }
        return "success";
    }
}
```

#### 1.4 springmvc的配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns="http://www.springframework.org/schema/beans"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="
	http://www.springframework.org/schema/mvc
	http://www.springframework.org/schema/mvc/spring-mvc-4.3.xsd
	http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
	http://www.springframework.org/schema/context
	http://www.springframework.org/schema/context/spring-context-4.3.xsd">
    <!--1.扫描SpringMVC的注解@Controller  -->
    <context:component-scan base-package="com.sky.controller">
        <context:include-filter type="annotation"
                                expression="org.springframework.stereotype.Controller" />
        <context:include-filter type="annotation"                              									expression="org.springframework.web.bind.annotation.ControllerAdvice" />
    </context:component-scan>
    <!--2.配置处理器 (RequestMappingHandlerMapping,RequestMappingHandlerAdapter)-->
    <mvc:annotation-driven></mvc:annotation-driven>
    <!--3.放行静态资源-->
    <mvc:default-servlet-handler></mvc:default-servlet-handler>
    <mvc:resources mapping="/image/**" location="/WEB-INF/image/"></mvc:resources>
    <!--配置视图解析器-->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/jsp/"></property>
        <property name="suffix" value=".jsp"></property>
    </bean>
    <!--配置上传解析器-->
    <bean name="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
        <property name="maxUploadSize" value="5000000"></property>
    </bean> 
    <!--配置拦截器-->
    <mvc:interceptors>
        <mvc:interceptor>
            <!-- 所有请求都进入拦截器 -->
            <mvc:mapping path="/*"/>
            <!--配置具体的拦截器 -->
            <bean class="com.sky.interceptor.LoginInterceptor"></bean>
        </mvc:interceptor>
    </mvc:interceptors>
</beans>
```

## 第三章：公共部分

####  web.xml配置：

```XML
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    <!--加载Spring配置文件-->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath*:applicationContext_*.xml</param-value>
    </context-param>
    <!--加载SpringMVC前端控制器DispatcherServlet-->
    <servlet>
        <servlet-name>dispatcherServlet</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath*:springmvc.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcherServlet</servlet-name>
        <!--拦截所有，唯独放行jsp-->
        <url-pattern>/</url-pattern>
    </servlet-mapping>
    <!-- 表单提交过滤请求，防止中文乱码 -->
    <filter>
        <filter-name>encoding</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>encoding</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
</web-app>
```

#### log4j.properties

```properties
# Global logging configuration
log4j.rootLogger=DEBUG, stdout
# Console output...
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%5p [%t] - %m%n
```

#### SqlMapConfig.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration   PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration >
    <!-- 设置别名，不区分大小写 -->
    <typeAliases>
        <package name="com.sky.domain" />
    </typeAliases>
    <!-- 开发环境配置数据源等信息交由Spring管理 -->
    <!-- 加载实体类的映射文件 -->
    <mappers>
      <package name="com.sky.dao"></package>
    </mappers>
</configuration>
```

