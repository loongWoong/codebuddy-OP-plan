# Datart 后端代码编写规范

## 1. 项目架构概述

### 1.1 模块结构
- **core**: 核心模块，包含实体类、Mapper、基础工具类
- **server**: 服务模块，包含Controller、Service实现
- **security**: 安全模块，处理认证授权
- **data-providers**: 数据提供者模块

### 1.2 技术栈
- Spring Boot 2.4.3
- MyBatis
- Java 8
- Lombok
- JWT认证

## 2. 包结构规范

```
datart.core/
├── base/                    # 基础类和注解
│   ├── annotations/         # 自定义注解
│   ├── consts/             # 常量定义
│   ├── exception/          # 异常类
│   └── processor/          # 处理器
├── common/                 # 通用工具类
├── entity/                 # 实体类
├── mappers/               # MyBatis Mapper接口
└── data/                  # 数据相关

datart.server/
├── controller/            # 控制器层
├── service/              # 服务接口
├── service/impl/         # 服务实现
└── base/                 # 基础DTO和参数类
```

## 3. 实体类编写规范

### 3.1 基础实体类
所有实体类必须继承 `BaseEntity`：

```java
@Data
@SuperBuilder
@NoArgsConstructor
public class BaseEntity implements Serializable {
    private String id;
    private String createBy;
    private Date createTime;
    private String updateBy;
    private Date updateTime;
    private Integer permission;
}
```

### 3.2 实体类规范
```java
@Data
@EqualsAndHashCode(callSuper = true)
public class User extends BaseEntity {
    private String email;
    private String username;
    private String password;
    private Boolean active;
    private String name;
    private String description;
    private String avatar;
}
```

**规范要点：**
- 使用 `@Data` 注解自动生成getter/setter
- 继承BaseEntity时使用 `@EqualsAndHashCode(callSuper = true)`
- 字段命名使用驼峰命名法
- 布尔类型使用包装类 `Boolean`

## 4. Controller层编写规范

### 4.1 Controller基本结构
```java
@Api
@Slf4j
@RestController
@RequestMapping(value = "/users")
public class UserController extends BaseController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }
}
```

### 4.2 接口方法规范
```java
@ApiOperation(value = "用户注册")
@PostMapping("/register")
public ResponseData<Boolean> register(@Validated @RequestBody UserRegisterParam user) 
    throws MessagingException, UnsupportedEncodingException {
    if (!Application.canRegister()) {
        Exceptions.tr(PermissionDeniedException.class, "message.provider.execute.operation.denied");
    }
    return ResponseData.success(userService.register(user));
}
```

**规范要点：**
- 使用 `@ApiOperation` 添加接口描述
- 参数验证使用 `@Validated`
- 统一返回 `ResponseData<T>` 格式
- 异常处理使用 `Exceptions` 工具类
- 权限检查在方法开始处进行

### 4.3 权限控制
```java
@SkipLogin  // 跳过登录验证
@ApiOperation(value = "用户激活")
@GetMapping(value = "/active")
public ResponseData<String> activate(@RequestParam("token") String activeToken) {
    checkBlank(activeToken, "activeToken");
    return ResponseData.success(userService.activeUser(activeToken));
}
```

## 5. Service层编写规范

### 5.1 Service接口定义
```java
public interface UserService extends BaseCRUDService<User, UserMapperExt> {
    
    UserProfile getUserProfile();
    
    List<UserBaseInfo> listUsersByKeyword(String keyword);
    
    boolean register(UserRegisterParam user) throws MessagingException, UnsupportedEncodingException;
    
    String login(PasswordToken passwordToken);
}
```

### 5.2 Service实现类
```java
@Service
@Slf4j
public class UserServiceImpl extends BaseService implements UserService {

    private final UserMapperExt userMapper;
    private final OrgService orgService;
    private final RoleService roleService;
    private final MailService mailService;

    public UserServiceImpl(UserMapperExt userMapper,
                           OrgService orgService,
                           RoleService roleService,
                           MailService mailService) {
        this.userMapper = userMapper;
        this.orgService = orgService;
        this.roleService = roleService;
        this.mailService = mailService;
    }
}
```

**规范要点：**
- 使用 `@Service` 注解
- 添加 `@Slf4j` 用于日志记录
- 继承 `BaseService`
- 构造函数注入依赖
- 所有依赖声明为 `final`

