# Datart 后端代码编写规范

## 1. 项目概述

Datart 是一个基于 Spring Boot 的数据可视化平台，采用分层架构设计，包含控制器层（Controller）、服务层（Service）、数据访问层（Mapper）和实体层（Entity）。

### 技术栈
- **框架**: Spring Boot 2.x
- **数据库**: MySQL + MyBatis
- **安全**: Spring Security + JWT
- **文档**: Swagger/OpenAPI
- **构建工具**: Maven
- **日志**: Logback + SLF4J

## 2. 项目结构规范

### 2.1 包结构
```
datart.server
├── controller/          # 控制器层
├── service/            # 服务接口层
│   └── impl/          # 服务实现层
├── base/              # 基础类和DTO
│   └── dto/          # 数据传输对象
├── config/            # 配置类
└── common/            # 通用工具类

datart.core
├── entity/            # 实体类
├── mappers/           # MyBatis Mapper接口
│   └── ext/          # 扩展Mapper接口
└── base/             # 核心基础类
```

### 2.2 文件命名规范
- **控制器**: `XxxController.java`
- **服务接口**: `XxxService.java`
- **服务实现**: `XxxServiceImpl.java`
- **实体类**: `Xxx.java`
- **Mapper接口**: `XxxMapper.java`
- **扩展Mapper**: `XxxMapperExt.java`
- **DTO类**: `XxxDto.java`、`XxxCreateParam.java`、`XxxUpdateParam.java`

## 3. 代码结构规范

### 3.1 控制器层（Controller）

#### 基本结构
```java
@Api(tags = "模块名称")
@Slf4j
@RestController
@RequestMapping(value = "/api-path")
@Validated
public class XxxController extends BaseController {

    private final XxxService xxxService;

    public XxxController(XxxService xxxService) {
        this.xxxService = xxxService;
    }

    // API方法
}
```

#### 注解使用规范
- **类级别注解**:
  - `@Api(tags = "模块描述")`: Swagger文档标签
  - `@Slf4j`: 日志支持
  - `@RestController`: REST控制器
  - `@RequestMapping`: 基础路径映射
  - `@Validated`: 参数验证支持

- **方法级别注解**:
  - `@ApiOperation("操作描述")`: API操作描述
  - `@GetMapping`、`@PostMapping`、`@PutMapping`、`@DeleteMapping`: HTTP方法映射
  - `@Valid`: 参数验证
  - `@PathVariable`、`@RequestParam`、`@RequestBody`: 参数绑定

#### 示例代码
```java
@ApiOperation("创建指标定义")
@PostMapping
public ResponseData<MetricDefinitionDto> create(@Valid @RequestBody MetricDefinitionCreateParam createParam) {
    MetricDefinitionDto result = metricDefinitionService.create(createParam);
    return ResponseData.success(result);
}

@ApiOperation("获取指标定义详情")
@GetMapping("/retrieve/{id}")
public ResponseData<MetricDefinitionDto> retrieve(@ApiParam("指标ID") @PathVariable @NotBlank String id) {
    MetricDefinitionDto result = metricDefinitionService.getMetricDetail(id);
    return ResponseData.success(result);
}
```

#### 响应格式规范
- 统一使用 `ResponseData<T>` 包装返回结果
- 成功响应: `ResponseData.success(data)`
- 失败响应: `ResponseData.failure(message)`

### 3.2 服务层（Service）

#### 接口定义
```java
public interface XxxService extends BaseCRUDService<Entity, MapperExt> {
    
    // 业务方法声明
    List<XxxDto> listByOrgId(String orgId);
    
    XxxDto getDetail(String id);
    
    boolean validateExpression(String expression, String sourceId);
}
```

#### 实现类结构
```java
@Slf4j
@Service
public class XxxServiceImpl extends BaseService implements XxxService {

    private final XxxMapperExt xxxMapper;
    private final RelRoleResourceMapperExt rrrMapper;

    public XxxServiceImpl(XxxMapperExt xxxMapper, RelRoleResourceMapperExt rrrMapper) {
        this.xxxMapper = xxxMapper;
        this.rrrMapper = rrrMapper;
    }

    @Override
    public void requirePermission(Entity entity, int permission) {
        // 权限检查逻辑
    }

    @Override
    public RelRoleResourceMapperExt getRRRMapper() {
        return rrrMapper;
    }

    // 业务方法实现
}
```

#### 事务管理规范
- **创建、更新、删除操作**: 必须使用 `@Transactional`
- **查询操作**: 一般不需要事务注解
- **异常回滚**: 使用 `@Transactional(rollbackFor = Exception.class)`

