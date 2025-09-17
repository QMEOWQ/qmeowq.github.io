# 短链接系统

## 开发工具

### 1. 数据库迁移工具 - golang-migrate

用于数据库版本管理和迁移

```bash
go install -tags postgres github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

### 2. SQL 转 Go 代码工具 - sqlc

将 SQL 语句转换为类型安全的 Go 代码

```bash
go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest
```

## 数据库环境

### 3. PostgreSQL 数据库（使用 Docker）

主数据库，用于存储短链接数据

```bash
docker run --name postgres-url \
  -e POSTGRES_USER=urltest \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=urldb \
  -p 5432:5432 \
  -d postgres

若本地已安装postgresql,请将端口映射改为
-p 5433:5432（总之不要用5432端口）
```

**连接信息：**

- 主机：localhost
- 端口：5432
- 用户名：urltest
- 密码：password
- 数据库名：urldb

### 4. Redis 缓存（使用 Docker）

用于缓存和提高性能

```bash
docker run --name redis \
  -p 6379:6379 \
  -d redis
```

**连接信息：**

- 主机：localhost
- 端口：6379

## 系统要求

### 基础环境

- **Go 版本**：1.19 或更高版本
- **Docker**：用于运行数据库和缓存服务
- **Git**：版本控制

### 开发工具推荐

- **IDE**：VS Code、GoLand 或其他支持 Go 的编辑器
- **API 测试工具**：Postman、curl 或类似工具

## 快速启动

### 1. 安装开发工具

```bash
# 安装数据库迁移工具
go install -tags postgres github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# 安装SQL代码生成工具
go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest
```

### 2. 启动数据库服务

```bash
# 启动PostgreSQL
docker run --name postgres-url \
  -e POSTGRES_USER=lang \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=urldb \
  -p 5432:5432 \
  -d postgres

# 启动Redis
docker run --name redis \
  -p 6379:6379 \
  -d redis
```

### 3. 验证环境

```bash
# 检查PostgreSQL连接
docker exec -it postgres-url psql -U lang -d urldb -c "SELECT version();"

# 检查Redis连接
docker exec -it redis redis-cli ping
```

## 注意事项

1. **端口占用**：确保 5432（PostgreSQL）和 6379（Redis）端口未被占用
2. **Docker 权限**：确保 Docker 服务正在运行且有足够权限
3. **网络连接**：首次运行时需要下载 Docker 镜像，确保网络连接正常
4. **数据持久化**：以上命令未配置数据持久化，重启容器后数据会丢失。生产环境请配置数据卷

## 故障排除

### 常见问题

1. **端口被占用**：使用`netstat -an | grep :5432`检查端口占用情况
2. **Docker 容器启动失败**：检查 Docker 服务状态和日志
3. **数据库连接失败**：确认容器正在运行且端口映射正确

### 有用的命令

```bash
# 查看运行中的容器
docker ps

# 查看容器日志
docker logs postgres-url
docker logs redis

# 停止容器
docker stop postgres-url redis

# 删除容器
docker rm postgres-url redis
```

sqlc 所需命令及流程

```
启动 redis, postgres 容器后，运行

migrate create -ext=sql -dir=./database/migrate -seq init_schema

make migrate_up

sqlc init

设置sqlc.yaml 后 sqlc generate在 repo文件夹下

```

运行项目：

后端：

```
docker 启动对应postgres,redis 容器
go run main.go
```

![alt text](../images/2025-09-17-urlShortener-images/start-backend.png)

测试：

```
选用 vscode 中的 `postman` 和 `postgresql` 插件进行测试

【注】：Headers 中 Content-Type 默认设置为 text/plain, 这会导致 send 请求失败，需要设置为 application/json
```

![alt text](../images/2025-09-17-urlShortener-images/postman-test.png)

结构图：

系统架构图：

```mermaid
graph TB
    subgraph "Frontend Layer"
        FE[React Frontend]
    end

    subgraph "API Layer"
        Router[Echo Router]
        MW[Middleware]
        URLHandler[URL Handler]
        UserHandler[User Handler]
    end

    subgraph "Service Layer"
        URLService[URL Service]
        UserService[User Service]
    end

    subgraph "Repository Layer"
        DB[(PostgreSQL)]
        Redis[(Redis Cache)]
    end

    subgraph "External Services"
        Email[Email Sender]
    end

    subgraph "Utilities"
        JWT[JWT Token]
        Hash[Password Hash]
        ShortCode[Short Code Generator]
        Validator[Custom Validator]
        Logger[Logger]
    end

    FE --> Router
    Router --> MW
    MW --> URLHandler
    MW --> UserHandler
    URLHandler --> URLService
    UserHandler --> UserService
    URLService --> DB
    URLService --> Redis
    UserService --> DB
    UserService --> Redis
    UserService --> Email
    UserService --> JWT
    UserService --> Hash
    URLService --> ShortCode
    Router --> Validator
    Router --> Logger