### 5.3 事务处理
```java
@Override
@Transactional
public boolean register(UserRegisterParam userRegisterParam) 
    throws MessagingException, UnsupportedEncodingException {
    // 业务逻辑
    return register(userRegisterParam, sendEmail);
}

@Transactional(propagation = Propagation.MANDATORY)
public void initUser(User user) {
    // 必须在事务中执行的方法
}
```

## 6. 数据访问层规范

### 6.1 Mapper接口定义
```java
@Mapper
public interface UserMapperExt extends UserMapper {

    @Select({
        "SELECT ",
        "	u.*  ",
        "FROM ",
        "	`user` u  ",
        "WHERE ",
        "	u.username = #{username} or ",
        "	u.email = #{username}"
    })
    User selectByNameOrEmail(@Param("username") String username);

    @Update({
        "UPDATE `user` u " +
        "SET u.active = 1  " +
        "WHERE " +
        "	u.active = 0  " +
        "	AND u.id = #{userId}"
    })
    int updateToActiveById(String id);
}
```

**规范要点：**
- 使用 `@Mapper` 注解
- 继承基础Mapper接口
- SQL语句使用注解方式编写
- 参数使用 `@Param` 注解
- 表名和字段名使用反引号包围

### 6.2 缓存配置
```java
@Mapper
@CacheNamespace(flushInterval = 5 * 1000)
public interface FolderMapperExt extends FolderMapper {
    // 设置缓存刷新间隔
}

@Mapper
@CacheNamespaceRef(value = FolderMapperExt.class)
public interface DatachartMapperExt extends DatachartMapper {
    // 引用其他Mapper的缓存配置
}
```

## 7. 异常处理规范

### 7.1 异常类层次结构
```java
public class BaseException extends RuntimeException {
    @Getter
    @Setter
    private int errCode;
    
    public BaseException(String message, int errCode) {
        super(message);
        this.errCode = errCode;
    }
}
```

### 7.2 异常使用规范
```java
// 参数校验异常
if (!checkUserName(userRegisterParam.getUsername())) {
    log.error("The username({}) has been registered", userRegisterParam.getUsername());
    Exceptions.tr(ParamException.class, "error.param.occupied", "resource.user.username");
}

// 资源不存在异常
if (user == null) {
    Exceptions.notFound("resource.not-exist", "resource.user");
}

// 权限异常
if (!Application.canRegister()) {
    Exceptions.tr(PermissionDeniedException.class, "message.provider.execute.operation.denied");
}
```

## 8. 响应数据格式规范

### 8.1 统一响应格式
```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ResponseData<T> {
    private boolean success;
    private int errCode;
    private String message;
    private List<String> warnings;
    private T data;
    private PageInfo pageInfo;
    private Exception exception;

    public static <E> ResponseData<E> success(E data) {
        return ResponseData.<E>builder()
                .data(data)
                .success(true)
                .build();
    }

    public static <E> ResponseData<E> failure(String message) {
        return ResponseData.<E>builder()
                .success(false)
                .message(message)
                .build();
    }
}
```

### 8.2 响应使用规范
```java
// 成功响应
return ResponseData.success(userService.getUserProfile());

// 失败响应
return ResponseData.failure("操作失败");
```

## 9. 日志记录规范

### 9.1 日志级别使用
```java
@Slf4j
public class UserServiceImpl {
    
    // 信息日志
    log.info("User({}) activation success", user.getUsername());
    
    // 错误日志
    log.error("The username({}) has been registered", userRegisterParam.getUsername());
    
    // 调试日志
    log.debug("Processing user registration for: {}", userRegisterParam.getUsername());
}
```

### 9.2 日志格式规范
- 使用占位符 `{}` 而不是字符串拼接
- 关键操作记录INFO级别日志
- 异常情况记录ERROR级别日志
- 调试信息记录DEBUG级别日志

## 10. 安全编码规范

### 10.1 密码处理
```java
// 密码加密
user.setPassword(BCrypt.hashpw(user.getPassword(), BCrypt.gensalt()));

// 密码验证
if (!BCrypt.checkpw(passwordParam.getOldPassword(), user.getPassword())) {
    Exceptions.tr(ParamException.class, "error.param.invalid", "resource.user.password");
}
```