```java
@Override
@Transactional(rollbackFor = Exception.class)
public MetricDefinitionDto create(MetricDefinitionCreateParam createParam) {
    // 参数验证
    if (createParam == null) {
        Exceptions.msg("message.param.empty");
    }
    
    // 业务逻辑
    MetricDefinition entity = new MetricDefinition();
    BeanUtils.copyProperties(createParam, entity);
    entity.setId(UUIDGenerator.generate());
    entity.setCreateBy(getCurrentUser().getId());
    entity.setCreateTime(new Date());
    
    // 数据库操作
    int result = xxxMapper.insert(entity);
    if (result <= 0) {
        Exceptions.msg("message.create.failed");
    }
    
    return entityToDto(entity);
}
```

#### 权限检查规范
```java
@Override
public void requirePermission(Entity entity, int permission) {
    if ((Const.CREATE | permission) == Const.CREATE) {
        return;
    }
    securityManager.requireAllPermissions(PermissionHelper.rolePermission(entity.getId(), permission));
}

private void requireEntityPermission(Entity entity, int permission) {
    if (entity == null) {
        Exceptions.msg("message.entity.not.found");
    }
    
    // 检查组织权限
    if (!securityManager.isOrgOwner(entity.getOrgId())) {
        Exceptions.msg("message.permission.denied");
    }
}
```

### 3.3 数据访问层（Mapper）

#### 基础Mapper继承
```java
@Mapper
public interface XxxMapperExt extends XxxMapper {
    
    // 扩展查询方法
}
```

#### SQL注解规范
```java
@Select({
    "SELECT id, name, code, expression, description, data_type, unit, owner, status, ",
    "org_id, source_id, create_by, create_time, update_by, update_time ",
    "FROM metric_definition ",
    "WHERE org_id = #{orgId} ",
    "ORDER BY create_time DESC"
})
List<MetricDefinition> selectByOrgId(@Param("orgId") String orgId);

@Select({
    "SELECT id, name, code, expression, description, data_type, unit, owner, status, ",
    "org_id, source_id, create_by, create_time, update_by, update_time ",
    "FROM metric_definition ",
    "WHERE org_id = #{orgId} ",
    "AND (name LIKE CONCAT('%', #{keyword}, '%') ",
    "OR code LIKE CONCAT('%', #{keyword}, '%') ",
    "OR description LIKE CONCAT('%', #{keyword}, '%')) ",
    "ORDER BY create_time DESC"
})
List<MetricDefinition> searchByKeyword(@Param("orgId") String orgId, @Param("keyword") String keyword);
```

### 3.4 实体类（Entity）

#### 基础实体结构
```java
@Data
@EqualsAndHashCode(callSuper = true)
public class MetricDefinition extends BaseEntity {

    /**
     * 指标名称
     */
    private String name;

    /**
     * 指标编码
     */
    private String code;

    /**
     * 指标表达式
     */
    private String expression;

    /**
     * 指标描述
     */
    private String description;

    /**
     * 数据类型
     */
    private String dataType;

    /**
     * 单位
     */
    private String unit;

    /**
     * 负责人
     */
    private String owner;

    /**
     * 状态：DRAFT-草稿，PUBLISHED-已发布，ARCHIVED-已归档
     */
    private String status;

    /**
     * 组织ID
     */
    private String orgId;

    /**
     * 关联数据源ID
     */
    private String sourceId;
}
```

#### BaseEntity结构
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

### 3.5 DTO类规范

#### 数据传输对象
```java
@Data
public class MetricDefinitionDto {
    
    private String id;
    private String name;
    private String code;
    private String expression;
    private String description;
    private String dataType;
    private String unit;
    private String owner;
    private String ownerName;
    private String status;
    private String orgId;
    private String sourceId;
    private String sourceName;
    private Integer usageCount;
    private String createBy;
    private String createByName;
    private Date createTime;
    private String updateBy;
    private String updateByName;
    private Date updateTime;
}
```

#### 参数对象
```java
@Data
public class MetricDefinitionCreateParam {
    
    @NotBlank(message = "指标名称不能为空")
    private String name;
    
    @NotBlank(message = "指标编码不能为空")
    private String code;
    
    @NotBlank(message = "指标表达式不能为空")
    private String expression;
    
    private String description;
    
    @NotBlank(message = "数据类型不能为空")
    private String dataType;
    
    private String unit;
    
    @NotBlank(message = "负责人不能为空")
    private String owner;
    
    @NotBlank(message = "组织ID不能为空")
    private String orgId;
    
    private String sourceId;
}
```

## 4. 异常处理规范