```

url 短链接核心流程：

```mermaid
sequenceDiagram
    participant Client
    participant Router
    participant URLHandler
    participant URLService
    participant ShortCode
    participant DB
    participant Redis

    Client->>Router: POST /api/urls (原始URL)
    Router->>URLHandler: CreateURL()
    URLHandler->>URLService: CreateURL()
    URLService->>ShortCode: Generate()
    ShortCode-->>URLService: 短码
    URLService->>DB: 保存URL映射
    URLService->>Redis: 缓存URL映射
    URLService-->>URLHandler: URL对象
    URLHandler-->>Router: JSON响应
    Router-->>Client: 短链接URL

    Note over Client,Redis: 访问短链接流程
    Client->>Router: GET /:shortCode
    Router->>URLHandler: RedirectURL()
    URLHandler->>URLService: GetURL()
    URLService->>Redis: 查询缓存
    alt 缓存命中
        Redis-->>URLService: URL数据
    else 缓存未命中
        URLService->>DB: 查询数据库
        DB-->>URLService: URL数据
        URLService->>Redis: 更新缓存
    end
    URLService-->>URLHandler: 原始URL
    URLHandler-->>Router: 重定向响应
    Router-->>Client: 302重定向
```

用户认证流程

```mermaid
sequenceDiagram
    participant Client
    participant Router
    participant UserHandler
    participant UserService
    participant Hash
    participant JWT
    participant Email
    participant DB
    participant Redis

    Note over Client,Redis: 用户注册流程
    Client->>Router: POST /api/users/register
    Router->>UserHandler: Register()
    UserHandler->>UserService: Register()
    UserService->>Hash: HashPassword()
    Hash-->>UserService: 加密密码
    UserService->>Email: SendVerificationCode()
    Email-->>UserService: 发送成功
    UserService->>Redis: 存储验证码
    UserService->>DB: 保存用户信息
    UserService-->>UserHandler: 注册结果
    UserHandler-->>Router: JSON响应
    Router-->>Client: 注册成功

    Note over Client,Redis: 用户登录流程
    Client->>Router: POST /api/users/login
    Router->>UserHandler: Login()
    UserHandler->>UserService: Login()
    UserService->>DB: 查询用户
    UserService->>Hash: VerifyPassword()
    Hash-->>UserService: 验证结果
    UserService->>JWT: GenerateToken()
    JWT-->>UserService: JWT Token
    UserService-->>UserHandler: 登录结果
    UserHandler-->>Router: JSON响应
    Router-->>Client: JWT Token
```

组件依赖关系图：

```mermaid
graph LR
    subgraph "Application Layer"
        App[Application]
        Echo[Echo Server]
        Router[Router]
    end

    subgraph "Handler Layer"
        URLHandler[URL Handler]
        UserHandler[User Handler]
    end

    subgraph "Service Layer"
        URLService[URL Service]
        UserService[User Service]
    end

    subgraph "Repository Layer"
        DBRepo[Database Repository]
        CacheRepo[Cache Repository]
    end

    subgraph "Infrastructure"
        Config[Config]
        DB[(Database)]
        Redis[(Redis)]
        Email[Email Service]
    end

    subgraph "Utilities"
        JWT[JWT]
        Hash[Hasher]
        ShortCode[ShortCode]
        Validator[Validator]
        Logger[Logger]
    end

    App --> Echo
    App --> Router
    App --> Config
    Router --> URLHandler
    Router --> UserHandler
    URLHandler --> URLService
    UserHandler --> UserService
    URLService --> DBRepo
    URLService --> CacheRepo
    URLService --> ShortCode
    UserService --> DBRepo
    UserService --> CacheRepo
    UserService --> JWT
    UserService --> Hash
    UserService --> Email
    DBRepo --> DB
    CacheRepo --> Redis
    Echo --> Validator
    Echo --> Logger
```

数据流图：

```mermaid
flowchart TD
    A[HTTP请求] --> B{路由匹配}
    B -->|URL相关| C[URL Handler]
    B -->|用户相关| D[User Handler]

    C --> E[URL Service]
    D --> F[User Service]

    E --> G{操作类型}
    G -->|创建| H[生成短码]
    G -->|查询| I[缓存查询]
    G -->|统计| J[更新访问量]

    F --> K{操作类型}
    K -->|注册| L[密码加密]
    K -->|登录| M[密码验证]
    K -->|验证| N[邮件发送]

    H --> O[数据库存储]
    I --> P{缓存命中?}
    P -->|是| Q[返回结果]
    P -->|否| R[数据库查询]
    R --> S[更新缓存]
    S --> Q

    L --> T[用户创建]
    M --> U[JWT生成]
    N --> V[验证码存储]

    O --> Q
    T --> Q
    U --> Q
    V --> Q
    J --> Q

    Q --> W[HTTP响应]
```

中间件执行过程：

```mermaid
sequenceDiagram
    participant Client
    participant Logger
    participant JWT
    participant Handler
    participant Service

    Client->>Logger: HTTP请求
    Logger->>Logger: 记录请求日志
    Logger->>JWT: 继续处理

    alt 需要认证的路由
        JWT->>JWT: 验证Token
        alt Token有效
            JWT->>Handler: 继续处理
        else Token无效
            JWT-->>Client: 401 Unauthorized
        end
    else 公开路由
        JWT->>Handler: 直接处理
    end

    Handler->>Service: 业务逻辑处理
    Service-->>Handler: 处理结果
    Handler-->>JWT: 响应
    JWT-->>Logger: 响应
    Logger->>Logger: 记录响应日志
    Logger-->>Client: HTTP响应
```
