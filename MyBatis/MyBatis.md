> 基于《深入浅出 MyBatis 技术原理与实战》、[【官网】](https://mybatis.org/mybatis-3/zh/index.html)

## 一、基本概念

[MyBatis](https://mybatis.org/mybatis-3/) 是一个优秀的半自动 ORM（Object Relation Mapping）框架。

MyBatis 对 JDBC 操作数据库的过程进行封装，使开发者只需要关注 SQL 本身，而不需要花费精力去处理例如注册驱动、创建 connection、创建 statement、手动设置参数、结果集检索等 jdbc 繁杂的过程代码。

> 半自动 ORM 框架意思是：需要自己手动将 SQL 语句查询的结果和 POJO 类映射起来（需要手动指定 POJO 类去接收结果集）。

> 背景：MyBatis 本是 Apache 的一个开源项目——iBatis，2010 年迁移到 Google Code，并改名为 MyBatis，2013 年迁移到 github。

## 二、[MyBatis 配置文件参数](https://mybatis.org/mybatis-3/zh/configuration.html)

### （一）`<typeAliases>`

当类的全限定名过长时，可以指定一个别名，在 MyBatis 上下文中使用（例如：ResultType 中）别名。

一个 typeAliases 的实例实在解析配置文件时生成的，然后长期保存在 Configuration 对象中，使用时再拿出来，这样就没必要运行时再次生成它的实例。

> 注意：别名不区分大小写。

#### 1、系统定义别名

MyBatis 提供的别名，在`org.apache.ibatis.type.TypeAliasRegistry`中查看。

#### 2、自定义别名

##### （1）在配置文件中定义别名

```xml
// 给 class MyName 起别名为 myName
<typeAliases>
    <typeAlias alias="myName" type="com.stydunotes.mybatis.MyName"/>
</typeAliases>
```

##### （2）使用@Alias 注解

先配置自动扫描包，再在对应类上添加@Alias 注解。

```xml
// 扫描 mybatis 包下 @Alias 注解
<typeAliases>
    <package name="com.stydunotes.mybatis"/>
</typeAliases>
```

```java
@Alias("myName")
public class MyName {

}
```

### （二）`<typeHandlers>`

传参给 PreparedStatement 或从 SQL 结果获取返回值时，会使用 typeHandler 对字段类型（javaType）和 JDBC 类型（jdbcType）进行映射。

#### 1、系统定义 typeHandler

MyBatis 提供的 typeHandler，在`org.apache.ibatis.type.TypeHandlerRegistry`查看。

#### 2、自定义 typeHandler

当系统定义的 typeHandler 不满足需求时，可以自定义 typeHandler。自定义 typeHandler 需要实现 TypeHandler 接口，并在在 mybatis-config 配置文件中全局配置。

```java
// 匹配指定的 Java 类型
@MappedTypes(String.class)
// 匹配指定的 JdbcType 枚举类型
@MappedJdbcTypes(JdbcType.VARCHAR)
@Slf4j
public class MyStringTypeHandler implements TypeHandler<String> {

    @Override
    public void setParameter(PreparedStatement preparedStatement, int i, String s, JdbcType jdbcType) throws SQLException {
        log.info("使用我的TypeHandler");
        preparedStatement.setString(i, s);
    }

    @Override
    public String getResult(ResultSet resultSet, String s) throws SQLException {
        log.info("使用我的TypeHandler，ResultSet列名获取字符串");
        return resultSet.getString(s);
    }

    @Override
    public String getResult(ResultSet resultSet, int i) throws SQLException {
        log.info("使用我的TypeHandler，ResultSet下标获取字符串");
        return resultSet.getString(i);
    }

    @Override
    public String getResult(CallableStatement callableStatement, int i) throws SQLException {
        log.info("使用我的TypeHandler，CallableStatement下标获取字符串");
        return callableStatement.getString(i);
    }
}
```

- 直接注册

  ```xml
  <typeHandlers>
      <!-- 这里的 string 就是使用了别名 -->
      <!-- <typeHandler jdbcType="VARCHAR" javaType="string" handler="com.stydunotes.mybatis.MyStringTypeHandler" /> -->

      <!-- 使用@MappedTypes、@MappedJdbcTypes注解时，可以不用写jdbcType和javaType -->
      <typeHandler handler="com.stydunotes.mybatis.MyStringTypeHandler" />
  </typeHandlers>
  ```

- 通过包扫描注册

  ```xml
  <typeHandlers>
      <!-- 没有使用@MappedTypes、@MappedJdbcTypes注解时，需要在使用时添加jdbcType和javaType属性 -->
      <package name="com.stydunotes.mybatis" />
  </typeHandlers>
  ```

配置完自定义的 typeHandler 后，有两种使用方法：

> 注意：只有 jdbcType、javaType 和自定义 typeHandler 的一致时才生效

##### （1）在 ResultMap 的列属性中使用

```xml
<resultMap type="com.studynotes.mybatis.Role" id="RoleMap">
    <result property="id" column="id" jdbcType="INTEGER"/>
   	<result property="roleName" column="role_name" typeHandler="com.stydunotes.mybatis.MyStringTypeHandler"/>
    <result property="age" column="age" jdbcType="TINYINT"/>
    <result property="sex" column="sex" jdbcType="TINYINT"/>
</resultMap>
```

##### （2）在 SQL 语句的参数中使用

```xml
<select id="findRole" parameterType="string" resultMap="roleMap">
    select * from t_role
    where role_name like concat('%',
    #{roleName javaType=string, jdbcType=VARCHAR, typeHandler=com.stydunotes.mybatis.MyStringTypeHandler}
    , '%')
</select>
```

#### 3、枚举类型 typeHandler

MyBatis 内部提供两个转化枚举类型的 typeHandler：

- EnumTypeHandler：将 jdbcType 的字符串类型和枚举类型进行转化。

  > 注意：数据库中存储的是枚举类型对应的枚举名称，例如：枚举类`SexEnum.MALE`在数据库中存储的是字符串`MALE`。

- EnumOridinalTypeHandler：将 jdbcType 的整型和枚举类型进行转化。

  > 注意：数据库中存储的是枚举类型对应的下标值，例如：枚举类`SexEnum.values()`返回枚举数组，枚举类`SexEnum.MALE`在数据库中存储的是此枚举类在枚举数据中对应的下标值。

```java
// 枚举类例子：
public enum SexEnum {
    MALE(1, "male"), FEMALE(2, "female");

    // 注意：这里的 id、name 只是枚举类附带的信息，和数据库中存储的数据毫无任何关系
    private int id;
    private String name;

    SexEnum(int id, String name) {
        this.id = id;
        this.name = name;
    }

    public int getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public void setId(int id) {
        this.id = id;
    }

    public void setName(String name) {
        this.name = name;
    }

    public static SexEnum getSexEnum(int id) {
        if(id == 1) {
            return MALE;
        } else if (id == 2) {
            return FEMALE;
        }
        return null;
    }
}
```

##### （1）EnumOridinalTypeHandler

注册 EnumOridinalTypeHandler 后就可以直接使用：

```xml
<typeHandlers>
    <typeHandler handler="org.apache.ibatis.type.EnumOridinalTypeHandler" javaType="com.stydunotes.mybatis.SexEnum"/>
</typeHandlers>
```

```xml
<resultMap type="com.studynotes.mybatis.Role" id="RoleMap">
    <result property="id" column="id" jdbcType="INTEGER"/>
   	<result property="roleName" column="role_name" jdbcType="VARCHAR"/>
    <result property="age" column="age" jdbcType="TINYINT"/>
    <!-- 数据库 sex 列属性为 TINYINT，class Role 的 sex 类型为 SexEnum -->
    <result property="sex" column="sex" typeHandler="org.apache.ibatis.type.EnumOridinalTypeHandler"/>
</resultMap>
```

##### （2）EnumTypeHandler

```xml
<typeHandlers>
    <typeHandler handler="org.apache.ibatis.type.EnumTypeHandler" javaType="com.stydunotes.mybatis.SexEnum"/>
</typeHandlers>
```

```xml
<resultMap type="com.studynotes.mybatis.Role" id="RoleMap">
    <result property="id" column="id" jdbcType="INTEGER"/>
   	<result property="roleName" column="role_name" jdbcType="VARCHAR"/>
    <result property="age" column="age" jdbcType="TINYINT"/>
    <!-- 数据库 sex 列属性为 CHAR，class Role 的 sex 类型为 SexEnum -->
    <result property="sex" column="sex" typeHandler="org.apache.ibatis.type.EnumOridinalTypeHandler"/>
</resultMap>
```

##### （3）自定义枚举类型 typeHandler

```java
// 自定义枚举类型 typeHandler，和EnumOridinalTypeHandler一样的功能
public class MyEnumTypeHandler implements TypeHandler<SexEnum> {

    @Override
    public void setParameter(PreparedStatement preparedStatement, int i, SexEnum sexEnum, JdbcType jdbcType) throws SQLException {
        preparedStatement.setInt(i, sexEnum.getId());
    }

    @Override
    public SexEnum getResult(ResultSet resultSet, String s) throws SQLException {
        int id = resultSet.getInt(s);
        return SexEnum.getSexEnum(id);
    }

    @Override
    public SexEnum getResult(ResultSet resultSet, int i) throws SQLException {
        int id = resultSet.getInt(i);
        return SexEnum.getSexEnum(id);
    }

    @Override
    public SexEnum getResult(CallableStatement callableStatement, int i) throws SQLException {
        int id = callableStatement.getInt(i);
        return SexEnum.getSexEnum(id);
    }
}
```

```xml
<typeHandlers>
    <typeHandler handler="com.stydunotes.mybatis.MyEnumTypeHandler" javaType="com.stydunotes.mybatis.SexEnum"/>
</typeHandlers>
```

```xml
<resultMap type="com.studynotes.mybatis.Role" id="RoleMap">
    <result property="id" column="id" jdbcType="INTEGER"/>
   	<result property="roleName" column="role_name" jdbcType="VARCHAR"/>
    <result property="age" column="age" jdbcType="TINYINT"/>
    <!-- 数据库 sex 列属性为 TINYINT，class Role 的 sex 类型为 SexEnum -->
    <result property="sex" column="sex" typeHandler="org.apache.ibatis.type.EnumOridinalTypeHandler"/>
</resultMap>
```

### （三）`<objectFactory>`

当 MyBatis 在构建一个返回结果时，都会使用 ObjectFactory 去构建 POJO。

#### 1、系统定义 ObjectFactory

系统默认提供并使用`org.apache.ibatis.reflection.factory.DefaultObjectFactory`。

#### 2、自定义 ObjectFactory

需要实现 ObjectFactory 接口，也可以继承 DefaulObjectFactory 类并扩展功能。

```java
@Slf4j
public class MyObjectFactory extends DefaultObjectFactory {
    @Override
    public <T> T create(Class<T> type) {
        log.info("自定义ObjectFactory的create()方法创建POJO类");
        return super.create(type);
    }

    @Override
    public void setProperties(Properties properties) {
        log.info("定制属性：{}", properties);
        super.setProperties(properties);
    }
}
```

```xml
<objectFactory type="com.stydunotes.mybatis.MyObjectFactory">
    <property name="name" value="MyObjectFactory" />
</objectFactory>
```

### （四）`<environments>`

可以有多个配置环境，每个 environment 下分为两个部分：

- 数据库配置 dataSource。
- 数据库事务 transactionManager。

```xml
<!-- default 属性指定使用哪个 environment -->
<environments default="development">
    <!-- id 是 environment 的唯一标志 -->
    <environment id="development">
        <transactionManager type="JDBC">
            <!-- 关闭自动提交 -->
            <property name="autoCommit" value="false" />
        </transactionManager>
        <dataSource type="POOLED">
            <property name="driver" value="com.mysql.jdbc.Driver"/>
            <property name="url" value="jdbc:mysql://localhost:3306/mybatis_study_test?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&autoReconnect=true"/>
            <property name="username" value="root"/>
            <property name="password" value="w110514"/>
        </dataSource>
    </environment>
</environments>
```

#### 1、`<transactionManager>`

transactionManager 配置的是数据库事务，其中 type 数据值有三种：

> 一般使用 Spring 事务管理。

- JDBC：采用 JDBC 方式管理事务，在独立编码中使用。
- MANAGED：采用容器方式管理事务，在 JNDI 数据源中常用。
- 自定义：自定义数据库事务，适用于特殊应用。

#### 2、`<dataSource>`

配置数据源连接的信息，type 属性提供对数据库连接方式的配置：

- UNPOLLED：非连接池，使用 MyBatis 提供的 UnpolledDataSource 实现。

- POOLED：连接池，使用 MyBatis 提供的 PooledDataSource 实现。

- 自定义数据库源实现：需要实现 DataSourceFactory。

  ```java
  public class DbcpDataSourceFactory extends BasicDataSource implements DataSourceFactory {

      private Properties properties;

      @Override
      public void setProperties(Properties properties) {
          this.properties = properties;
      }

      @Override
      public DataSource getDataSource() {
          DataSource dataSource = null;
          try {
              // 获取 DBCP 的数据源
              dataSource = BasicDataSourceFactory.createDataSource(properties);
          } catch (Exception e) {
              e.printStackTrace();
          }
          return dataSource;
      }

      @Override
      public Logger getParentLogger() throws SQLFeatureNotSupportedException {
          return Logger.getLogger(DbcpDataSourceFactory.class.getName());
      }
  }
  ```

  ```xml
  <dataSource type="com.studynotes.mybatis.DbcpDataSourceFactory">
      <property name="driver" value="com.mysql.jdbc.Driver"/>
      <property name="url" value="jdbc:mysql://localhost:3306/mybatis_study_test?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&autoReconnect=true"/>
      <property name="password" value="w110514"/>
  </dataSource>
  ```

### （五）`<mappers>`

引入映射器。

- `<mapper resource="配置文件路径"/>`

  ```xml
  <!-- 放在 classpath 下 -->
  <mapper resource="mapper/RoleDao.xml"/>

  <!-- 放在 package 中 -->
  <mapper resource="com/studynotes/mybatis/RoleDao.xml"/>
  ```

- `<package name="dao包名"/>`，不需要 Mapper 配置文件，适用于使用 iBatis 注解进行增删改查的情况，例如：@Select、@Delete、@Insert 和@Update。

  ```xml
  // 扫描指定包下所有 Dao 接口
  <package name="com.studynotes.mybatis"/>

  // 引入指定 Dao 接口
  <mapper class="com.studynotes.mybatis.RoleDao"/>
  ```

### （六）`<settings>`

MyBatis 的全局配置。

```xml
<settings>
    <setting name="autoMappingBehavior" value="PARTIAL" />
    <setting name="mapUnderscoreToCamelCase" value="false" />
</settings>
```

主要属性：

- autoMappingBehavior：自动映射，自动将结果集绑定 POJO 类。值有三种：
  - NONE：取消自动映射。
  - PARTIAL：默认值。只会自动映射，没有嵌套结果集映射的功能。
  - FULL：会自动映射任意复杂的结果集，无论是否嵌套。性能会下降。
- mapUnderscoreToCamelCase：默认 false，为 true 时可以自动将数据表中下划线式的列名对应 POJO 中驼峰式的属性名，例如：my_name 对应 myName。
- lazyLoadingEnabled：为 true 开启延迟加载。
- aggressiveLazyLoading：为 false 关闭按层级延迟加载。
- localCacheScope：本地缓存的模式，有两个值：
  - SESSION：默认值，开启一级缓存，缓存一个 SqlSession 中执行的所有查询。
  - STATEMENT：本地缓存仅用于此次查询语句，查询完成后便清空此 SqlSession 中的所有缓存。
- cacheEnabled：为 true 开启二级缓存。

## 三、[Mapper 配置文件参数](https://mybatis.org/mybatis-3/zh/sqlmap-xml.html)

MyBatis 是针对映射器构造的 SQL 构建的轻量级框架，并且通过配置文件生成对应的 JavaBean 返回给调用者，而这些配置文件主要是映射器。

> MyBatis 支持自动绑定 JavaBean，只需要 SQL 返回的字段名和 JavaBean 的属性名保持一致（或采用驼峰命名）。

### （一）`<select>`

对应 SQL 的查询语句，可以传入各种类型形参，返回结果。

#### 1、主要属性：

- id：`namespace + id`是唯一的，可以通过 SqlSession 调用，或者在动态 Mapper 中对应 Dao 的方法名。

- parameterType：表示 PreparStatement 中的参数的类型，也就是对应 Dao 方法中的形参类型。

- resultType：resultType 可以自动将结果集映射为指定的 POJO 类，MyBatis 支持自动绑定。

  > 返回一个结果时，resultType 对应 Dao 方法返回值类型；返回多个结果时，resultType 对应 Dao 方法返回值类型中集合的泛型。
  >
  > 例如：方法返回值为`Role`或`List<Role>`，resultType 的值都是 com.studynotes.mybatis.Role。

- resultMap：resultMap 可以自动将结果对应 `<resultMap>` 自定义配置规则，是`<resultMap>`标签映射集的引用。

#### 2、传递多个参数方式

需要传递多个参数给`<select>`。

##### （1）使用 Map 传递参数

Dao 的形参类型为 Map，同时 parameterType 中指定参数类型为 map。

```java
// 传递 roleName 和 note
List<Role> findRoleByMap(Map<String, String> params);
```

```xml
<select id="findRoleByMap" parameterType="map" resultMap="roleMap">
    select * from t_role where
    role_name like concat('%', #{roleName}, '%')
    and
    note like concat('%', #{note}, %)
</select>
```

##### （2）使用注解方式传递属性

MyBatis 提供@Param 注解用于将参数映射到 PreparStatement 中的参数，格式为：`@Param("PreparStatement 的参数名") 参数名`。

```java
// 传递 roleName 和 note
List<Role> findRoleByMap(@Param("roleName") String roleName, @Param("note") String note);
```

```xml
<!-- 这里就不用添加 parameterType 属性 -->
<select id="findRoleByMap" resultMap="roleMap">
    select * from t_role where
    role_name like concat('%', #{roleName}, '%')
    and
    note like concat('%', #{note}, %)
</select>
```

##### （3）使用 POJO 传递参数

将多个参数封装为 POJO 类，需要在 paremeterType 中指定参数类型为 POJO 类型。

```java
@Data
public class RoleParam {
    private String roleName;
    private String note;
}
```

```xml
<select id="findRoleByMap" parameterType="com.studynotes.mybatis.RoleParam" resultMap="roleMap">
    select * from t_role where
    role_name like concat('%', #{roleName}, '%')
    and
    note like concat('%', #{note}, %)
</select>
```

#### 3、参数

##### （1）参数配置

对于传入 SQL 语句中的参数，可以进行配置：

- 指定 javaType、jdbcType 类型，用以确定使用哪个 typeHandler 处理。

  ```xml
  #{age, javaType=int, jdbcType=NUMBERIC}
  ```

- 指定 typeHandler 处理。

  ```xml
  #{age, javaType=int, jdbcType=NUMBERIC, typeHandler=myTypeHandler}
  ```

- 对一些数值型的参数设置保存的精度。

  ```
  #{price, javaType=int, jdbcType=NUMBERIC, numericScale=2}
  ```

### （二）`<insert>`

#### 1、主要属性

- keyProperty：表示哪个列为主键，如果是联合主键，用逗号隔开。
- useGenerateKeys：和 keyProperty 搭配使用，当插入数据的主键没有赋值，而是由数据库内部生成时，MyBatis 会获取主键值并回填给 POJO 类对应属性。

```xml
<insert id="insertRole" parameterType="role" useGenerateKeys="true" keyProperty="id">
    insert into t_role(role_name, note) values (#{roleName}, #{note})
</insert>
```

```java
@Test
public void test() {
    Role role = new Role();
    role.setRoleName("zhangsan");
    role.setNote("is a person");
    roleDao.insertRole(role);
    // 插入完成后可以查看主键 id
    System.out.println(role.getId());
}
```

#### 2、[`<selectKey>`](https://blog.csdn.net/kongkongyanan/article/details/86096657)

将查询值回填给 POJO 类指定属性，和`<insert>、<update>`搭配使用。

主要属性：

- keyProperty
- resultType：返回的结果类型，对应主键类型。
- order：回填查询值的时机，值有两个：
  - BEFORE：先将查询值回填给 POJO 类指定属性，再执行 insert、update 语句。
  - AFTER：先执行 insert、update 语句，再将查询值回填给 POJO 类指定属性。

```xml
<insert id="insertRole" parameterType="role" useGenerateKeys="true" keyProperty="id">
    <!-- 若 t_role 中有数据，则取最大 id + 2 设置插入数据的主键值; 若 t_role 无数据，则取 1 为插入数据的主键值 -->
    <selectKey keyProperty="id" resultType="int" order="BEFORE">
        select if(max(id) is null, 1, max(id) + 2) as newId from t_role
    </selectKey>
    insert into t_role(id, role_name, note) values (#{id}, #{roleName}, #{note})
</insert>
```

### （三）`<update>`

### （四）`<delete>`

### （五）`<sql>`

定义可复用的 sql 语句，通过`<include>`调用。

```xml
<sql id="role_columns">
    #{prefix}.role_no, #{prefix}.role_name
</sql>
```

```xml
<!-- 传入表名，获取指定表名中的列 -->
<select id="getRole" parameterType="string" result="roleMap">
    select
    <include refid="role_columns">
        <property name="prefix" value="${tableName}"/>
    </include>
    from t_role
</select>
```

### （六）`<resultMap>`

用于定义映射规则（自定义 sql 结果集到 POJO 类的映射）、级联的更新、定制 typeHandler 等。

#### 1、`<constructor>`

当 POJO 类没有无参构造时，需要指定`<constructor>`，MyBatis 会使用此构造方法构造 POJO 类。

> `<idArg>`：标记出作为 ID 的结果可以帮助提高整体性能。

```java
public class Role {
    private Integer id;
    private String roleName;
    public Role(Integer id, String roleName) {
        this.id = id;
        this.roleName = roleName;
    }
}
```

```xml
<resultMap>
    <constructor>
        <idArg column="id" name="id" javaType="int" />
        <arg column="role_name" name="roleName" javaType="string" />
    </constructor>
</resultMap>
```

#### 2、`<result>`

注入到字段或 JavaBean 属性的普通结果。

主要属性：

- property：POJO 类字段名。
- colunm：数据库查询结果中的字段名。
- javaType：java 属性类型。
- jdbcType：数据库字段类型。
- typeHandler：类型处理器，制定 jdbcType 和 javaType 转化规则。

#### 3、级联

当数据表有一对一、一对多和多对多关系时，resultMap 中也需要对结果集进行相应的映射。

##### （1）[`<association>`](https://www.jianshu.com/p/018c0f083501)

代表一对一关系，共有三种写法：

```java
// Role 和 RoleCard 一对一关联，RoleCardDto 是关联的 POJO
@Data
public class RoleCardDto implements Serializable {
    private Integer id;
    private String roleName;
    private RoleCard roleCard;
}

@Data
public class RoleCard implements Serializable {
    private Integer id;
    private Integer roleId;
    private String cardName;
}
```

第一种写法：直接把 RoleCard 定义在`<association>`中。

```xml
<resultMap type="com.studynotes.mybatis.RoleCardDto" id="RoleCardDtoMap">
    <result property="id" column="role_id" jdbcType="INTEGER"/>
    <result property="roleName" column="role_name" jdbcType="VARCHAR"/>
    <association property="roleCard" javaType="com.studynotes.mybatis.RoleCard">
        <result property="id" column="card_id" jdbcType="INTEGER"/>
        <result property="roleId" column="role_id" jdbcType="INTEGER"/>
        <result property="cardName" column="card_name" jdbcType="VARCHAR"/>
    </association>
</resultMap>

<select id="findRoleCardDtoById" resultMap="RoleCardDtoMap">
    select
        id as role_id,
        role_name,
        rc.id as card_id,
        card_name
    from role left join role_card rc on role.id = rc.role_id
    where role.id = #{id}
</select>
```

第二种写法：使用 resultMap 属性引用`<resultMap>`的 id。

```xml
<resultMap type="com.studynotes.mybatis.RoleCardDto" id="RoleCardDtoMap">
    <result property="id" column="role_id" jdbcType="INTEGER"/>
    <result property="roleName" column="role_name" jdbcType="VARCHAR"/>
    <!-- 使用 resultMap 属性引用 -->
    <association property="roleCard" resultMap="cardMap" />
</resultMap>

<resultMap type="com.studynotes.mybatis.RoleCard" >
    <result property="id" column="card_id" jdbcType="INTEGER"/>
    <result property="roleId" column="role_id" jdbcType="INTEGER"/>
    <result property="cardName" column="card_name" jdbcType="VARCHAR"/>
</resultMap>

<select id="findRoleCardDtoById" resultMap="RoleCardDtoMap">
    select
        id as role_id,
        role_name,
        rc.id as card_id,
        card_name
    from role left join role_card rc on role.id = rc.role_id
    where role.id = #{id}
</select>
```

第三种写法：使用 select 属性引用`<select>`的 id。

- select：对应`<select>`的 id 名或者是 Dao 方法的全路径名。

- column：对应父查询中的列名或列的别名，同时也是传入子查询的参数名。若需要传入多个参数，则格式为`column="{prop1=col1,prop2=col2}"`

  > 注意：MyBatis 有个特性，当只有一个值作为参数传递时，传入到 PreparStatement 中的参数名不必和 Dao 方法中的形参名一致。
  >
  > ```java
  > Role getById(String aaaaa);
  >
  > <select id="findRoleCardDtoById" resultMap="RoleCardDtoMap">
  >     select * from role where role.id = #{bbbbb}
  > </select>
  > ```
  >
  > 在这里 column 也一样。
  >
  > ```xml
  > <!-- 获取父查询结果的列 my_id 作为参数传递给子查询 -->
  > <association property="roleCard" column="my_id" select="com.studynotes.mybatis.demo01.dao.RoleDtoDao.queryByRoleId"/>
  > <!-- 这里的参数可以随便写，例如写：ccccc -->
  > <select id="queryByRoleId" resultType="com.studynotes.mybatis.RoleCard">
  >     select * from role_card where role_id = #{ccccc}
  > </select>
  >
  > <!-- 获取父查询多个列作为参数传递给子查询 -->
  > <association property="roleCard" column="{myId=my_id,roleName=role_name}" select="com.studynotes.mybatis.demo01.dao.RoleDtoDao.queryByRoleId"/>
  > ```

```xml
<resultMap type="com.studynotes.mybatis.demo01.entity.dto.RoleCardDto" id="RoleCardDtoMap2">
    <result property="id" column="id" jdbcType="INTEGER"/>
    <result property="roleName" column="role_name" jdbcType="VARCHAR"/>
    <!-- select 表示使用的 select 语句，column 表示传入select语句的参数（多个用逗号隔开）-->
    <association property="roleCard" column="id" select="com.studynotes.mybatis.RoleDtoDao.queryByRoleId"/>
</resultMap>

<select id="queryById2" resultMap="RoleCardDtoMap2">
    select id, role_name
    from role
    where id = #{id}
</select>

<select id="queryByRoleId" resultType="com.studynotes.mybatis.RoleCard">
    select id, role_id as roleId, card_name as cardName
    from role_card
    where role_id = #{id}
</select>
```

##### （2）`<collection>`

代表一对多关系，共有三种写法：

> `<collection>`的用法和`<association>`基本一致。

```java
// Role 和 Phone 一对多关联，RoleCardDto 是关联的 POJO
@Data
public class RolePhoneDto implements Serializable {
    private Integer id;
    private String roleName;
    private List<RoleCard> phoneList;
}

@Data
public class Phone implements Serializable {
    private Integer id;
    private String phoneNumber;
}
```

第一种用法：直接把 Phone 定义在`<collection>`中。

> 注意：一对多时，javaType 为 List 类型，`<collection>`提供 ofType 属性的值为 List 的泛型，也就是多的一方的类型。

```xml
<resultMap type="com.studynotes.mybatis.RoleCardDto" id="RoleCardDtoMap">
    <result property="id" column="role_id" jdbcType="INTEGER"/>
    <result property="roleName" column="role_name" jdbcType="VARCHAR"/>
    <!-- ofType 为 Phone 的类型 -->
    <collection property="phoneList" javaType="list" ofType="com.studynotes.mybatis.Phone">
        <result property="id" column="card_id" jdbcType="INTEGER"/>
        <result property="phoneNumber" column="phone_number" jdbcType="VARCHAR"/>
    </collection>
</resultMap>
```

##### （3）`<discriminator>`

鉴别器是获取结果集后，根据特定条件使用不同的 POJO。

> 鉴别器类似于 Java 中的 switch 语句，用于根据不同情景生成类的不同子类。

```java
/**
 * 数据表字段：
 * id int,
 * year CHAR(4),
 * color VARCHAR(10),
 * vehicle_type int,
 * door_count int,
 * box_size int,
 * all_wheel_drive int
 **/

// 根据 vehicle_type 值的不同，将结果集映射为的 Vehicle 不同实现类
@Data
public class Vehicle {
    private int id;
    private String year;
    private String color;
    private int vehicleType;
}

@Data
public class Car extends Vehicle {
    private int doorCount;
}

@Data
public class Truck extends Vehicle {
    private int boxSize;
}

@Data
public class Suv extends Vehicle {
    private int allWheelDrive;
}
```

第一种：

```xml
<resultMap id="vehicleResult" type="Vehicle">
    <id property="id" column="id" />
    <result property="vin" column="vin"/>
    <result property="year" column="year"/>
    <result property="make" column="make"/>
    <result property="model" column="model"/>
    <result property="color" column="color"/>
    <discriminator javaType="int" column="vehicle_type">
        <case value="1" resultType="carResult">
            <result property="doorCount" column="door_count" />
        </case>
        <case value="2" resultType="truckResult">
            <result property="boxSize" column="box_size" />
        </case>
        <case value="3" resultType="suvResult">
            <result property="allWheelDrive" column="all_wheel_drive" />
        </case>
  </discriminator>
</resultMap>
```

第二种：使用 resultMap 属性引用`<resultMap>`的 id，使用指定结果集映射获取结果。

> `<discriminator>`根据`<resultMap>`id 指定的结果集映射提供 extends 属性，用于表示继承某个`<resultMap>`，当没有使用 extends 属性时，只会填充`<resultMap>`中包含的属性。

```xml
<resultMap id="vehicleResult" type="Vehicle">
    <id property="id" column="id" />
    <result property="vin" column="vin"/>
    <result property="year" column="year"/>
    <result property="make" column="make"/>
    <result property="model" column="model"/>
    <result property="color" column="color"/>
    <discriminator javaType="int" column="vehicle_type">
        <case value="1" resultMap="carResult"/>
        <case value="2" resultMap="truckResult"/>
        <case value="3" resultMap="suvResult"/>
    </discriminator>
</resultMap>

<!-- 使用 extends 属性继承 vehicleResult resultMap 的属性 -->
<resultMap id="carResult" type="Car" extends="vehicleResult">
  <result property="doorCount" column="door_count" />
</resultMap>

<!-- 不使用 extends 属性继承时，返回结果只有 boxSize 字段 -->
<resultMap id="carResult" type="Truck">
  <result property="boxSize" column="box_size" />
</resultMap>
```

#### 4、延迟加载

当级联的表很多时，查出所有级联的数据却又不使用时，很耗性能。因此可以延迟加载级联的数据，访问级联数据时 MyBatis 才会发送 SQL 查出响应的级联数据。

> 注意：
>
> - 只有使用 select 属性引用`<select>`的 id 级联方式才有效，例如：`<association property="roleCard" column="id" select="com.studynotes.mybatis.demo01.dao.RoleDtoDao.queryByRoleId"/>`。
> - 使用表连接方式进行级联的话，就没有延迟加载的概念。

##### （1）全局配置

在 mybatis-config 的`<settings>`中设置：

- lazyLoadingEnabled：为 true 开启延迟加载。

- aggressiveLazyLoading： 为 false 关闭按层级延迟加载。

  > 例如：学生级联（课程成绩和学生证）时，此时课程成绩和学生证在同一层级（在 resultMap 中标签也是同一层级）。若开启延迟后，访问课程成绩时，同时也会对学生证加载。此时需要关闭按层级延迟加载。

##### （2）局部配置

若不想进行全局配置，只想对某些级联查询开启/关闭延迟加载，可以在`<association>`标签中添加 fetchType 属性。

- fetchType：lazy 表示开启延时加载；eager 表示关闭延时加载

## 四、动态 SQL 标签

MyBatis 中提供动态 SQL 标签，用于动态的生成 SQL 语句，提高 MyBatis 的灵活性。

### （一）`<if>`

提供 test 属性，当 test 属性值为 true 时，才会将包裹的 SQL 语句加入。

```xml
<select>
    select * from table_name where 1 = 1
    <if test = "if != null and id != ''">
        and id = #{id}
    </if>
</select>
```

### （二）`<choose>、<when>、<otherwise>`

组合起来类似于 if...else if...else 的功能。

```xml
<select>
    select * from table_name where 1 = 1
    <choose>
        <when test="id != null and id != ''">
            and id = #{id}
        </when>
        <when test="name != null and name != ''">
            and name like concat("", #{name}, "")
        </when>
        <otherwise>
            and age is not null
        </otherwise>
    </choose>
</select>
```

### （三）`<where>`

一般和 if 标签一起使用，相当于 where 1 = 1。

```xml
<select>
    select * from table_name
    <where>
        <if test = "if != null and id != ''">
            and id = #{id}
        </if>
    </where>
</select>
```

### （四）`<trim>`

用于替代 SQL 语句中指定的字符串，prefix 属性值替代 prefixOverrides 属性值，多个属性值用【|】隔开，例如：`AND |OR `。

```xml
<select>
    select * from table_name
    <trim prefix="where" prefixOverrides="and">
        <if test="id != null and id != ''">
            and id = #{id}
        </if>
    </trim>
</select>
```

### （五）`<set>`

`<set>`会动态地在行首插入 SET 关键字，并会删掉额外的逗号。

```xml
<update id="updateAuthorIfNecessary">
    update Author
        <set>
            <if test="username != null">username=#{username},</if>
            <if test="password != null">password=#{password},</if>
            <if test="email != null">email=#{email},</if>
            <if test="bio != null">bio=#{bio}</if>
        </set>
    where id=#{id}
</update>
```

### （六）`<foreach>`

循环语句，item 属性为循环变量，index 属性为下标，collection 属性为参数，open 和 close 属性为以什么符号将这些元素包裹，separator 为各个元素的间隔符。

```xml
<select parameterType="list">
    select * from table_name where
    sex in
    <foreach item="sex" index="index" collection="sexLits" open="(" separator="," close=")">
        #{sex}
    </foreach>
</select>
```

### （七）`<script>`

要在带注解的映射器接口类中使用动态 SQL，可以使用`<script>`元素。

```java
@Update({"<script>",
    "update Author",
    "  <set>",
    "    <if test='username != null'>username=#{username},</if>",
    "    <if test='password != null'>password=#{password},</if>",
    "    <if test='email != null'>email=#{email},</if>",
    "    <if test='bio != null'>bio=#{bio}</if>",
    "  </set>",
    "where id=#{id}",
    "</script>"})
void updateAuthorValues(Author author);
```

### （八）`<bind>`

`<bind >`用于通过 OGNL 表达式自定义一个变量，并将其绑定到当前的上下文。

```xml
<!-- 使用 bind 对 name 进行处理 -->
<select id="selectBlogsLike" resultType="Blog">
    <bind name="pattern_name" value="'%' + name + '%'" />
    select * from table_name where name like #{pattern_name}
</select>

<!-- 不使用 bind 对 name 进行处理 -->
<select id="selectBlogsLike" resultType="Blog">
    <bind name="pattern_name" value="'%' + name + '%'" />
    select * from table_name where name like concat('%', #{name}, '%')
</select>
```

## 五、[MyBatis 缓存](https://tech.meituan.com/2018/01/19/mybatis-cache.html)

MyBatis 提供一级缓存和二级缓存，用于减少 MyBatis 重复查询数据库。

![MyBatis一、二级缓存模型](https://onezilin.github.io/StudyNotes/MyBatis/MyBatis一、二级缓存模型.png)

### （一）一级缓存

一级缓存只针对一个 SqlSession 有效，每个`namespace + selectId`的首次查询结果都会缓存。接下来的对于`namespace + selectId`的查询不会查询数据库，而是直接从缓存中获取。若此 SqlSession 执行了增删改 SQL 语句，则会清空此 SqlSession 中的所有缓存。

在 mybatis-config 的`<settings>`中设置 localCacheScope（本地缓存的模式），有两个值：

- SESSION：默认值，开启一级缓存，缓存一个 SqlSession 中执行的所有查询。
- STATEMENT：本地缓存仅用于此次查询语句，查询完成后便清空此 SqlSession 中的所有缓存。

> MyBatis 默认开启一级缓存。
>
> 优缺点：
>
> - 可以减少对 MyBatis 的查询次数，提高性能。
> - 一级缓存只对一个 SqlSession 有效，在分布式或者多线程的环境下，若其他 SqlSession 进行了增删改操作，对于此 SqlSession 是不可知的，会造成脏数据。

#### 1、原理

![一级缓存原理](https://onezilin.github.io/StudyNotes/MyBatis/一级缓存原理.png)

- SqlSession 内部维护一个 Executor，CURD 方法都最终都会调用 Executor 的对应方法。
- Executor（BaseExecutor）维护一个 Cache（PerpetualCache），SqlSession 调用查询方法时，最终调用 Executor 的 query()方法。
  1. 获取此次查询的 CacheKey（由`MappedStatement的Id、SQL的offset、SQL的limit、SQL本身以及传给SQL的参数`组成）。
  2. 当 flushCache 为 true 时清空 Cache。
  3. 尝试通过 CacheKey 获取 Cache 中对应的值，若有，则返回；若没有，则从数据库中查询后返回。
  4. 当 localCacheScope 值为【STATEMENT】时，则会清空 Cache。

> BaseExecutor 的 queryStack 属性，确保在多线程环境下，SqlSession 可以使用 Cache。

### （二）二级缓存

二级缓存和一级缓存不同的是：二级缓存中对于每个 namespace 有单独的 Cache，可以被多个 SqlSession 共享。

> 注意：调用 SqlSession.commit()二级缓存才生效。

在 mybatis-config 中添加`<setting name="cacheEnabled" value="true"/>`开启二级缓存。

> 优缺点：
>
> - 可以解决 SqlSession 中不共享的问题。
>
> - 每个 namespace 有单独的 Cache，一个 namespace 中的增删改对于另一个 namespace 是不可知的，因此对于级联（多表关联）查询时也可能读到脏数据。
>
>   > 在 mapper 配置文件中添加`<cache-ref namespace="其他namespaceId"/>`声明和其他 namespace 使用同一个 Cache 空间和配置，就可以解决级联（多表关联）查询中更新不可知的问题。

在 mapper 配置文件中添加`<cache/>`声明此 namespace 使用二级缓存，属性如下：

- type：Cache 使用的类型，默认是 PerpetualCache，也可以实现 Cache 接口，自定义 Cache。

  > PerpetualCache 最终会被装饰成 SynchronizedCache ，装饰链为：`PerpetualCache <- LruCache <- SerializedCache <- LoggingCache <- SynchronizedCache`。
  >
  > - PerpetualCache： 作为最基础的缓存类，底层实现比较简单，直接使用了 HashMap。
  > - LruCache：采用了 Lru 算法的 Cache 实现，移除最近最少使用的 Key/Value。
  > - SerializedCache：序列化功能，将值序列化后存到缓存中。该功能用于缓存返回一份实例的 Copy，用于保存线程安全。
  > - LoggingCache：日志功能，装饰类，用于记录缓存的命中率，如果开启了 DEBUG 模式，则会输出命中率日志。
  > - SynchronizedCache：同步 Cache，实现比较简单，直接使用 synchronized 修饰方法。
  >
  > 自定义的 Cache 也会根据成员变量不同，被上面的 Cache 类装饰。

- eviction：定义回收策略，常见的有 FIFO、LRU。

- flushInterval：刷新缓存时间。

- size：最多缓存对象的个数。

- readOnly：只读，意味着缓存数据只能读取不能修改。默认 false，也就是可读写，需要对应实体类能够序列化。

`<select>、<insert>、<update>、<delete>`中有两个属性 useCache 和 flushCache，用来更细化地控制缓存：

- useCache：是否需要使用二级缓存。
- flushCache：此操作是否清空一级、二级缓存，默认`<select>`为 false，`<insert>、<update>、<delete>`为 true。

#### 1、原理

![二级缓存原理](https://onezilin.github.io/StudyNotes/MyBatis/二级缓存原理.png)

- 当开启二级缓存时，会使用装饰器模式，将创建的 BaseExecutor 装饰成 CacheExecutor。

- Executor（CacheExecutor）维护一个 TransactionalCacheManager，SqlSession 调用查询方法时，最终调用 Executor 的 query()方法。

  > TransactionalCacheManager 中维护一个 Map（Map<Cache, TransactionalCache>类型），保存 MappedStatement 的 Cache 属性（二次缓存，以 Map<String, Cache>的形式存储在 Configuration 上，String 为 nameSpace 名称，也就是此 namespace 的 Cache，可以被所有 SqlSession 共享）和 TransactionalCache 的映射。
  >
  > TransactionalCache 实现 Cache 接口，封装 Cache（和上面的 Cache 是同一个，下面的 Cache 同理），作用是如果事务提交，对缓存的操作才会生效，如果事务回滚或者不提交事务，则不对缓存产生影响。

  1. 获取此次查询的 CacheKey（由`MappedStatement的Id、SQL的offset、SQL的limit、SQL本身以及传给SQL的参数`组成）。

  2. <font color="DeepSkyBlue">flushCacheIfRequired(MappedStatement)</font>方法作用：当 flushCache 为 true 时清空 TransactionalCache，同时设置清理标志位 clearOnCommit 为 true。

  > 注意：此方法只会清空 TransactionalCache，对 Cache 没有影响。

  3. 判断 useCache 是否为 true，为 true 才可以使用二级缓存。
  4. 尝试通过 Cache 和 CacheKey 从 TransactionalCacheManager 中获取对应的值（最终还是从 Cache 中 CacheKey 对应的值）：

     - 若有，则返回。

     - 若没有，则执行一次缓存的查询逻辑，将返回值通过 tcm.putObject(Cache, CacheKey, List)存储在 TransactionalCache 中。

       > 注意：此时查询结果还没有放在 Cache 中，而是在 TransactionalCache 中。

- SqlSession 调用增删改方法时，也会调用<font color="DeepSkyBlue">flushCacheIfRequired(MappedStatement)</font>方法。

- SqlSession 调用 commit()方法时，首先判断清理标志位 clearOnCommit 为 true 时，清空 Cache；再将 TransactionalCache 中缓存放入 Cache 中。

  > 从这里可以看出，调用 SqlSession.commit()时，对二级缓存的操作才生效。

---

## 六、MyBatis 主要对象

![Mybatis基本架构](https://onezilin.github.io/StudyNotes/MyBatis/Mybatis基本架构.png)

### （一）SqlSessionFactoryBuilder

SqlSessionFactoryBuilder 主要作用是通过配置文件或 Configuration 类生成 SqlSessionFactory。

### （二）<font color=orange>SqlSessionFactory</font>

SqlSessionFactory 是工厂类，主要用于生成 SqlSession 对象实例。

> SqlSessionFactory 创建和销毁都比较重量级，因此建议使用单例模式创建。

共有[两种构建 SqlSessionFactory 的方式](https://blog.csdn.net/zy12306/article/details/84260956)：

第一种：通过 xml 配置文件。

```xml
<configuration>
    <!-- 定义Role类的别名，在mapper中可以直接使用role -->
    <typeAliases>
        <typeAlias type="com.studynotes.mybatis.Role" alias="role"/>
    </typeAliases>
    <!-- 和spring整合后 environments配置将废除-->
    <environments default="development">
        <environment id="development">
            <!-- 使用jdbc事务管理-->
            <transactionManager type="JDBC"/>
            <!-- 数据库连接池-->
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/mybatis_study_test?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&autoReconnect=true"/>
                <property name="username" value="root"/>
                <property name="password" value="w110514"/>
            </dataSource>
        </environment>
    </environments>
    <!-- 加载sql映射文件-->
    <mappers>
        <mapper resource="mapper/RoleDao.xml"/>
    </mappers>
</configuration>
```

```java
// 使用配置文件方式构建SqlSessionFactory
public static SqlSessionFactory getSqlSessionFactory() {
    if (sqlSessionFactory == null) {
        try {
            String resource = "mybatis_config.xml";
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(Resources.getResourceAsReader(resource));
            return sqlSessionFactory;
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    return sqlSessionFactory;
}
```

第二种：通过 Java API。

```java
// 对应着配置文件，使用代码方式构建SqlSessionFactory（不推荐）
public static SqlSessionFactory getSqlSessionFactory2() {
    if (sqlSessionFactory == null) {
        // 数据库连接池
        PooledDataSource dataSource = new PooledDataSource();
        dataSource.setDriver(Driver.class.getName());
        dataSource.setUrl("jdbc:mysql://localhost:3306/mybatis_study_test?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&autoReconnect=true");
        dataSource.setUsername("root");
        dataSource.setPassword("w110514");
        // 构建数据库事务方式
        TransactionFactory transactionFactory = new JdbcTransactionFactory();
        Environment environment = new Environment("development", transactionFactory, dataSource);
        // 构建Configuration
        Configuration configuration = new Configuration();
        // 注册别名
        configuration.getTypeAliasRegistry().registerAlias("role", Role.class);
        // 添加映射器
        configuration.addMapper(RoleDao.class);
        // 构建SqlSessionFactory
        sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
    }
    return sqlSessionFactory;
}
```

### （三）<font color=orange>SqlSession</font>

SqlSession 类似于 JDBC 中的一个 Connection 连接，对外提供了用户和数据库之间交互需要的所有方法，隐藏了底层的细节。

> 使用完需要将 SqlSession 连接关闭，放回连接池。

#### 1、DefaultSqlSession

##### （1）[MyBatis 执行原理](https://www.cnblogs.com/jian0110/p/9452592.html)

![MyBatis执行原理](https://onezilin.github.io/StudyNotes/MyBatis/MyBatis执行原理.png)

- SqlSession 内部维护一个 Executor，CURD 方法都最终都会调用 Executor 的对应方法。
- Executor 实现类的 doQuery 中，通过`configuration.newStatementHandler(Executor, MappedStatement, Object, RowBounds, ResultHandler, BoundSql)`有参构造 StatementHandler。
  - Executor：当前执行 doQuery(..)方法的 Executor。
  - MappedStatement：CURD 标签 id 对应的对象，内部存储当前标签的所有信息，例如：标签属性值、参数、SQL 语句等。
  - Object：传入的参数。
  - RowBounds：存储 SQL 的分页参数（limit 和 offset）。
  - ResultHandler：处理结果集 ResultSet。
  - BoundSql：存储 SQL 语句等信息。
- 创建 Statement 实例对象（也就是 JDBC 的 Statement），访问数据库，执行 SQL 语句。

### （四）Mapper

Mapper 映射器是由 Java 接口和 XML 文件（或注解）共同组成，可以定义参数类型、SQL 语句、结果集与 POJO 映射等功能。

三种实现：

#### 1、xml 文件

SqlSession 支持 iBatis 方式发送 SQL 并返回数据。

> 优缺点：
>
> - 只需要 XML 文件（也要在 Configuration 上注册映射器），就可以执行 PreparStatement 预处理语句。
> - 需要使用（namespace + 增删改查标签 id）方式调用对应的 PreparStatement，不易于编码和维护。

```java
public Role queryById(Integer id) {
    SqlSession sqlSession = sqlSessionFactory().openSession();
    // 以（xml 中的 namespace） + （xml 中增删改查标签的 id）方式构建 statement
    Role role = sqlSession.selectOne("aaaa.queryById", "1");
    sqlSession.close();
    return role;
}
```

#### 2、Java 接口和注解

iBatis 支持在 Java 接口方法上添加注解：@Select、@Delete、@Insert、@Update。

> 优缺点：
>
> - 只需要 Java 接口（也要在 Configuration 上注册映射器），就可以执行 PreparStatement 预处理语句，不用添加对应的 xml 文件。
> - 不如在 xml 中那么灵活；注解的 value 值很长等问题。

```java
public interface RoleDao {
    @Select("select * from role where id = #{id}")
    Role queryById(Integer id);
}
```

```java
@Test
public void test(Integer id) {
    try (SqlSession sqlSession = MyBatisUtil.getSqlSessionFactory().openSession()) {
        RoleDao roleDao = sqlSession.getMapper(RoleDao.class);
        Role role = roleDao.queryById(1);
        log.info("role：[{}]", role);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

#### 3、Java 接口和 xml 文件

MyBatis 通过动态 Mapper 方式，同时需要 xml 和 Java 接口。xml 中 namespace 和 Java 接口全限定名相同，增删改查 id 和接口中方法名相同。

> 优点：只需要 XML 文件（也要在 Configuration 上注册映射器），就可以执行 PreparStatement 预处理语句。

```xml
<!-- xml 中 namespace 和 Java 接口全限定名相同 -->
<mapper namespace="com.studynotes.mybatis.RoleDao">
    <resultMap type="com.studynotes.mybatis.Role" id="RoleMap">
        <result property="id" column="id" jdbcType="INTEGER"/>
        <result property="roleName" column="role_name" jdbcType="VARCHAR"/>
        <result property="age" column="age" jdbcType="TINYINT"/>
        <result property="sex" column="sex" jdbcType="TINYINT"/>
    </resultMap>

    <!-- 增删改查 id 和接口中方法名相同 -->
    <select id="queryById" resultMap="RoleMap">
        select id,
               role_name,
               age,
               sex
        from mybatis_study_test.role
        where id = #{id}
    </select>
</mapper>
```

```java
@Test
public void test(Integer id) {
    try (SqlSession sqlSession = MyBatisUtil.getSqlSessionFactory().openSession()) {
        RoleDao roleDao = sqlSession.getMapper(RoleDao.class);
        Role role = roleDao.queryById(1);
        log.info("role：[{}]", role);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

##### （1）[动态 Mapper 原理](https://blog.csdn.net/xiaokang123456kao/article/details/76228684)

- SqlSession.getMapper(..)时，MyBatis 通过动态代理方式生成对应接口的代理类。

  - Configuration 中的 MapperRegistry 维护`Map<Class<?>, MapperProxyFactory<?>> knownMappers`属性。knownMappers 是根据 mybatis_config 中`<mappers>` 引入的映射器进行初始化（根据 namespace 的值获取对应的 Class 对象作为 key，根据 Class 对象构建 MapperProxyFactory 作为 value）。

    > 所以动态 Mapper 中 namespace 的值必须是 Dao 接口的全路径名。

  - MapperProxyFactory.instance(SqlSession)方法中创建了 MapperProxy 实例。

  - MapperProxy 实现了 InvocationHandler，通过动态代理方式生成 Dao 接口的代理类。

- 调用 Dao 接口的方法时，代理类调用了 MapperProxy.invoke(..)方法，最终会执行 MapperMethod.execute(..)方法。

  - MapperMethod 封装了 MethodSignature（存储 Dao 接口的方法签名等信息）和 SqlCommand（Dao 接口的方法对应的 Mapper 配置文件中标签的 id 和类型，例如：`<select id="queryById">`）。

  - execute(..)方法中根据 SqlCommand 的属性，最终会通过 iBatis 方式发送 SQL 并返回数据，从而实现动态 Mapper 功能。

    > 所以动态 Mapper 中增删改查标签的 id 值必须和 Dao 接口的方法名相同。

## 七、MyBatis 应用

### （一）[Spring 中应用](https://www.jianshu.com/p/412051d41d73)

在 Mybatis+Spring 中为了方便，一般都使用动态 Mapper+MapperScannerConfigurer 方式。

> [原理](https://www.cnblogs.com/xiaolang8762400/p/7399274.html)：MapperScannerConfigurer 可以为指定包下的接口生成对应的代理类，并注册到 Spring 中，这样就可以直接使用 Dao 接口的 bean 对象。

MapperScannerConfigurer 在 application.xml 中配置：

```xml
<!-- 分解配置 jdbc.properites -->
<context:property-placeholder location="classpath:jdbc.properties"/>

<!-- 数据源Druid -->
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
    <property name="driver" value="${jdbc.driverClassName}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
</bean>

<!-- sessionFactory 将spring和mybatis整合 -->
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="configLocation" value="classpath:spring-mybatis.xml"/>
    <property name="mapperLocations" value="classpath:/mapper/*Mapper.xml"/>
</bean>

<!-- Mapper 扫描器 -->
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <!-- 扫描指定包下的组件 -->
    <property name="basePackage" value="com.studynotes.mybatis"/>
</bean>

<!-- 也可以使用这种形式 -->
<mybatis:scan base-package="org.mybatis.spring.sample.mapper" />
```

### （二）[SpringBoot 中应用](https://segmentfault.com/a/1190000018559769)

```java
// 在Configuration配置类中添加Mapper扫描器注解
@Configuration
@MapperScan(basePackages = "com.studynotes.mybatis")
public class MyBatisConfig {

}
```

```yml
# 设置连接信息
spring:
  datasource:
    name: warehouse_project
    url: jdbc:mysql://localhost:3306/mybatis_study_test?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&autoReconnect=true
    username: root
    password: W110514
    type: com.alibaba.druid.pool.DruidDataSource
# 引入映射器
mybatis:
  mapper-locations: classpath:mapper/*.xml
```

## 八、疑问

### （一）`#{}`和`${}`的区别？

- `#{}`会创建预编译的语句，然后 MyBatis 设值，并且会在值前后添加【'】单引号。
- `${}`不会对传入的参数做任何处理，容易被注入攻击。

### （二）MyBatis、JDBC 和 Hibernate 的区别？

- JDBC：MyBatis 对 JDBC 操作数据库的过程进行封装，使开发者只需关注 SQL 本身，不需要处理例如：注册驱动、创建 Connection、创建 PreparStatement（预处理语句）、设置 SQL 参数、结果集检索等冗余复杂的过程。
- Hibernate：更加面向对象化，与 SQL 语句无关，更容易进行数据库迁移；Hibernate 是全自动 ORM 框架，需要进行全表映射，因此更新时需要发送所有字段；无法进行 SQL 优化。