### 4.1 全局异常处理
```java
@Slf4j
@ControllerAdvice
public class WebExceptionHandler {

    @ResponseBody
    @ResponseStatus(code = HttpStatus.UNAUTHORIZED)
    @ExceptionHandler(AuthException.class)
    public ResponseData<String> exceptionHandler(AuthException e) {
        ResponseData.ResponseDataBuilder<String> builder = ResponseData.builder();
        return builder.success(false)
                .message(e.getMessage())
                .errCode(e.getErrCode())
                .build();
    }

    @ResponseBody
    @ResponseStatus(code = HttpStatus.BAD_REQUEST)
    @ExceptionHandler(BindException.class)
    public ResponseData<String> validateException(BindException e) {
        ResponseData.ResponseDataBuilder<String> builder = ResponseData.builder();
        return builder.success(false)
                .message(e.getBindingResult().getFieldErrors().stream()
                        .map(error -> error.getField() + ":" + error.getDefaultMessage())
                        .collect(Collectors.toList()).toString())
                .build();
    }
}
```

### 4.2 业务异常处理
```java
// 参数验证
if (!StringUtils.hasText(createParam.getName())) {
    Exceptions.msg("message.metric.name.empty");
}

// 业务规则验证
if (checkCodeExists(orgId, createParam.getCode(), null)) {
    Exceptions.msg("message.metric.code.exists", createParam.getCode());
}

// 权限检查
if (!securityManager.isOrgOwner(orgId)) {
    Exceptions.msg("message.permission.denied");
}
```

## 5. 数据库操作规范

### 5.1 查询规范
- 使用参数化查询防止SQL注入
- 合理使用索引，避免全表扫描
- 分页查询使用统一的分页参数

### 5.2 事务规范
- 事务边界明确，避免长事务
- 读操作不开启事务
- 写操作必须开启事务
- 异常时自动回滚

### 5.3 数据验证
- 主键使用UUID生成器
- 必填字段进行非空验证
- 业务唯一性验证
- 数据格式验证

## 6. 安全规范

### 6.1 权限控制
```java
// 组织级权限检查
if (!securityManager.isOrgOwner(entity.getOrgId())) {
    Exceptions.msg("message.permission.denied");
}

// 资源级权限检查
securityManager.requireAllPermissions(PermissionHelper.rolePermission(entity.getId(), permission));
```

### 6.2 数据过滤
```java
// 根据用户权限过滤数据
return metricDefinitions.stream()
        .filter(metric -> hasPermission(metric.getOrgId()))
        .map(this::entityToDto)
        .collect(Collectors.toList());
```

## 7. 日志规范

### 7.1 日志级别
- **ERROR**: 系统错误、异常信息
- **WARN**: 警告信息、业务异常
- **INFO**: 关键业务操作、系统启动信息
- **DEBUG**: 调试信息、详细执行过程

### 7.2 日志格式
```java
@Slf4j
public class XxxServiceImpl {
    
    public void someMethod() {
        log.info("开始执行业务操作，参数: {}", param);
        
        try {
            // 业务逻辑
            log.debug("执行步骤1完成");
        } catch (Exception e) {
            log.error("业务操作执行失败", e);
            throw e;
        }
        
        log.info("业务操作执行成功");
    }
}
```

## 8. 性能优化规范

### 8.1 数据库优化
- 合理使用索引
- 避免N+1查询问题
- 使用批量操作
- 合理的分页大小

### 8.2 缓存策略
- 热点数据缓存
- 查询结果缓存
- 缓存失效策略

### 8.3 异步处理
- 耗时操作异步执行
- 消息队列处理
- 定时任务调度

## 9. 测试规范

### 9.1 单元测试
- 服务层方法测试
- 边界条件测试
- 异常情况测试

### 9.2 集成测试
- API接口测试
- 数据库操作测试
- 权限验证测试

## 10. 文档规范

### 10.1 代码注释
- 类和方法必须有JavaDoc注释
- 复杂业务逻辑添加行内注释
- 参数和返回值说明

### 10.2 API文档
- 使用Swagger注解生成API文档
- 详细的参数说明
- 示例请求和响应

## 11. 版本控制规范

### 11.1 提交规范
- 提交信息清晰明确
- 单次提交功能单一
- 代码审查通过后合并

### 11.2 分支管理
- 功能开发使用feature分支
- 发布使用release分支
- 紧急修复使用hotfix分支

## 12. 部署规范

### 12.1 环境配置
- 开发、测试、生产环境分离
- 配置文件外部化
- 敏感信息加密存储

### 12.2 监控告警
- 应用性能监控
- 错误日志监控
- 业务指标监控

---

本规范文档基于Datart项目的实际代码结构和最佳实践制定，旨在确保代码质量、可维护性和团队协作效率。所有开发人员应严格遵循此规范进行后端代码开发。