### 10.2 权限检查
```java
// 组织权限检查
securityManager.requireOrgOwner(orgId);

// 当前用户获取
User currentUser = securityManager.getCurrentUser();
```

## 11. 配置和常量管理

### 11.1 常量定义
```java
public class Const {
    public static final String TOKEN = "X-Access-Token";
    public static final String USER_DEFAULT_PSW = "123456";
}
```

### 11.2 配置属性
```java
@Value("${datart.user.active.send-mail:false}")
private boolean sendEmail;
```

## 12. 代码注释规范

### 12.1 类注释
```java
/*
 * Datart
 * <p>
 * Copyright 2021
 * <p>
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 */
```

### 12.2 方法注释
```java
/**
 * 新用户注册
 * 1: 校验用户名和邮箱不能重复
 * 2: 根据应用程序配置，决定是否为用户发送激活邮件
 * 3: 激活用户，为用户创建默认组织和角色
 *
 * @param userRegisterParam 用户注册信息
 * @return 注册是否成功
 */
```

## 13. 最佳实践

### 13.1 代码组织
- 单一职责原则：每个类和方法只负责一个功能
- 依赖注入：使用构造函数注入，避免字段注入
- 异常处理：统一使用项目定义的异常类
- 事务管理：合理使用 `@Transactional` 注解

### 13.2 性能优化
- 数据库查询优化：合理使用索引和缓存
- 分页查询：大数据量查询必须分页
- 缓存使用：合理配置MyBatis缓存

### 13.3 代码质量
- 参数校验：使用 `@Validated` 进行参数校验
- 空值检查：使用工具方法进行空值检查
- 资源管理：及时关闭资源，使用try-with-resources

## 14. 开发工具和插件

### 14.1 必需依赖
```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>

<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.11</version>
</dependency>
```

### 14.2 代码生成
- 使用MyBatis Generator生成基础Mapper
- 使用Lombok减少样板代码
- 统一使用UUID生成器生成ID

## 15. 测试规范

### 15.1 单元测试
```java
@SpringBootTest
class UserServiceTest {
    
    @Autowired
    private UserService userService;
    
    @Test
    void testRegister() {
        // 测试用户注册功能
    }
}
```

### 15.2 集成测试
- 使用 `@SpringBootTest` 进行集成测试
- 测试覆盖主要业务流程
- 使用测试数据库进行测试

## 16. 数据库操作规范

### 16.1 数据库表设计规范

#### 16.1.1 表命名规范
```sql
-- 业务表使用单数形式，小写字母，下划线分隔
CREATE TABLE `user` (...)
CREATE TABLE `organization` (...)
CREATE TABLE `access_log` (...)

-- 关联表使用 rel_ 前缀
CREATE TABLE `rel_role_user` (...)
CREATE TABLE `rel_user_organization` (...)
```

#### 16.1.2 字段设计规范
```sql
-- 主键统一使用 varchar(32)
`id` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,

-- 基础审计字段（继承自BaseEntity）
`create_by` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
`create_time` timestamp(0) NULL DEFAULT NULL,
`update_by` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
`update_time` timestamp(0) NULL DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP(0),

-- 状态字段
`status` tinyint(6) NOT NULL DEFAULT 1,

-- 布尔字段使用 tinyint(1) 或 tinyint(4)
`active` tinyint(1) NULL DEFAULT NULL,
`is_folder` tinyint(1) NULL DEFAULT NULL,

-- 配置字段统一使用 text 类型
`config` text CHARACTER SET utf8 COLLATE utf8_general_ci NULL,

-- 字符集和排序规则
CHARACTER SET utf8 COLLATE utf8_general_ci
```

#### 16.1.3 索引设计规范
```sql
-- 主键索引
PRIMARY KEY (`id`) USING BTREE,

-- 唯一索引
UNIQUE INDEX `username`(`username`) USING BTREE,
UNIQUE INDEX `org_name`(`name`, `org_id`) USING BTREE,

-- 普通索引
INDEX `org_id`(`org_id`) USING BTREE,
INDEX `view_id`(`view_id`) USING BTREE,

-- 复合唯一索引
UNIQUE INDEX `org_id`(`org_id`, `view_id`, `name`) USING BTREE,
```

