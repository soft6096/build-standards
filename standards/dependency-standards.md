# 依赖管理规范 (Dependency Standards)

## 适用范围

添加/审查项目依赖、解决依赖冲突、设计依赖层级时加载。

## 强制规则

### 1. 依赖引入原则

- 加依赖前问：项目真需要吗？（一个方法可用 JDK 实现就不引库）
- 优先官方/主流库（Apache/Google/Spring 生态），小众库评估维护活跃度
- 同职责只选一个：JSON 库（Jackson/Gson 选一）、工具库（Guava/Hutool 选一），禁并存
- 新增依赖记到 README/依赖说明，注明用途

### 2. 版本策略

- 版本号统一在父 pom `dependencyManagement` 声明，子模块只写 groupId/artifactId（见 maven 规范）
- 优先使用 BOM 管理版本（Spring Boot BOM 已覆盖大部分）
- 重大升级（跨大版本）评估兼容性 + 回归测试，不盲目升
- 安全漏洞（CVE）依赖：及时升补丁版本，记录升级原因

### 3. scope 规范

| scope | 用途 | 注意 |
|---|---|---|
| compile（默认） | 运行时需要 | 主流依赖 |
| provided | 容器提供（servlet-api、lombok） | 不打包进产物 |
| runtime | 仅运行需要（JDBC 驱动） | 编译不需要 |
| test | 仅测试（JUnit/Mockito/Testcontainers） | 禁 compile 误用 |
| optional | 可选特性依赖 | 传递依赖不传播 |

- 测试依赖必须 `test` scope，禁止 `compile` 引入 JUnit/Mockito 等
- Lombok 用 `provided` + `optional`（编译期注解处理，不打进产物）

```xml
<!-- 正例 -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>   <!-- 不传递给下游 -->
</dependency>

<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

### 4. 依赖冲突处理

- 冲突排查：`mvn dependency:tree` 定位重复版本，禁盲加 exclusion
- 处理顺序：统一版本（dependencyManagement 强制）→ 升级兼容 → 最后才 exclusion
- 多个传递依赖引同一库不同版本 → 父 pom 统一版本，禁散落 exclusion

```bash
mvn dependency:tree -Dincludes=com.google.guava
```

- 冲突修复后验证：构建 + 相关测试通过

### 4.5 数据库驱动版本（MySQL 实战红线）

- **MySQL 8 服务器必须配 Connector/J 8.x**（Spring Boot 3.x 已通过 BOM 管理版本，无需手写），**禁止 5.1.x 旧驱动**：旧驱动配 MySQL 8 有一串兼容问题——`characterEncoding=utf8mb4` 报 `Unsupported character encoding`、时区/加密协议（caching_sha2_password）不兼容、emoji 乱码
- 驱动版本统一由 Spring Boot BOM 管理，不手写版本号；确需覆盖时在父 pom `dependencyManagement` 声明

```xml
<!-- ✅ 正确：由 Spring Boot BOM 管理版本，不写版本号 -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- ❌ 错误：手写 5.1.x 旧驱动版本（配 MySQL 8 会踩字符集/时区/加密协议坑） -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.49</version>
</dependency>
```

### 5. 禁止事项

- ❌ 传递依赖带进来的多余库不清理（体积 + 安全面）
- ❌ 依赖版本写 `LATEST`/`RELEASE`（构建不可复现）
- ❌ 同一库多版本共存（类加载混乱，运行时诡异 bug）
- ❌ 无脑 exclusion 压冲突（掩盖问题，不解决）
- ❌ 测试框架进 compile scope

## 自检清单

- [ ] 版本统一在父 pom 管理
- [ ] scope 正确（test 依赖不进 compile）
- [ ] 无 LATEST/RELEASE 版本
- [ ] 无同一库多版本
- [ ] 冲突已通过 dependencyManagement 解决，非盲目 exclusion
- [ ] 新依赖有用途说明
- [ ] MySQL 8 → Connector/J 8.x（BOM 管理），无 5.1 旧驱动
