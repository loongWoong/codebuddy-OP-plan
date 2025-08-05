# Datart 指标管理功能集成指南

## 概述

本文档介绍了在 Datart 项目中新增的"指标管理"功能模块，该功能实现了指标的统一定义、管理和在图表配置中的复用。

## 功能特性

### 核心功能
- ✅ **指标定义管理**：支持创建、编辑、删除指标定义
- ✅ **指标状态管理**：支持草稿、发布、归档状态流转
- ✅ **指标使用追踪**：记录指标在图表、仪表板中的使用情况
- ✅ **权限控制**：基于 Datart 现有权限体系进行访问控制
- ✅ **图表集成**：在图表配置中可直接选择预定义指标

### 技术特性
- 🔧 **遵循项目规范**：严格按照 Datart 后端和前端编码规范实现
- 🔧 **数据库版本管理**：使用 Flyway 进行数据库迁移
- 🔧 **MyBatis 集成**：使用 MyBatis Generator 生成基础 CRUD 代码
- 🔧 **国际化支持**：提供中英文双语支持

## 数据库设计

### 核心表结构

#### metric_definition（指标定义表）
```sql
CREATE TABLE metric_definition (
    id VARCHAR(64) PRIMARY KEY,
    name VARCHAR(128) NOT NULL COMMENT '指标名称',
    code VARCHAR(128) NOT NULL COMMENT '指标编码',
    data_type VARCHAR(32) NOT NULL COMMENT '数据类型',
    unit VARCHAR(32) COMMENT '单位',
    expression TEXT NOT NULL COMMENT '指标表达式',
    description TEXT COMMENT '指标描述',
    source_id VARCHAR(64) COMMENT '数据源ID',
    status VARCHAR(32) DEFAULT 'DRAFT' COMMENT '状态',
    usage_count INT DEFAULT 0 COMMENT '使用次数',
    owner VARCHAR(64) COMMENT '负责人',
    create_by VARCHAR(64) NOT NULL,
    create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    update_by VARCHAR(64),
    update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_code (code),
    KEY idx_source_status (source_id, status),
    KEY idx_owner (owner)
);
```

#### metric_usage（指标使用记录表）
```sql
CREATE TABLE metric_usage (
    id VARCHAR(64) PRIMARY KEY,
    metric_id VARCHAR(64) NOT NULL COMMENT '指标ID',
    resource_type VARCHAR(32) NOT NULL COMMENT '资源类型',
    resource_id VARCHAR(64) NOT NULL COMMENT '资源ID',
    resource_name VARCHAR(256) COMMENT '资源名称',
    create_by VARCHAR(64) NOT NULL,
    create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    KEY idx_metric_id (metric_id),
    KEY idx_resource (resource_type, resource_id),
    FOREIGN KEY (metric_id) REFERENCES metric_definition(id) ON DELETE CASCADE
);
```

## 后端实现

### 项目结构
```
datart-master/
├── core/src/main/java/datart/core/
│   ├── entity/
│   │   ├── MetricDefinition.java          # 指标定义实体
│   │   └── MetricUsage.java               # 指标使用记录实体
│   └── mappers/
│       ├── MetricDefinitionMapper.java    # 基础Mapper
│       ├── MetricDefinitionSqlProvider.java
│       ├── MetricUsageMapper.java
│       ├── MetricUsageSqlProvider.java
│       └── ext/
│           ├── MetricDefinitionMapperExt.java  # 扩展Mapper
│           └── MetricUsageMapperExt.java
└── server/src/main/java/datart/server/
    ├── base/dto/
    │   ├── MetricDefinitionCreateParam.java   # 创建参数DTO
    │   ├── MetricDefinitionUpdateParam.java   # 更新参数DTO
    │   ├── MetricDefinitionDto.java           # 响应DTO
    │   └── MetricUsageDto.java
    ├── service/
    │   ├── MetricDefinitionService.java       # 服务接口
    │   └── impl/
    │       └── MetricDefinitionServiceImpl.java  # 服务实现
    └── controller/
        └── MetricDefinitionController.java    # REST控制器
```

### 关键API接口

#### 指标管理接口
```java
// 获取指标列表
GET /api/v1/metrics?status=PUBLISHED&sourceId={sourceId}&page=1&size=20

// 创建指标
POST /api/v1/metrics
{
  "name": "用户数量",
  "code": "user_count",
  "dataType": "INTEGER",
  "unit": "个",
  "expression": "COUNT(DISTINCT user_id)",
  "description": "统计用户总数",
  "sourceId": "source_123"
}

// 更新指标
PUT /api/v1/metrics/{id}

// 删除指标
DELETE /api/v1/metrics/{id}

// 发布指标
PUT /api/v1/metrics/{id}/publish

// 归档指标
PUT /api/v1/metrics/{id}/archive

// 获取指标使用记录
GET /api/v1/metrics/{id}/usage
```

## 前端实现