#### 16.1.4 外键约束规范
```sql
-- 外键约束定义
CONSTRAINT `qrtz_triggers_ibfk_1` 
FOREIGN KEY (`SCHED_NAME`, `JOB_NAME`, `JOB_GROUP`) 
REFERENCES `QRTZ_JOB_DETAILS` (`SCHED_NAME`, `JOB_NAME`, `JOB_GROUP`) 
ON DELETE RESTRICT ON UPDATE RESTRICT
```

### 16.2 MyBatis Generator配置规范

#### 16.2.1 Generator配置文件
```xml
<generatorConfiguration>
    <context id="my" targetRuntime="MyBatis3">
        <!-- 关键字自动分隔 -->
        <property name="autoDelimitKeywords" value="true"/>
        <property name="beginningDelimiter" value="`"/>
        <property name="endingDelimiter" value="`"/>
        
        <!-- 自定义插件 -->
        <plugin type="datart.core.common.MybatisGeneratorPlugin">
        </plugin>

        <!-- 注释生成器 -->
        <commentGenerator>
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>

        <!-- 实体类生成配置 -->
        <javaModelGenerator targetPackage="datart.core.entity"
                            targetProject="./src/main/java">
            <property name="enableSubPackages" value="false"/>
            <property name="trimStrings" value="true"/>
            <property name="rootClass" value="datart.core.entity.BaseEntity"/>
        </javaModelGenerator>

        <!-- Mapper接口生成配置 -->
        <javaClientGenerator targetPackage="datart.core.mappers"
                             targetProject="./src/main/java"
                             type="ANNOTATEDMAPPER">
            <property name="enableSubPackages" value="false"/>
            <property name="useLegacyBuilder" value="false"/>
            <property name="rootInterface" value="datart.core.mappers.ext.CRUDMapper"/>
        </javaClientGenerator>
    </context>
</generatorConfiguration>
```

#### 16.2.2 表配置规范
```xml
<table tableName="user"
       enableCountByExample="false" 
       enableUpdateByExample="false"
       enableDeleteByExample="false" 
       enableSelectByExample="false"
       selectByExampleQueryId="false">
    <property name="ignoreQualifiersAtRuntime" value="true"/>
    <!-- 字段类型覆盖 -->
    <columnOverride column="active" javaType="java.lang.Boolean" jdbcType="TINYINT"/>
    <columnOverride column="config" javaType="java.lang.String" jdbcType="VARCHAR"/>
</table>
```

### 16.3 自定义Generator插件规范

#### 16.3.1 插件实现
```java
public class MybatisGeneratorPlugin extends PluginAdapter {

    @Override
    public boolean modelBaseRecordClassGenerated(TopLevelClass topLevelClass, 
                                               IntrospectedTable introspectedTable) {
        // 添加Lombok注解
        topLevelClass.addImportedType("lombok.Data");
        topLevelClass.addImportedType("lombok.EqualsAndHashCode");
        topLevelClass.addAnnotation("@Data");
        topLevelClass.addAnnotation("@EqualsAndHashCode(callSuper = true)");
        
        return true;
    }

    @Override
    public boolean modelFieldGenerated(Field field, TopLevelClass topLevelClass, 
                                     IntrospectedColumn introspectedColumn,
                                     IntrospectedTable introspectedTable,
                                     Plugin.ModelClassType modelClassType) {
        // 过滤BaseEntity中已有的字段
        return !Arrays.asList("id", "createBy", "createTime", "updateBy", "updateTime")
                     .contains(field.getName());
    }
}
```

### 16.4 Mapper接口编写规范

#### 16.4.1 基础Mapper接口
```java
@Mapper
public interface UserMapper {
    int deleteByPrimaryKey(String id);
    int insert(User record);
    int insertSelective(User record);
    User selectByPrimaryKey(String id);
    int updateByPrimaryKeySelective(User record);
    int updateByPrimaryKey(User record);
}
```

#### 16.4.2 扩展Mapper接口
```java
@Mapper
public interface UserMapperExt extends UserMapper {

    @Select({
        "SELECT ",
        "	u.*  ",
        "FROM ",
        "	`user` u  ",
        "WHERE ",
        "	u.username = #{username} or ",
        "	u.email = #{username}"
    })
    User selectByNameOrEmail(@Param("username") String username);

    @Update({
        "UPDATE `user` u " +
        "SET u.active = 1  " +
        "WHERE " +
        "	u.active = 0  " +
        "	AND u.id = #{userId}"
    })
    int updateToActiveById(String id);
}
```

### 16.5 SQL Provider规范

#### 16.5.1 动态SQL构建
```java
public class UserSqlProvider {
    
    public String insertSelective(User record) {
        SQL sql = new SQL();
        sql.INSERT_INTO("user");
        
        if (record.getId() != null) {
            sql.VALUES("id", "#{id,jdbcType=VARCHAR}");
        }
        if (record.getEmail() != null) {
            sql.VALUES("email", "#{email,jdbcType=VARCHAR}");
        }
        // 关键字字段使用反引号
        if (record.getPassword() != null) {
            sql.VALUES("`password`", "#{password,jdbcType=VARCHAR}");
        }
        
        return sql.toString();
    }

    public String updateByPrimaryKeySelective(User record) {
        SQL sql = new SQL();
        sql.UPDATE("user");
        
        if (record.getEmail() != null) {
            sql.SET("email = #{email,jdbcType=VARCHAR}");
        }
        if (record.getPassword() != null) {
            sql.SET("`password` = #{password,jdbcType=VARCHAR}");
        }
        
        sql.WHERE("id = #{id,jdbcType=VARCHAR}");
        return sql.toString();
    }
}
```

### 16.6 数据库迁移规范

#### 16.6.1 迁移文件命名
```
V2022.02.18__baseline.sql          # 基线版本
V2022.03.14__1.0.0.beta.3.sql     # 版本升级
R2022.03.14__1.0.0.beta.3.sql     # 回滚脚本
```

#### 16.6.2 迁移脚本结构
```sql
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for user
-- ----------------------------
DROP TABLE IF EXISTS `user`;
CREATE TABLE `user` (
  `id` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `email` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `username` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE INDEX `username`(`username`) USING BTREE,
  UNIQUE INDEX `email`(`email`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- 表结构变更
-- ----------------------------
ALTER TABLE `user`
    ADD COLUMN `active` tinyint(1) NULL DEFAULT NULL AFTER `password`;

ALTER TABLE `user`
    ADD INDEX `active`(`active`) USING BTREE;

SET FOREIGN_KEY_CHECKS = 1;
```

### 16.7 数据库操作最佳实践

#### 16.7.1 事务管理
```java
@Transactional
public boolean register(UserRegisterParam userRegisterParam) {
    // 数据库操作
    userMapper.insert(user);
    orgService.addUserToOrg(user.getId(), orgId);
    return true;
}

@Transactional(propagation = Propagation.MANDATORY)
public void initUser(User user) {
    // 必须在事务中执行
}
```

#### 16.7.2 批量操作
```java
// 批量插入
@Insert({
    "<script>",
    "INSERT INTO user (id, username, email) VALUES ",
    "<foreach collection='users' item='user' separator=','>",
    "(#{user.id}, #{user.username}, #{user.email})",
    "</foreach>",
    "</script>"
})
int batchInsert(@Param("users") List<User> users);
```

#### 16.7.3 分页查询
```java
@Select({
    "SELECT * FROM user ",
    "WHERE org_id = #{orgId} ",
    "ORDER BY create_time DESC ",
    "LIMIT #{offset}, #{limit}"
})
List<User> selectByPage(@Param("orgId") String orgId, 
                       @Param("offset") int offset, 
                       @Param("limit") int limit);
```

#### 16.7.4 缓存配置
```java
@Mapper
@CacheNamespace(flushInterval = 5 * 1000)
public interface FolderMapperExt extends FolderMapper {
    // 设置缓存刷新间隔为5秒
}

@Mapper
@CacheNamespaceRef(value = FolderMapperExt.class)
public interface DatachartMapperExt extends DatachartMapper {
    // 引用其他Mapper的缓存配置
}
```

### 16.8 数据库连接配置

#### 16.8.1 连接池配置
```properties
# 数据库连接配置
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/datart?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# 连接池配置
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

#### 16.8.2 MyBatis配置
```properties
# MyBatis配置
mybatis.configuration.map-underscore-to-camel-case=true
mybatis.configuration.use-generated-keys=true
mybatis.configuration.default-fetch-size=100
mybatis.configuration.default-statement-timeout=30
```

---

本规范基于Datart项目实际代码分析总结，旨在保持代码风格一致性和提高代码质量。开发人员应严格遵循此规范进行后端代码开发。