### 项目结构
```
datart-master/frontend/src/
├── app/
│   ├── types/
│   │   └── MetricDefinition.ts            # 类型定义
│   ├── services/
│   │   └── metricDefinitionService.ts     # API服务
│   ├── components/
│   │   └── MetricSelector/                # 指标选择器组件
│   │       └── index.tsx
│   └── pages/
│       ├── MetricManagementPage/          # 指标管理页面
│       │   ├── index.tsx                  # 主页面
│       │   ├── MetricDefinitionForm.tsx   # 表单组件
│       │   ├── MetricDefinitionDetail.tsx # 详情组件
│       │   └── Loadable.ts               # 懒加载配置
│       └── ChartWorkbenchPage/components/ChartOperationPanel/components/
│           └── MetricFieldPanel.tsx       # 图表配置集成
└── locales/
    ├── zh-CN/
    │   └── metric.json                    # 中文国际化
    └── en-US/
        └── metric.json                    # 英文国际化
```

### 核心组件

#### 1. MetricSelector（指标选择器）
```typescript
import { MetricSelector } from 'app/components/MetricSelector';

// 在图表配置中使用
<MetricSelector
  value={selectedMetricId}
  onChange={(metricId, metric) => {
    // 处理指标选择
  }}
  sourceId={currentSourceId}
  placeholder="请选择指标"
/>
```

#### 2. MetricManagementPage（指标管理页面）
- 指标列表展示和搜索
- 指标创建、编辑、删除
- 指标状态管理（发布、归档）
- 指标使用情况查看

#### 3. MetricFieldPanel（图表配置集成）
- 在图表配置面板中集成指标选择
- 支持从指标库选择预定义指标
- 自动应用指标的聚合逻辑

## 使用指南

### 1. 创建指标

1. 访问指标管理页面：`/metrics`
2. 点击"新建指标"按钮
3. 填写指标信息：
   - **指标名称**：用户友好的显示名称
   - **指标编码**：唯一标识符，建议使用下划线命名
   - **数据类型**：INTEGER、DECIMAL、STRING等
   - **单位**：如"个"、"元"、"%"等
   - **指标表达式**：SQL表达式，如`COUNT(DISTINCT user_id)`
   - **指标描述**：详细说明指标含义和计算逻辑
   - **数据源**：选择对应的数据源
4. 保存为草稿或直接发布

### 2. 在图表中使用指标

1. 进入图表配置页面
2. 在字段配置面板中找到"指标字段"区域
3. 从下拉列表中选择预定义的指标
4. 指标会自动应用相应的聚合函数和计算逻辑
5. 完成图表配置并保存

### 3. 指标生命周期管理

- **草稿状态**：指标创建后的初始状态，可以编辑和删除
- **发布状态**：指标可以在图表配置中使用
- **归档状态**：指标不再可用，但保留历史数据

### 4. 权限控制

指标管理功能复用 Datart 现有的权限体系：
- **查看权限**：可以查看指标列表和详情
- **编辑权限**：可以创建、编辑、删除指标
- **管理权限**：可以发布、归档指标

## 集成步骤

### 1. 数据库迁移
```bash
# 执行数据库迁移脚本
# 脚本位置：server/src/main/resources/db/migration/V2024.01.15__metric_management.sql
```

### 2. 后端配置
确保以下配置正确：
- MyBatis Mapper 扫描路径包含新增的 Mapper
- Spring Boot 组件扫描包含新增的 Service 和 Controller

### 3. 前端路由配置
在主路由配置中添加指标管理页面路由：
```typescript
// 在路由配置中添加
{
  path: '/metrics',
  component: MetricManagementPage,
  // 其他路由配置...
}
```

### 4. 菜单配置
在主菜单中添加指标管理入口：
```typescript
// 在菜单配置中添加
{
  key: 'metrics',
  title: '指标管理',
  icon: 'database',
  path: '/metrics',
  // 其他菜单配置...
}
```

## 扩展建议

### 短期扩展（P1）
- **指标表达式校验**：集成 SQL 语法校验器
- **指标预览功能**：支持指标表达式的数据预览
- **批量操作**：支持批量发布、归档指标

### 中期扩展（P2）
- **指标血缘分析**：展示指标的上下游依赖关系
- **指标变更影响分析**：分析指标变更对现有图表的影响
- **指标使用统计**：提供指标使用频率和趋势分析

### 长期扩展（P3）
- **指标自动调度**：支持指标数据的定时刷新
- **指标异常监控**：监控指标数据的异常波动
- **指标市场**：构建企业级指标共享平台

## 注意事项

1. **编码规范**：严格遵循项目现有的编码规范
2. **数据一致性**：删除指标前需检查使用情况
3. **性能考虑**：大量指标时需要考虑分页和缓存
4. **权限安全**：确保指标访问权限的正确控制
5. **向后兼容**：新功能不影响现有图表功能

## 测试建议

### 单元测试
- Service 层业务逻辑测试
- Mapper 层数据访问测试
- Controller 层接口测试

### 集成测试
- 指标创建到使用的完整流程测试
- 权限控制测试
- 数据一致性测试

### 前端测试
- 组件渲染测试
- 用户交互测试
- API 调用测试

## 总结

指标管理功能为 Datart 提供了统一的指标定义和管理能力，通过与现有图表功能的深度集成，实现了指标的标准化和复用，提升了数据分析的效率和一致性。

该功能严格遵循 Datart 项目的技术架构和编码规范，确保了代码质量和系统稳定性。通过模块化的设计，为后续功能扩展提供了良好的基础。