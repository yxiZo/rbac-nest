# NestJS JWT 认证与 RBAC 系统实现计划

> 📅 文档创建时间: 2026-01-14
> 📦 项目: rbac-nest
> 🎯 目标: 实现基于 JWT 的用户认证和基于角色的访问控制系统

---

## 📋 目录

1. [项目概述](#项目概述)
2. [技术栈选型](#技术栈选型)
3. [系统架构设计](#系统架构设计)
4. [数据库设计](#数据库设计)
5. [模块架构](#模块架构)
6. [认证流程设计](#认证流程设计)
7. [JWT 配置与安全](#jwt-配置与安全)
8. [守卫与装饰器](#守卫与装饰器)
9. [DTOs 与数据验证](#dtos-与数据验证)
10. [API 端点设计](#api-端点设计)
11. [错误处理](#错误处理)
12. [中间件与拦截器](#中间件与拦截器)
13. [数据库迁移与种子数据](#数据库迁移与种子数据)
14. [测试策略](#测试策略)
15. [配置管理](#配置管理)
16. [安全最佳实践](#安全最佳实践)
17. [实施计划](#实施计划)
18. [依赖包清单](#依赖包清单)

---

## 🎯 项目概述

### 当前状态
- ✅ NestJS 基础项目骨架已创建
- ❌ 无数据库配置
- ❌ 无用户认证模块
- ❌ 无授权机制
- ❌ 无 RBAC 实现

### 实现目标
- ✅ 用户注册与登录
- ✅ JWT Token 认证
- ✅ Access Token + Refresh Token 双令牌机制
- ✅ 基于角色的访问控制 (RBAC)
- ✅ 细粒度权限管理
- ✅ 安全的密码存储
- ✅ 完整的错误处理
- ✅ API 文档

---

## 🛠 技术栈选型

| 类别 | 技术选择 | 理由 |
|------|----------|------|
| **后端框架** | NestJS | ✅ 已选用 |
| **数据库** | PostgreSQL | 企业级，支持复杂查询，ACID 保证 |
| **ORM** | TypeORM | NestJS 官方推荐，文档完善 |
| **认证框架** | Passport.js | NestJS 官方集成，社区成熟 |
| **JWT 库** | @nestjs/jwt | 官方封装，简化使用 |
| **密码加密** | bcrypt | 业界标准，防止彩虹表攻击 |
| **数据验证** | class-validator + class-transformer | 装饰器风格，类型安全 |
| **配置管理** | @nestjs/config | 官方配置模块 |
| **API 文档** | Swagger (@nestjs/swagger) | 自动生成，交互式文档 |
| **缓存** | Redis (可选) | 高性能，存储 token 黑名单 |

### 可选技术对比

**数据库选择:**
- **PostgreSQL** ✅ 推荐 - 功能强大，扩展性好
- MySQL - 简单场景够用
- MongoDB - 无需复杂关系时可选

**ORM 选择:**
- **TypeORM** ✅ 推荐 - 成熟稳定
- Prisma - 类型安全更好，但生态较新
- MikroORM - 功能丰富，学习曲线陡峭

---

## 🏗 系统架构设计

```
┌─────────────────────────────────────────────────────────┐
│                     Client Application                   │
│              (Web/Mobile/Desktop Client)                 │
└───────────────────────┬─────────────────────────────────┘
                        │
                        │ HTTP Requests (JSON)
                        │ Authorization: Bearer {token}
                        ▼
┌─────────────────────────────────────────────────────────┐
│                    NestJS Application                    │
│  ┌──────────────────────────────────────────────────┐  │
│  │             Middleware Layer                      │  │
│  │  - LoggerMiddleware                              │  │
│  │  - CORS                                          │  │
│  │  - Helmet (Security Headers)                     │  │
│  │  - Compression                                   │  │
│  └──────────────────────┬───────────────────────────┘  │
│                         │                               │
│  ┌──────────────────────▼───────────────────────────┐  │
│  │             Guards Layer                          │  │
│  │  - JwtAuthGuard (认证)                          │  │
│  │  - RolesGuard (角色授权)                        │  │
│  │  - PermissionsGuard (权限授权)                  │  │
│  └──────────────────────┬───────────────────────────┘  │
│                         │                               │
│  ┌──────────────────────▼───────────────────────────┐  │
│  │           Controller Layer                        │  │
│  │  - AuthController                                │  │
│  │  - UsersController                               │  │
│  │  - RolesController                               │  │
│  │  - PermissionsController                         │  │
│  └──────────────────────┬───────────────────────────┘  │
│                         │                               │
│  ┌──────────────────────▼───────────────────────────┐  │
│  │             Service Layer                         │  │
│  │  - AuthService (认证逻辑)                       │  │
│  │  - UsersService (用户管理)                      │  │
│  │  - RolesService (角色管理)                      │  │
│  │  - PermissionsService (权限管理)                │  │
│  └──────────────────────┬───────────────────────────┘  │
│                         │                               │
│  ┌──────────────────────▼───────────────────────────┐  │
│  │          Repository Layer (TypeORM)               │  │
│  │  - UserRepository                                │  │
│  │  - RoleRepository                                │  │
│  │  - PermissionRepository                          │  │
│  └──────────────────────┬───────────────────────────┘  │
└────────────────────────┼─────────────────────────────┘
                         │
                         │ SQL Queries
                         ▼
         ┌───────────────────────────────┐
         │      PostgreSQL Database       │
         │  - users                      │
         │  - roles                      │
         │  - permissions                │
         │  - user_roles                 │
         │  - role_permissions           │
         └───────────────────────────────┘
```

---

## 🗄 数据库设计

### ER 图

```
┌─────────────────────┐
│       User          │
│─────────────────────│
│ id (UUID, PK)       │
│ username (unique)   │
│ email (unique)      │
│ password (hashed)   │
│ isActive            │
│ createdAt           │
│ updatedAt           │
└──────────┬──────────┘
           │
           │ Many-to-Many
           ▼
    ┌──────────────┐
    │  user_roles  │
    │──────────────│
    │ userId (FK)  │
    │ roleId (FK)  │
    └──────┬───────┘
           │
           │
┌──────────▼──────────┐
│       Role          │
│─────────────────────│
│ id (INT, PK)        │
│ name (unique)       │
│ description         │
│ createdAt           │
└──────────┬──────────┘
           │
           │ Many-to-Many
           ▼
  ┌─────────────────────┐
  │  role_permissions   │
  │─────────────────────│
  │ roleId (FK)         │
  │ permissionId (FK)   │
  └──────┬──────────────┘
         │
         │
┌────────▼────────────┐
│    Permission       │
│─────────────────────│
│ id (INT, PK)        │
│ name (unique)       │
│ resource            │
│ action (enum)       │
│ description         │
│ createdAt           │
└─────────────────────┘
```

### 表结构详细定义

#### 1. users 表

```sql
CREATE TABLE users (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  username          VARCHAR(50) UNIQUE NOT NULL,
  email             VARCHAR(100) UNIQUE NOT NULL,
  password          VARCHAR(255) NOT NULL, -- bcrypt hashed
  is_active         BOOLEAN DEFAULT true,
  created_at        TIMESTAMP DEFAULT NOW(),
  updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
```

**字段说明:**
- `id`: UUID 主键，比自增 ID 更安全
- `username`: 用户名，3-20 字符，唯一
- `email`: 邮箱地址，唯一
- `password`: bcrypt 加密后的密码 (60 字符)
- `is_active`: 账户状态，支持禁用用户
- `created_at/updated_at`: 审计字段

---

#### 2. roles 表

```sql
CREATE TABLE roles (
  id              SERIAL PRIMARY KEY,
  name            VARCHAR(50) UNIQUE NOT NULL,
  description     TEXT,
  created_at      TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_roles_name ON roles(name);
```

**预定义角色:**
- `admin` - 系统管理员，拥有所有权限
- `moderator` - 内容审核员，部分管理权限
- `user` - 普通用户，基础权限

---

#### 3. permissions 表

```sql
CREATE TABLE permissions (
  id              SERIAL PRIMARY KEY,
  name            VARCHAR(100) UNIQUE NOT NULL, -- 格式: resource:action
  resource        VARCHAR(50) NOT NULL,         -- 资源类型: user, role, post 等
  action          VARCHAR(20) NOT NULL,         -- 操作: create, read, update, delete
  description     TEXT,
  created_at      TIMESTAMP DEFAULT NOW(),

  CONSTRAINT chk_action CHECK (action IN ('create', 'read', 'update', 'delete'))
);

CREATE INDEX idx_permissions_resource ON permissions(resource);
CREATE INDEX idx_permissions_action ON permissions(action);
```

**权限命名规范:**
- `user:create` - 创建用户
- `user:read` - 读取用户信息
- `user:update` - 更新用户
- `user:delete` - 删除用户
- `role:assign` - 分配角色

---

#### 4. user_roles 关联表

```sql
CREATE TABLE user_roles (
  user_id   UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role_id   INT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,

  PRIMARY KEY (user_id, role_id)
);

CREATE INDEX idx_user_roles_user ON user_roles(user_id);
CREATE INDEX idx_user_roles_role ON user_roles(role_id);
```

---

#### 5. role_permissions 关联表

```sql
CREATE TABLE role_permissions (
  role_id         INT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
  permission_id   INT NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,

  PRIMARY KEY (role_id, permission_id)
);

CREATE INDEX idx_role_permissions_role ON role_permissions(role_id);
CREATE INDEX idx_role_permissions_permission ON role_permissions(permission_id);
```

---

## 📦 模块架构

### 项目目录结构

```
src/
├── app.module.ts                    # 根模块
├── main.ts                          # 应用入口
│
├── auth/                            # 认证模块
│   ├── auth.module.ts
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── dto/
│   │   ├── register.dto.ts
│   │   ├── login.dto.ts
│   │   └── refresh-token.dto.ts
│   ├── strategies/
│   │   ├── jwt.strategy.ts
│   │   └── local.strategy.ts
│   └── guards/
│       ├── jwt-auth.guard.ts
│       └── local-auth.guard.ts
│
├── users/                           # 用户模块
│   ├── users.module.ts
│   ├── users.controller.ts
│   ├── users.service.ts
│   ├── entities/
│   │   └── user.entity.ts
│   └── dto/
│       ├── create-user.dto.ts
│       └── update-user.dto.ts
│
├── roles/                           # 角色模块
│   ├── roles.module.ts
│   ├── roles.controller.ts
│   ├── roles.service.ts
│   ├── entities/
│   │   └── role.entity.ts
│   ├── dto/
│   │   ├── create-role.dto.ts
│   │   └── assign-role.dto.ts
│   └── guards/
│       └── roles.guard.ts
│
├── permissions/                     # 权限模块
│   ├── permissions.module.ts
│   ├── permissions.controller.ts
│   ├── permissions.service.ts
│   ├── entities/
│   │   └── permission.entity.ts
│   ├── dto/
│   │   └── create-permission.dto.ts
│   └── guards/
│       └── permissions.guard.ts
│
├── database/                        # 数据库模块
│   ├── database.module.ts
│   ├── migrations/
│   │   └── 1705200000000-InitialSchema.ts
│   └── seeds/
│       ├── permissions.seed.ts
│       ├── roles.seed.ts
│       └── admin.seed.ts
│
├── config/                          # 配置模块
│   ├── app.config.ts
│   ├── database.config.ts
│   ├── jwt.config.ts
│   └── validation.schema.ts
│
├── common/                          # 通用模块
│   ├── decorators/
│   │   ├── public.decorator.ts
│   │   ├── current-user.decorator.ts
│   │   ├── roles.decorator.ts
│   │   └── permissions.decorator.ts
│   ├── filters/
│   │   └── http-exception.filter.ts
│   ├── interceptors/
│   │   ├── transform.interceptor.ts
│   │   └── timeout.interceptor.ts
│   ├── pipes/
│   │   └── validation.pipe.ts
│   ├── middleware/
│   │   └── logger.middleware.ts
│   └── exceptions/
│       ├── user-already-exists.exception.ts
│       ├── invalid-credentials.exception.ts
│       └── insufficient-permissions.exception.ts
│
└── types/                           # 类型定义
    └── express.d.ts                 # 扩展 Express Request 类型
```

---

### 模块依赖关系

```
AppModule
├── ConfigModule (全局)
├── DatabaseModule
│   └── TypeOrmModule
│
├── UsersModule
│   ├── imports: [TypeOrmModule.forFeature([User])]
│   └── exports: [UsersService]
│
├── PermissionsModule
│   ├── imports: [TypeOrmModule.forFeature([Permission])]
│   └── exports: [PermissionsService]
│
├── RolesModule
│   ├── imports: [
│   │     TypeOrmModule.forFeature([Role]),
│   │     PermissionsModule
│   │   ]
│   └── exports: [RolesService]
│
└── AuthModule
    ├── imports: [
    │     UsersModule,
    │     RolesModule,
    │     JwtModule,
    │     PassportModule
    │   ]
    └── providers: [
          AuthService,
          LocalStrategy,
          JwtStrategy
        ]
```

---

## 🔐 认证流程设计

### 1. 用户注册流程

```
Client                   Controller              Service                Database
  │                         │                       │                       │
  ├─ POST /auth/register ──▶│                       │                       │
  │  {username, email,      │                       │                       │
  │   password}             │                       │                       │
  │                         │                       │                       │
  │                         ├─ RegisterDto ────────▶│                       │
  │                         │   (validation)        │                       │
  │                         │                       │                       │
  │                         │                       ├─ Check username ─────▶│
  │                         │                       │   exists              │
  │                         │                       │◀──────────────────────┤
  │                         │                       │   (false)             │
  │                         │                       │                       │
  │                         │                       ├─ Check email ────────▶│
  │                         │                       │   exists              │
  │                         │                       │◀──────────────────────┤
  │                         │                       │   (false)             │
  │                         │                       │                       │
  │                         │                       ├─ Hash password        │
  │                         │                       │   (bcrypt, 10 rounds) │
  │                         │                       │                       │
  │                         │                       ├─ Create user ────────▶│
  │                         │                       │   with default role   │
  │                         │                       │◀──────────────────────┤
  │                         │                       │   (user saved)        │
  │                         │                       │                       │
  │                         │◀─ Success ────────────┤                       │
  │◀─ 201 Created ──────────┤                       │                       │
  │  {message: "注册成功"}  │                       │                       │
```

**关键点:**
- 密码使用 bcrypt 加密 (salt rounds: 10)
- 自动分配默认 "user" 角色
- 不返回密码等敏感信息
- 验证用户名和邮箱唯一性

---

### 2. 用户登录流程

```
Client                   Controller              Service                Database
  │                         │                       │                       │
  ├─ POST /auth/login ─────▶│                       │                       │
  │  {usernameOrEmail,      │                       │                       │
  │   password}             │                       │                       │
  │                         │                       │                       │
  │                         ├─ LoginDto ───────────▶│                       │
  │                         │   (validation)        │                       │
  │                         │                       │                       │
  │                         │                       ├─ Find user ──────────▶│
  │                         │                       │   by username/email   │
  │                         │                       │◀──────────────────────┤
  │                         │                       │   (user + roles +     │
  │                         │                       │    permissions)       │
  │                         │                       │                       │
  │                         │                       ├─ Compare password     │
  │                         │                       │   bcrypt.compare()    │
  │                         │                       │   ✓ Match             │
  │                         │                       │                       │
  │                         │                       ├─ Generate tokens      │
  │                         │                       │   - accessToken       │
  │                         │                       │     (15m exp)         │
  │                         │                       │   - refreshToken      │
  │                         │                       │     (7d exp)          │
  │                         │                       │                       │
  │                         │◀─ Tokens ─────────────┤                       │
  │◀─ 200 OK ───────────────┤                       │                       │
  │  {                      │                       │                       │
  │    accessToken: "...",  │                       │                       │
  │    refreshToken: "...", │                       │                       │
  │    user: {...}          │                       │                       │
  │  }                      │                       │                       │
```

**响应示例:**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "username": "johndoe",
      "email": "john@example.com",
      "roles": ["user"],
      "permissions": ["user:read"]
    }
  }
}
```

---

### 3. 访问受保护资源流程

```
Client                   Guard                   Service                Database
  │                         │                       │                       │
  ├─ GET /users/me ────────▶│                       │                       │
  │  Authorization:         │                       │                       │
  │  Bearer {accessToken}   │                       │                       │
  │                         │                       │                       │
  │                         ├─ Extract token        │                       │
  │                         │   from header         │                       │
  │                         │                       │                       │
  │                         ├─ Verify token ───────▶│                       │
  │                         │   jwt.verify()        │                       │
  │                         │                       │                       │
  │                         │                       ├─ Decode payload       │
  │                         │                       │   {sub, username,     │
  │                         │                       │    roles, ...}        │
  │                         │                       │                       │
  │                         │◀─ Valid ──────────────┤                       │
  │                         │                       │                       │
  │                         ├─ Attach user          │                       │
  │                         │   to request          │                       │
  │                         │                       │                       │
  │                         ├─ Pass to handler ─────▶                       │
  │                         │                       │                       │
  │◀─ 200 OK ───────────────┴───────────────────────┤                       │
  │  {user data}                                    │                       │
```

---

### 4. Token 刷新流程

```
Client                   Controller              Service                Cache/DB
  │                         │                       │                       │
  ├─ POST /auth/refresh ───▶│                       │                       │
  │  {refreshToken}         │                       │                       │
  │                         │                       │                       │
  │                         ├─ RefreshTokenDto ────▶│                       │
  │                         │                       │                       │
  │                         │                       ├─ Verify refresh ─────▶│
  │                         │                       │   token               │
  │                         │                       │◀──────────────────────┤
  │                         │                       │   (valid)             │
  │                         │                       │                       │
  │                         │                       ├─ Generate new         │
  │                         │                       │   accessToken         │
  │                         │                       │   (15m exp)           │
  │                         │                       │                       │
  │                         │◀─ New token ──────────┤                       │
  │◀─ 200 OK ───────────────┤                       │                       │
  │  {accessToken: "..."}   │                       │                       │
```

**刷新策略:**
- Refresh token 存储位置: HttpOnly Cookie (推荐) 或 LocalStorage
- Refresh token 过期后需重新登录
- 可选: 实现 refresh token rotation (每次刷新生成新的 refresh token)

---

### 5. 用户登出流程

```
Client                   Controller              Service                Cache/Redis
  │                         │                       │                       │
  ├─ POST /auth/logout ────▶│                       │                       │
  │  Authorization:         │                       │                       │
  │  Bearer {accessToken}   │                       │                       │
  │                         │                       │                       │
  │                         ├─ Extract token ───────▶│                       │
  │                         │                       │                       │
  │                         │                       ├─ Add to blacklist ───▶│
  │                         │                       │   (TTL = token exp)   │
  │                         │                       │◀──────────────────────┤
  │                         │                       │                       │
  │                         │◀─ Success ────────────┤                       │
  │◀─ 200 OK ───────────────┤                       │                       │
  │  {message: "已登出"}    │                       │                       │
```

**实现方式:**
1. **简单方式**: 客户端删除 token (无服务端验证)
2. **完整方式**: Token 黑名单 (需要 Redis)
   - 将 token 添加到黑名单
   - 设置 TTL = token 过期时间
   - 认证时检查黑名单

---

## 🔑 JWT 配置与安全

### JWT Payload 结构

```typescript
interface JwtPayload {
  sub: string;              // Subject: 用户 ID (UUID)
  username: string;         // 用户名
  email: string;            // 邮箱
  roles: string[];          // 角色列表 ["admin", "user"]
  permissions: string[];    // 权限列表 ["user:create", "post:read"]
  iat: number;              // Issued At: 签发时间戳
  exp: number;              // Expiration: 过期时间戳
}
```

**示例:**
```json
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "username": "johndoe",
  "email": "john@example.com",
  "roles": ["user", "moderator"],
  "permissions": ["user:read", "user:update", "post:create"],
  "iat": 1705200000,
  "exp": 1705200900
}
```

---

### 环境变量配置

**.env.example**
```bash
# 应用配置
NODE_ENV=development
PORT=3000
API_PREFIX=api/v1

# 数据库配置
DATABASE_TYPE=postgres
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=your_password
DATABASE_NAME=rbac_nest_db
DATABASE_SYNCHRONIZE=false  # 生产环境必须为 false
DATABASE_LOGGING=true

# JWT 配置
JWT_ACCESS_SECRET=your_super_secret_access_key_min_32_chars
JWT_REFRESH_SECRET=your_super_secret_refresh_key_min_32_chars
JWT_ACCESS_EXPIRATION=15m
JWT_REFRESH_EXPIRATION=7d

# CORS 配置
CORS_ORIGIN=http://localhost:3001
CORS_CREDENTIALS=true

# Redis 配置 (可选 - 用于 token 黑名单)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# 安全配置
BCRYPT_SALT_ROUNDS=10
MAX_LOGIN_ATTEMPTS=5
ACCOUNT_LOCKOUT_DURATION=900  # 秒 (15分钟)
```

---

### JWT 配置代码

**jwt.config.ts**
```typescript
import { registerAs } from '@nestjs/config';

export default registerAs('jwt', () => ({
  accessSecret: process.env.JWT_ACCESS_SECRET,
  refreshSecret: process.env.JWT_REFRESH_SECRET,
  accessExpiration: process.env.JWT_ACCESS_EXPIRATION || '15m',
  refreshExpiration: process.env.JWT_REFRESH_EXPIRATION || '7d',
}));
```

**在 AuthModule 中注册:**
```typescript
@Module({
  imports: [
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        secret: configService.get<string>('jwt.accessSecret'),
        signOptions: {
          expiresIn: configService.get<string>('jwt.accessExpiration'),
        },
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AuthModule {}
```

---

### 安全建议

#### 1. Secret 生成
```bash
# 使用 OpenSSL 生成强随机字符串
openssl rand -base64 64
```

#### 2. Token 过期时间建议

| Token 类型 | 推荐过期时间 | 使用场景 |
|-----------|-------------|---------|
| Access Token | 15分钟 - 1小时 | 一般应用 |
| Access Token | 5分钟 | 高安全性应用 (银行) |
| Refresh Token | 7天 | 一般应用 |
| Refresh Token | 30天 | 移动应用 |

#### 3. 算法选择

- **HS256** (HMAC + SHA256): 对称加密，简单快速，适合单服务器
- **RS256** (RSA + SHA256): 非对称加密，更安全，适合微服务

**使用 RS256 示例:**
```typescript
// 生成密钥对
// ssh-keygen -t rsa -b 4096 -m PEM -f jwtRS256.key
// openssl rsa -in jwtRS256.key -pubout -outform PEM -out jwtRS256.key.pub

JwtModule.register({
  privateKey: fs.readFileSync('jwtRS256.key', 'utf8'),
  publicKey: fs.readFileSync('jwtRS256.key.pub', 'utf8'),
  signOptions: {
    algorithm: 'RS256',
    expiresIn: '15m',
  },
})
```

#### 4. Token 存储最佳实践

| 存储位置 | 优点 | 缺点 | 推荐 |
|---------|------|------|------|
| LocalStorage | 简单，跨标签页共享 | 易受 XSS 攻击 | ❌ 不推荐 |
| SessionStorage | 简单，标签页隔离 | 易受 XSS 攻击 | ❌ 不推荐 |
| HttpOnly Cookie | 防 XSS | 需配置 CSRF 防护 | ✅ 推荐 |
| Memory (状态管理) | 安全 | 刷新页面丢失 | ⚠️ 需配合 refresh |

---

## 🛡 守卫与装饰器

### 1. JwtAuthGuard - JWT 认证守卫

**jwt-auth.guard.ts**
```typescript
import { Injectable, ExecutionContext } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { Reflector } from '@nestjs/core';
import { IS_PUBLIC_KEY } from '../decorators/public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    // 检查是否标记为公开路由
    const isPublic = this.reflector.getAllAndOverride<boolean>(
      IS_PUBLIC_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (isPublic) {
      return true; // 跳过认证
    }

    return super.canActivate(context);
  }
}
```

**使用方式:**
```typescript
@Controller('users')
@UseGuards(JwtAuthGuard)  // 应用到整个控制器
export class UsersController {
  @Get('profile')
  getProfile(@CurrentUser() user: User) {
    return user;
  }
}
```

---

### 2. RolesGuard - 角色授权守卫

**roles.guard.ts**
```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    // 获取路由所需角色
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(
      ROLES_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredRoles) {
      return true; // 未设置角色要求，允许通过
    }

    // 获取当前用户 (由 JwtAuthGuard 注入)
    const { user } = context.switchToHttp().getRequest();

    // 检查用户是否拥有所需角色之一
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
```

**使用方式:**
```typescript
@Controller('users')
export class UsersController {
  @Delete(':id')
  @UseGuards(JwtAuthGuard, RolesGuard)
  @Roles('admin', 'moderator')  // 需要 admin 或 moderator 角色
  deleteUser(@Param('id') id: string) {
    return this.usersService.remove(id);
  }
}
```

---

### 3. PermissionsGuard - 权限授权守卫

**permissions.guard.ts**
```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { PERMISSIONS_KEY } from '../decorators/permissions.decorator';

@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredPermissions = this.reflector.getAllAndOverride<string[]>(
      PERMISSIONS_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredPermissions) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();

    // 检查用户是否拥有所有所需权限
    return requiredPermissions.every((permission) =>
      user.permissions?.includes(permission),
    );
  }
}
```

**使用方式:**
```typescript
@Controller('posts')
export class PostsController {
  @Post()
  @UseGuards(JwtAuthGuard, PermissionsGuard)
  @RequirePermissions('post:create')
  createPost(@Body() createPostDto: CreatePostDto) {
    return this.postsService.create(createPostDto);
  }
}
```

---

### 4. 自定义装饰器

#### @Public() - 标记公开路由

**public.decorator.ts**
```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

**使用:**
```typescript
@Controller('auth')
export class AuthController {
  @Post('login')
  @Public()  // 无需认证
  login(@Body() loginDto: LoginDto) {
    return this.authService.login(loginDto);
  }
}
```

---

#### @CurrentUser() - 获取当前用户

**current-user.decorator.ts**
```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);
```

**使用:**
```typescript
@Get('profile')
getProfile(@CurrentUser() user: User) {
  return user;
}

@Get('username')
getUsername(@CurrentUser('username') username: string) {
  return { username };
}
```

---

#### @Roles() - 指定所需角色

**roles.decorator.ts**
```typescript
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

---

#### @RequirePermissions() - 指定所需权限

**permissions.decorator.ts**
```typescript
import { SetMetadata } from '@nestjs/common';

export const PERMISSIONS_KEY = 'permissions';
export const RequirePermissions = (...permissions: string[]) =>
  SetMetadata(PERMISSIONS_KEY, permissions);
```

---

### 守卫执行顺序

在 `main.ts` 中设置全局守卫顺序:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 全局应用 JWT 认证守卫
  const reflector = app.get(Reflector);
  app.useGlobalGuards(
    new JwtAuthGuard(reflector),      // 1. 先验证身份
    new RolesGuard(reflector),        // 2. 再验证角色
    new PermissionsGuard(reflector),  // 3. 最后验证细粒度权限
  );

  await app.listen(3000);
}
```

---

## 📝 DTOs 与数据验证

### 安装依赖
```bash
pnpm add class-validator class-transformer
```

---

### 1. RegisterDto - 用户注册

**register.dto.ts**
```typescript
import { IsString, IsEmail, MinLength, MaxLength, Matches, IsNotEmpty } from 'class-validator';
import { Match } from '../decorators/match.decorator'; // 自定义验证器

export class RegisterDto {
  @IsString()
  @IsNotEmpty({ message: '用户名不能为空' })
  @MinLength(3, { message: '用户名至少 3 个字符' })
  @MaxLength(20, { message: '用户名最多 20 个字符' })
  @Matches(/^[a-zA-Z0-9_]+$/, {
    message: '用户名只能包含字母、数字和下划线',
  })
  username: string;

  @IsEmail({}, { message: '邮箱格式无效' })
  @IsNotEmpty({ message: '邮箱不能为空' })
  email: string;

  @IsString()
  @IsNotEmpty({ message: '密码不能为空' })
  @MinLength(8, { message: '密码至少 8 个字符' })
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, {
    message: '密码必须包含至少一个大写字母、一个小写字母和一个数字',
  })
  password: string;

  @IsString()
  @IsNotEmpty({ message: '确认密码不能为空' })
  @Match('password', { message: '两次密码输入不一致' })
  confirmPassword: string;
}
```

**自定义 @Match 装饰器:**
```typescript
// match.decorator.ts
import { registerDecorator, ValidationOptions, ValidationArguments } from 'class-validator';

export function Match(property: string, validationOptions?: ValidationOptions) {
  return (object: any, propertyName: string) => {
    registerDecorator({
      name: 'match',
      target: object.constructor,
      propertyName,
      options: validationOptions,
      constraints: [property],
      validator: {
        validate(value: any, args: ValidationArguments) {
          const [relatedPropertyName] = args.constraints;
          const relatedValue = (args.object as any)[relatedPropertyName];
          return value === relatedValue;
        },
      },
    });
  };
}
```

---

### 2. LoginDto - 用户登录

**login.dto.ts**
```typescript
import { IsString, IsNotEmpty } from 'class-validator';

export class LoginDto {
  @IsString()
  @IsNotEmpty({ message: '用户名或邮箱不能为空' })
  usernameOrEmail: string;

  @IsString()
  @IsNotEmpty({ message: '密码不能为空' })
  password: string;
}
```

---

### 3. RefreshTokenDto - 刷新 Token

**refresh-token.dto.ts**
```typescript
import { IsString, IsNotEmpty } from 'class-validator';

export class RefreshTokenDto {
  @IsString()
  @IsNotEmpty({ message: 'Refresh token 不能为空' })
  refreshToken: string;
}
```

---

### 4. CreateRoleDto - 创建角色

**create-role.dto.ts**
```typescript
import { IsString, IsNotEmpty, IsOptional, IsArray, IsInt } from 'class-validator';

export class CreateRoleDto {
  @IsString()
  @IsNotEmpty({ message: '角色名称不能为空' })
  @MinLength(2, { message: '角色名称至少 2 个字符' })
  @MaxLength(50, { message: '角色名称最多 50 个字符' })
  name: string;

  @IsString()
  @IsOptional()
  description?: string;

  @IsArray()
  @IsInt({ each: true })
  @IsOptional()
  permissionIds?: number[];
}
```

---

### 5. AssignRoleDto - 分配角色

**assign-role.dto.ts**
```typescript
import { IsUUID, IsArray, IsInt, ArrayMinSize } from 'class-validator';

export class AssignRoleDto {
  @IsUUID('4', { message: '无效的用户 ID' })
  userId: string;

  @IsArray()
  @IsInt({ each: true })
  @ArrayMinSize(1, { message: '至少需要分配一个角色' })
  roleIds: number[];
}
```

---

### 6. CreatePermissionDto - 创建权限

**create-permission.dto.ts**
```typescript
import { IsString, IsNotEmpty, IsEnum, IsOptional } from 'class-validator';

enum PermissionAction {
  CREATE = 'create',
  READ = 'read',
  UPDATE = 'update',
  DELETE = 'delete',
}

export class CreatePermissionDto {
  @IsString()
  @IsNotEmpty({ message: '权限名称不能为空' })
  name: string;

  @IsString()
  @IsNotEmpty({ message: '资源类型不能为空' })
  resource: string;

  @IsEnum(PermissionAction, {
    message: 'action 必须是 create, read, update 或 delete',
  })
  action: PermissionAction;

  @IsString()
  @IsOptional()
  description?: string;
}
```

---

### 全局 ValidationPipe 配置

**main.ts**
```typescript
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,           // 剥离未定义的属性
      forbidNonWhitelisted: true, // 拒绝未定义的属性
      transform: true,            // 自动类型转换
      transformOptions: {
        enableImplicitConversion: true,
      },
      exceptionFactory: (errors) => {
        // 自定义错误响应格式
        const messages = errors.map(error => ({
          field: error.property,
          errors: Object.values(error.constraints || {}),
        }));
        return new BadRequestException({
          statusCode: 400,
          message: 'Validation failed',
          errors: messages,
        });
      },
    }),
  );

  await app.listen(3000);
}
```

---

## 🌐 API 端点设计

### 完整 API 端点列表

#### 认证相关 (`/api/v1/auth`)

| 方法 | 端点 | 描述 | 权限要求 |
|------|------|------|----------|
| POST | `/auth/register` | 用户注册 | 🌐 公开 |
| POST | `/auth/login` | 用户登录 | 🌐 公开 |
| POST | `/auth/refresh` | 刷新 access token | 🌐 公开 |
| POST | `/auth/logout` | 用户登出 | 🔒 需认证 |
| GET | `/auth/profile` | 获取当前用户信息 | 🔒 需认证 |
| PATCH | `/auth/change-password` | 修改密码 | 🔒 需认证 |

---

#### 用户管理 (`/api/v1/users`)

| 方法 | 端点 | 描述 | 权限要求 |
|------|------|------|----------|
| GET | `/users` | 获取用户列表 (分页) | 👑 admin |
| GET | `/users/:id` | 获取单个用户详情 | 👑 admin 或本人 |
| PATCH | `/users/:id` | 更新用户信息 | 👑 admin 或本人 |
| DELETE | `/users/:id` | 删除用户 | 👑 admin |
| POST | `/users/:id/roles` | 分配角色给用户 | 👑 admin |
| DELETE | `/users/:id/roles/:roleId` | 移除用户角色 | 👑 admin |
| GET | `/users/:id/permissions` | 获取用户所有权限 | 👑 admin 或本人 |

---

#### 角色管理 (`/api/v1/roles`)

| 方法 | 端点 | 描述 | 权限要求 |
|------|------|------|----------|
| GET | `/roles` | 获取所有角色 | 👑 admin |
| POST | `/roles` | 创建新角色 | 👑 admin |
| GET | `/roles/:id` | 获取单个角色详情 | 👑 admin |
| PATCH | `/roles/:id` | 更新角色信息 | 👑 admin |
| DELETE | `/roles/:id` | 删除角色 | 👑 admin |
| POST | `/roles/:id/permissions` | 分配权限给角色 | 👑 admin |
| DELETE | `/roles/:id/permissions/:permissionId` | 移除角色权限 | 👑 admin |

---

#### 权限管理 (`/api/v1/permissions`)

| 方法 | 端点 | 描述 | 权限要求 |
|------|------|------|----------|
| GET | `/permissions` | 获取所有权限 | 👑 admin |
| POST | `/permissions` | 创建新权限 | 👑 admin |
| GET | `/permissions/:id` | 获取权限详情 | 👑 admin |
| DELETE | `/permissions/:id` | 删除权限 | 👑 admin |

---

### 详细端点文档

#### 1. POST /auth/register - 用户注册

**请求:**
```http
POST /api/v1/auth/register
Content-Type: application/json

{
  "username": "johndoe",
  "email": "john@example.com",
  "password": "Password123",
  "confirmPassword": "Password123"
}
```

**成功响应 (201 Created):**
```json
{
  "success": true,
  "message": "注册成功",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "username": "johndoe",
    "email": "john@example.com",
    "createdAt": "2026-01-14T10:00:00.000Z"
  }
}
```

**错误响应 (409 Conflict):**
```json
{
  "success": false,
  "statusCode": 409,
  "error": {
    "code": "USER_EXISTS",
    "message": "用户名或邮箱已存在"
  }
}
```

---

#### 2. POST /auth/login - 用户登录

**请求:**
```http
POST /api/v1/auth/login
Content-Type: application/json

{
  "usernameOrEmail": "johndoe",
  "password": "Password123"
}
```

**成功响应 (200 OK):**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "username": "johndoe",
      "email": "john@example.com",
      "roles": ["user"],
      "permissions": ["user:read"]
    }
  }
}
```

**错误响应 (401 Unauthorized):**
```json
{
  "success": false,
  "statusCode": 401,
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "用户名或密码错误"
  }
}
```

---

#### 3. GET /users - 获取用户列表

**请求:**
```http
GET /api/v1/users?page=1&limit=10&search=john
Authorization: Bearer {accessToken}
```

**查询参数:**
- `page`: 页码 (默认: 1)
- `limit`: 每页数量 (默认: 10)
- `search`: 搜索关键词 (可选)
- `sort`: 排序字段 (默认: createdAt)
- `order`: 排序方向 ASC/DESC (默认: DESC)

**成功响应 (200 OK):**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "username": "johndoe",
        "email": "john@example.com",
        "isActive": true,
        "roles": ["user"],
        "createdAt": "2026-01-14T10:00:00.000Z"
      }
    ],
    "meta": {
      "currentPage": 1,
      "itemsPerPage": 10,
      "totalItems": 42,
      "totalPages": 5
    }
  }
}
```

---

#### 4. POST /users/:id/roles - 分配角色

**请求:**
```http
POST /api/v1/users/550e8400-e29b-41d4-a716-446655440000/roles
Authorization: Bearer {accessToken}
Content-Type: application/json

{
  "roleIds": [1, 2]
}
```

**成功响应 (200 OK):**
```json
{
  "success": true,
  "message": "角色分配成功",
  "data": {
    "userId": "550e8400-e29b-41d4-a716-446655440000",
    "roles": ["user", "moderator"]
  }
}
```

---

## ❌ 错误处理

### 统一错误响应格式

```typescript
interface ErrorResponse {
  success: false;
  statusCode: number;
  error: {
    code: string;           // 错误代码
    message: string;        // 错误消息
    details?: any[];        // 详细信息 (如验证错误)
    timestamp?: string;     // 时间戳
    path?: string;          // 请求路径
  };
}
```

---

### 自定义异常类

#### 1. UserAlreadyExistsException

**user-already-exists.exception.ts**
```typescript
import { ConflictException } from '@nestjs/common';

export class UserAlreadyExistsException extends ConflictException {
  constructor(field: 'username' | 'email') {
    super({
      code: 'USER_EXISTS',
      message: `${field === 'username' ? '用户名' : '邮箱'}已存在`,
      field,
    });
  }
}
```

---

#### 2. InvalidCredentialsException

**invalid-credentials.exception.ts**
```typescript
import { UnauthorizedException } from '@nestjs/common';

export class InvalidCredentialsException extends UnauthorizedException {
  constructor() {
    super({
      code: 'INVALID_CREDENTIALS',
      message: '用户名或密码错误',
    });
  }
}
```

---

#### 3. InsufficientPermissionsException

**insufficient-permissions.exception.ts**
```typescript
import { ForbiddenException } from '@nestjs/common';

export class InsufficientPermissionsException extends ForbiddenException {
  constructor(requiredPermission?: string) {
    super({
      code: 'INSUFFICIENT_PERMISSIONS',
      message: '权限不足',
      requiredPermission,
    });
  }
}
```

---

### 全局异常过滤器

**http-exception.filter.ts**
```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let errorResponse: any = {
      code: 'INTERNAL_ERROR',
      message: 'Internal server error',
    };

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const exceptionResponse = exception.getResponse();

      if (typeof exceptionResponse === 'object') {
        errorResponse = exceptionResponse;
      } else {
        errorResponse = {
          code: 'HTTP_EXCEPTION',
          message: exceptionResponse,
        };
      }
    } else if (exception instanceof Error) {
      errorResponse.message = exception.message;
      errorResponse.stack = process.env.NODE_ENV === 'development'
        ? exception.stack
        : undefined;
    }

    // 记录错误日志
    this.logger.error(
      `${request.method} ${request.url} - ${status} - ${errorResponse.message}`,
      exception instanceof Error ? exception.stack : '',
    );

    // 返回统一格式
    response.status(status).json({
      success: false,
      statusCode: status,
      error: {
        ...errorResponse,
        timestamp: new Date().toISOString(),
        path: request.url,
      },
    });
  }
}
```

**在 main.ts 中注册:**
```typescript
app.useGlobalFilters(new HttpExceptionFilter());
```

---

### HTTP 状态码使用规范

| 状态码 | 含义 | 使用场景 |
|--------|------|---------|
| 200 OK | 成功 | GET, PATCH, DELETE 成功 |
| 201 Created | 创建成功 | POST 创建资源成功 |
| 204 No Content | 成功但无内容 | DELETE 成功 (可选) |
| 400 Bad Request | 请求参数错误 | DTO 验证失败 |
| 401 Unauthorized | 未认证 | Token 无效或过期 |
| 403 Forbidden | 无权限 | 角色或权限不足 |
| 404 Not Found | 资源不存在 | 用户、角色不存在 |
| 409 Conflict | 资源冲突 | 用户名/邮箱已存在 |
| 422 Unprocessable Entity | 语义错误 | 业务逻辑验证失败 |
| 429 Too Many Requests | 请求过多 | 触发限流 |
| 500 Internal Server Error | 服务器错误 | 未处理的异常 |

---

## 🔧 中间件与拦截器

### 1. LoggerMiddleware - 请求日志

**logger.middleware.ts**
```typescript
import { Injectable, NestMiddleware, Logger } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  private logger = new Logger('HTTP');

  use(req: Request, res: Response, next: NextFunction) {
    const { method, originalUrl, ip } = req;
    const userAgent = req.get('user-agent') || '';
    const startTime = Date.now();

    res.on('finish', () => {
      const { statusCode } = res;
      const responseTime = Date.now() - startTime;

      this.logger.log(
        `${method} ${originalUrl} ${statusCode} ${responseTime}ms - ${ip} ${userAgent}`,
      );
    });

    next();
  }
}
```

**在 AppModule 中注册:**
```typescript
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes('*');
  }
}
```

---

### 2. TransformInterceptor - 响应转换

**transform.interceptor.ts**
```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  success: boolean;
  data: T;
  timestamp: string;
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, Response<T>> {
  intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Observable<Response<T>> {
    return next.handle().pipe(
      map((data) => ({
        success: true,
        data,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
```

**在 main.ts 中注册:**
```typescript
app.useGlobalInterceptors(new TransformInterceptor());
```

---

### 3. TimeoutInterceptor - 请求超时

**timeout.interceptor.ts**
```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  RequestTimeoutException,
} from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(30000), // 30 秒超时
      catchError((err) => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException('请求超时'));
        }
        return throwError(() => err);
      }),
    );
  }
}
```

---

### 4. CORS 配置

**main.ts**
```typescript
app.enableCors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3001',
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  credentials: true,
  allowedHeaders: ['Content-Type', 'Authorization'],
});
```

---

### 5. 安全头配置 (Helmet)

```bash
pnpm add helmet
```

**main.ts**
```typescript
import helmet from 'helmet';

app.use(helmet());
```

---

### 6. 压缩中间件

```bash
pnpm add compression
pnpm add -D @types/compression
```

**main.ts**
```typescript
import compression from 'compression';

app.use(compression());
```

---

## 🌱 数据库迁移与种子数据

### 1. TypeORM CLI 配置

**ormconfig.ts** (或在 package.json 中配置)
```typescript
import { DataSource } from 'typeorm';
import { config } from 'dotenv';

config();

export default new DataSource({
  type: 'postgres',
  host: process.env.DATABASE_HOST,
  port: parseInt(process.env.DATABASE_PORT || '5432'),
  username: process.env.DATABASE_USERNAME,
  password: process.env.DATABASE_PASSWORD,
  database: process.env.DATABASE_NAME,
  entities: ['src/**/*.entity.ts'],
  migrations: ['src/database/migrations/*.ts'],
  synchronize: false, // 生产环境必须为 false
});
```

**package.json scripts:**
```json
{
  "scripts": {
    "typeorm": "typeorm-ts-node-commonjs",
    "migration:generate": "npm run typeorm -- migration:generate",
    "migration:create": "npm run typeorm -- migration:create",
    "migration:run": "npm run typeorm -- migration:run",
    "migration:revert": "npm run typeorm -- migration:revert",
    "seed:run": "ts-node src/database/seeds/run-seeds.ts"
  }
}
```

---

### 2. 初始迁移

**生成迁移:**
```bash
pnpm run migration:generate src/database/migrations/InitialSchema
```

**示例迁移文件:**
```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class InitialSchema1705200000000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    // 创建 users 表
    await queryRunner.query(`
      CREATE TABLE users (
        id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
        username VARCHAR(50) UNIQUE NOT NULL,
        email VARCHAR(100) UNIQUE NOT NULL,
        password VARCHAR(255) NOT NULL,
        is_active BOOLEAN DEFAULT true,
        created_at TIMESTAMP DEFAULT NOW(),
        updated_at TIMESTAMP DEFAULT NOW()
      )
    `);

    // 创建 roles 表
    await queryRunner.query(`
      CREATE TABLE roles (
        id SERIAL PRIMARY KEY,
        name VARCHAR(50) UNIQUE NOT NULL,
        description TEXT,
        created_at TIMESTAMP DEFAULT NOW()
      )
    `);

    // 创建 permissions 表
    await queryRunner.query(`
      CREATE TABLE permissions (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) UNIQUE NOT NULL,
        resource VARCHAR(50) NOT NULL,
        action VARCHAR(20) NOT NULL,
        description TEXT,
        created_at TIMESTAMP DEFAULT NOW()
      )
    `);

    // 创建 user_roles 关联表
    await queryRunner.query(`
      CREATE TABLE user_roles (
        user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
        role_id INT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
        PRIMARY KEY (user_id, role_id)
      )
    `);

    // 创建 role_permissions 关联表
    await queryRunner.query(`
      CREATE TABLE role_permissions (
        role_id INT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
        permission_id INT NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
        PRIMARY KEY (role_id, permission_id)
      )
    `);

    // 创建索引
    await queryRunner.query(`
      CREATE INDEX idx_users_username ON users(username);
      CREATE INDEX idx_users_email ON users(email);
      CREATE INDEX idx_roles_name ON roles(name);
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP TABLE IF EXISTS role_permissions`);
    await queryRunner.query(`DROP TABLE IF EXISTS user_roles`);
    await queryRunner.query(`DROP TABLE IF EXISTS permissions`);
    await queryRunner.query(`DROP TABLE IF EXISTS roles`);
    await queryRunner.query(`DROP TABLE IF EXISTS users`);
  }
}
```

**运行迁移:**
```bash
pnpm run migration:run
```

---

### 3. 种子数据

#### permissions.seed.ts

```typescript
import { DataSource } from 'typeorm';
import { Permission } from '../entities/permission.entity';

export async function seedPermissions(dataSource: DataSource) {
  const permissionRepository = dataSource.getRepository(Permission);

  const permissions = [
    // 用户权限
    { name: 'user:create', resource: 'user', action: 'create', description: '创建用户' },
    { name: 'user:read', resource: 'user', action: 'read', description: '读取用户信息' },
    { name: 'user:update', resource: 'user', action: 'update', description: '更新用户信息' },
    { name: 'user:delete', resource: 'user', action: 'delete', description: '删除用户' },

    // 角色权限
    { name: 'role:create', resource: 'role', action: 'create', description: '创建角色' },
    { name: 'role:read', resource: 'role', action: 'read', description: '读取角色' },
    { name: 'role:update', resource: 'role', action: 'update', description: '更新角色' },
    { name: 'role:delete', resource: 'role', action: 'delete', description: '删除角色' },

    // 权限权限
    { name: 'permission:create', resource: 'permission', action: 'create', description: '创建权限' },
    { name: 'permission:read', resource: 'permission', action: 'read', description: '读取权限' },
    { name: 'permission:delete', resource: 'permission', action: 'delete', description: '删除权限' },
  ];

  for (const permission of permissions) {
    const exists = await permissionRepository.findOne({
      where: { name: permission.name },
    });

    if (!exists) {
      await permissionRepository.save(permission);
    }
  }

  console.log('✅ Permissions seeded successfully');
}
```

---

#### roles.seed.ts

```typescript
import { DataSource } from 'typeorm';
import { Role } from '../entities/role.entity';
import { Permission } from '../entities/permission.entity';

export async function seedRoles(dataSource: DataSource) {
  const roleRepository = dataSource.getRepository(Role);
  const permissionRepository = dataSource.getRepository(Permission);

  const allPermissions = await permissionRepository.find();
  const userPermissions = await permissionRepository.find({
    where: { name: 'user:read' },
  });

  const roles = [
    {
      name: 'admin',
      description: '系统管理员，拥有所有权限',
      permissions: allPermissions,
    },
    {
      name: 'moderator',
      description: '内容审核员，部分管理权限',
      permissions: allPermissions.filter(p => p.action === 'read' || p.action === 'update'),
    },
    {
      name: 'user',
      description: '普通用户，基础权限',
      permissions: userPermissions,
    },
  ];

  for (const roleData of roles) {
    const exists = await roleRepository.findOne({
      where: { name: roleData.name },
    });

    if (!exists) {
      const role = roleRepository.create(roleData);
      await roleRepository.save(role);
    }
  }

  console.log('✅ Roles seeded successfully');
}
```

---

#### admin.seed.ts

```typescript
import { DataSource } from 'typeorm';
import * as bcrypt from 'bcrypt';
import { User } from '../entities/user.entity';
import { Role } from '../entities/role.entity';

export async function seedAdmin(dataSource: DataSource) {
  const userRepository = dataSource.getRepository(User);
  const roleRepository = dataSource.getRepository(Role);

  const adminRole = await roleRepository.findOne({
    where: { name: 'admin' },
  });

  if (!adminRole) {
    throw new Error('Admin role not found. Please run roles seed first.');
  }

  const adminExists = await userRepository.findOne({
    where: { username: 'admin' },
  });

  if (!adminExists) {
    const hashedPassword = await bcrypt.hash('Admin@123', 10);

    const admin = userRepository.create({
      username: 'admin',
      email: 'admin@example.com',
      password: hashedPassword,
      roles: [adminRole],
      isActive: true,
    });

    await userRepository.save(admin);
    console.log('✅ Admin user created successfully');
    console.log('📧 Email: admin@example.com');
    console.log('🔑 Password: Admin@123');
    console.log('⚠️  请首次登录后立即修改密码！');
  } else {
    console.log('ℹ️  Admin user already exists');
  }
}
```

---

#### run-seeds.ts

```typescript
import { DataSource } from 'typeorm';
import { config } from 'dotenv';
import dataSource from '../../ormconfig';
import { seedPermissions } from './permissions.seed';
import { seedRoles } from './roles.seed';
import { seedAdmin } from './admin.seed';

config();

async function runSeeds() {
  try {
    await dataSource.initialize();
    console.log('📦 Database connected');

    await seedPermissions(dataSource);
    await seedRoles(dataSource);
    await seedAdmin(dataSource);

    console.log('🎉 All seeds completed successfully');
    await dataSource.destroy();
  } catch (error) {
    console.error('❌ Seed failed:', error);
    process.exit(1);
  }
}

runSeeds();
```

**运行种子数据:**
```bash
pnpm run seed:run
```

---

## 🧪 测试策略

### 1. 单元测试示例

#### AuthService 测试

**auth.service.spec.ts**
```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { JwtService } from '@nestjs/jwt';
import { AuthService } from './auth.service';
import { UsersService } from '../users/users.service';
import * as bcrypt from 'bcrypt';

describe('AuthService', () => {
  let service: AuthService;
  let usersService: UsersService;
  let jwtService: JwtService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        AuthService,
        {
          provide: UsersService,
          useValue: {
            findByUsername: jest.fn(),
            create: jest.fn(),
          },
        },
        {
          provide: JwtService,
          useValue: {
            sign: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<AuthService>(AuthService);
    usersService = module.get<UsersService>(UsersService);
    jwtService = module.get<JwtService>(JwtService);
  });

  describe('register', () => {
    it('should successfully register a new user', async () => {
      const registerDto = {
        username: 'testuser',
        email: 'test@example.com',
        password: 'Password123',
        confirmPassword: 'Password123',
      };

      jest.spyOn(usersService, 'findByUsername').mockResolvedValue(null);
      jest.spyOn(usersService, 'create').mockResolvedValue({
        id: '123',
        ...registerDto,
        password: 'hashed',
      } as any);

      const result = await service.register(registerDto);

      expect(result).toBeDefined();
      expect(result.username).toBe(registerDto.username);
    });

    it('should throw error if username already exists', async () => {
      const registerDto = {
        username: 'existinguser',
        email: 'test@example.com',
        password: 'Password123',
        confirmPassword: 'Password123',
      };

      jest.spyOn(usersService, 'findByUsername').mockResolvedValue({} as any);

      await expect(service.register(registerDto)).rejects.toThrow();
    });
  });

  describe('login', () => {
    it('should return tokens for valid credentials', async () => {
      const loginDto = {
        usernameOrEmail: 'testuser',
        password: 'Password123',
      };

      const user = {
        id: '123',
        username: 'testuser',
        password: await bcrypt.hash('Password123', 10),
        roles: ['user'],
      };

      jest.spyOn(usersService, 'findByUsername').mockResolvedValue(user as any);
      jest.spyOn(jwtService, 'sign').mockReturnValue('token');

      const result = await service.login(loginDto);

      expect(result).toHaveProperty('accessToken');
      expect(result).toHaveProperty('refreshToken');
    });
  });
});
```

---

### 2. E2E 测试示例

**auth.e2e-spec.ts**
```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import request from 'supertest';
import { AppModule } from './../src/app.module';

describe('Authentication (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe());
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('/auth/register (POST)', () => {
    it('should register a new user', () => {
      return request(app.getHttpServer())
        .post('/auth/register')
        .send({
          username: 'testuser',
          email: 'test@example.com',
          password: 'Password123',
          confirmPassword: 'Password123',
        })
        .expect(201)
        .expect((res) => {
          expect(res.body.success).toBe(true);
          expect(res.body.data).toHaveProperty('id');
        });
    });

    it('should fail with invalid email', () => {
      return request(app.getHttpServer())
        .post('/auth/register')
        .send({
          username: 'testuser',
          email: 'invalid-email',
          password: 'Password123',
          confirmPassword: 'Password123',
        })
        .expect(400);
    });
  });

  describe('/auth/login (POST)', () => {
    it('should login with valid credentials', () => {
      return request(app.getHttpServer())
        .post('/auth/login')
        .send({
          usernameOrEmail: 'admin',
          password: 'Admin@123',
        })
        .expect(200)
        .expect((res) => {
          expect(res.body.data).toHaveProperty('accessToken');
          expect(res.body.data).toHaveProperty('refreshToken');
        });
    });

    it('should fail with invalid credentials', () => {
      return request(app.getHttpServer())
        .post('/auth/login')
        .send({
          usernameOrEmail: 'admin',
          password: 'wrongpassword',
        })
        .expect(401);
    });
  });

  describe('/auth/profile (GET)', () => {
    let accessToken: string;

    beforeAll(async () => {
      const response = await request(app.getHttpServer())
        .post('/auth/login')
        .send({
          usernameOrEmail: 'admin',
          password: 'Admin@123',
        });

      accessToken = response.body.data.accessToken;
    });

    it('should return user profile with valid token', () => {
      return request(app.getHttpServer())
        .get('/auth/profile')
        .set('Authorization', `Bearer ${accessToken}`)
        .expect(200)
        .expect((res) => {
          expect(res.body.data).toHaveProperty('username');
        });
    });

    it('should fail without token', () => {
      return request(app.getHttpServer())
        .get('/auth/profile')
        .expect(401);
    });
  });
});
```

---

### 测试覆盖率目标

- ✅ 单元测试覆盖率 > 80%
- ✅ 关键业务逻辑覆盖率 100%
- ✅ E2E 测试覆盖所有端点

**运行测试:**
```bash
# 单元测试
pnpm run test

# E2E 测试
pnpm run test:e2e

# 测试覆盖率
pnpm run test:cov
```

---

## ⚙️ 配置管理

### ConfigModule 设置

**app.module.ts**
```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import appConfig from './config/app.config';
import databaseConfig from './config/database.config';
import jwtConfig from './config/jwt.config';
import validationSchema from './config/validation.schema';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [appConfig, databaseConfig, jwtConfig],
      validationSchema,
      envFilePath: `.env.${process.env.NODE_ENV || 'development'}`,
    }),
    // ... 其他模块
  ],
})
export class AppModule {}
```

---

### 配置文件

#### app.config.ts

```typescript
import { registerAs } from '@nestjs/config';

export default registerAs('app', () => ({
  env: process.env.NODE_ENV || 'development',
  port: parseInt(process.env.PORT || '3000', 10),
  apiPrefix: process.env.API_PREFIX || 'api/v1',
}));
```

---

#### validation.schema.ts

使用 Joi 验证环境变量:

```bash
pnpm add joi
```

```typescript
import * as Joi from 'joi';

export default Joi.object({
  NODE_ENV: Joi.string()
    .valid('development', 'production', 'test')
    .default('development'),
  PORT: Joi.number().default(3000),

  DATABASE_HOST: Joi.string().required(),
  DATABASE_PORT: Joi.number().default(5432),
  DATABASE_USERNAME: Joi.string().required(),
  DATABASE_PASSWORD: Joi.string().required(),
  DATABASE_NAME: Joi.string().required(),

  JWT_ACCESS_SECRET: Joi.string().min(32).required(),
  JWT_REFRESH_SECRET: Joi.string().min(32).required(),
  JWT_ACCESS_EXPIRATION: Joi.string().default('15m'),
  JWT_REFRESH_EXPIRATION: Joi.string().default('7d'),
});
```

---

## 🔒 安全最佳实践

### 1. 密码安全

- ✅ 使用 bcrypt 加密 (salt rounds: 10)
- ✅ 密码强度要求: 最少 8 字符, 包含大小写字母和数字
- ✅ 永远不返回密码字段
- ✅ 实现密码重置机制 (email token)

---

### 2. JWT 安全

- ✅ 使用强随机 Secret (最少 32 字符)
- ✅ 短过期时间 (15分钟 access token)
- ✅ 实现 refresh token rotation
- ✅ Token 黑名单机制 (Redis)
- ✅ 在 HttpOnly Cookie 中存储 token

---

### 3. 限流保护

```bash
pnpm add @nestjs/throttler
```

**app.module.ts**
```typescript
import { ThrottlerModule } from '@nestjs/throttler';

@Module({
  imports: [
    ThrottlerModule.forRoot({
      ttl: 60,    // 时间窗口 60 秒
      limit: 10,  // 最多 10 个请求
    }),
  ],
})
export class AppModule {}
```

**在控制器中使用:**
```typescript
import { Throttle } from '@nestjs/throttler';

@Controller('auth')
export class AuthController {
  @Post('login')
  @Throttle(5, 60)  // 登录端点: 60 秒内最多 5 次
  login(@Body() loginDto: LoginDto) {
    // ...
  }
}
```

---

### 4. 账户锁定机制

实现逻辑:
1. 用户登录失败时增加失败次数计数器 (Redis)
2. 失败次数超过阈值 (如 5 次) 后锁定账户
3. 锁定时间 (如 15 分钟) 后自动解锁

---

### 5. 数据验证

- ✅ 使用 class-validator 验证所有输入
- ✅ 启用 whitelist 和 forbidNonWhitelisted
- ✅ SQL 注入防护 (TypeORM 自动提供)
- ✅ XSS 防护 (sanitize 用户输入)

---

### 6. 环境变量保护

- ❌ 永远不要提交 .env 文件到 Git
- ✅ 使用 .env.example 作为模板
- ✅ 生产环境使用密钥管理服务 (AWS Secrets Manager, HashiCorp Vault)

---

### 7. HTTPS 强制

生产环境必须使用 HTTPS:

```typescript
// main.ts
if (process.env.NODE_ENV === 'production') {
  app.use((req, res, next) => {
    if (req.header('x-forwarded-proto') !== 'https') {
      res.redirect(`https://${req.header('host')}${req.url}`);
    } else {
      next();
    }
  });
}
```

---

### 8. 安全头配置

使用 Helmet 增强安全:

```typescript
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
  },
}));
```

---

## 📅 实施计划

### 阶段 1: 基础设施搭建 (1-2 天)

**任务清单:**
- [ ] 安装所有必需依赖包
- [ ] 配置环境变量和 ConfigModule
- [ ] 设置 TypeORM 和 PostgreSQL 连接
- [ ] 创建项目目录结构
- [ ] 配置 ESLint, Prettier

**验收标准:**
- ✅ 数据库连接成功
- ✅ 环境变量验证通过
- ✅ 项目结构清晰

---

### 阶段 2: 数据模型和数据库 (1-2 天)

**任务清单:**
- [ ] 创建 User, Role, Permission 实体
- [ ] 定义实体关系 (多对多)
- [ ] 生成数据库迁移
- [ ] 运行迁移创建表结构
- [ ] 创建种子数据脚本
- [ ] 运行种子数据

**验收标准:**
- ✅ 数据库表结构正确
- ✅ 关联关系正确
- ✅ 种子数据插入成功
- ✅ 能查询到默认 admin 用户

---

### 阶段 3: 认证核心功能 (2-3 天)

**任务清单:**
- [ ] 实现 UsersModule 和 UsersService
- [ ] 实现 AuthModule 和 AuthService
- [ ] 实现密码加密和验证
- [ ] 实现 JWT 策略和 token 生成
- [ ] 实现 POST /auth/register 端点
- [ ] 实现 POST /auth/login 端点
- [ ] 实现 POST /auth/refresh 端点
- [ ] 实现 POST /auth/logout 端点
- [ ] 实现 GET /auth/profile 端点

**验收标准:**
- ✅ 用户注册成功
- ✅ 用户登录返回 tokens
- ✅ Token 验证通过
- ✅ Refresh token 工作正常

---

### 阶段 4: 授权和 RBAC (2-3 天)

**任务清单:**
- [ ] 实现 RolesModule 和 RolesService
- [ ] 实现 PermissionsModule 和 PermissionsService
- [ ] 创建 JwtAuthGuard
- [ ] 创建 RolesGuard
- [ ] 创建 PermissionsGuard
- [ ] 创建自定义装饰器 (@Public, @CurrentUser, @Roles, @RequirePermissions)
- [ ] 实现角色管理 API
- [ ] 实现权限管理 API
- [ ] 实现用户角色分配 API

**验收标准:**
- ✅ 守卫正确拦截未授权请求
- ✅ 角色验证工作正常
- ✅ 权限验证工作正常
- ✅ Admin 可以管理角色和权限

---

### 阶段 5: 增强和优化 (1-2 天)

**任务清单:**
- [ ] 添加全局异常过滤器
- [ ] 实现响应转换拦截器
- [ ] 添加请求日志中间件
- [ ] 配置 CORS
- [ ] 添加 Helmet 安全头
- [ ] 添加 Compression 压缩
- [ ] 实现限流保护
- [ ] 添加 Swagger API 文档
- [ ] 实现分页功能
- [ ] 实现搜索和排序

**验收标准:**
- ✅ 错误处理统一
- ✅ 响应格式统一
- ✅ 日志记录完整
- ✅ API 文档可访问
- ✅ 限流生效

---

### 阶段 6: 测试和文档 (2-3 天)

**任务清单:**
- [ ] 编写 AuthService 单元测试
- [ ] 编写 UsersService 单元测试
- [ ] 编写 Guards 单元测试
- [ ] 编写认证流程 E2E 测试
- [ ] 编写授权流程 E2E 测试
- [ ] 编写角色管理 E2E 测试
- [ ] 完善 Swagger API 文档
- [ ] 编写部署文档
- [ ] 编写开发者指南

**验收标准:**
- ✅ 测试覆盖率 > 80%
- ✅ 所有测试通过
- ✅ API 文档完整
- ✅ 部署文档清晰

---

### 总工时估算

| 阶段 | 预估工时 |
|------|----------|
| 阶段 1: 基础设施 | 1-2 天 |
| 阶段 2: 数据模型 | 1-2 天 |
| 阶段 3: 认证功能 | 2-3 天 |
| 阶段 4: 授权 RBAC | 2-3 天 |
| 阶段 5: 增强优化 | 1-2 天 |
| 阶段 6: 测试文档 | 2-3 天 |
| **总计** | **9-15 天** |

---

## 📦 依赖包清单

### 核心依赖

```bash
# 已安装 (from package.json)
@nestjs/common@^11.0.1
@nestjs/core@^11.0.1
@nestjs/platform-express@^11.0.1
reflect-metadata@^0.2.2
rxjs@^7.8.1

# 需要安装 - 数据库
pnpm add @nestjs/typeorm typeorm pg

# 需要安装 - 认证
pnpm add @nestjs/jwt @nestjs/passport passport passport-jwt passport-local bcrypt
pnpm add -D @types/passport-jwt @types/passport-local @types/bcrypt

# 需要安装 - 配置
pnpm add @nestjs/config joi

# 需要安装 - 验证
pnpm add class-validator class-transformer

# 需要安装 - 安全
pnpm add helmet compression @nestjs/throttler

# 需要安装 - 文档
pnpm add @nestjs/swagger swagger-ui-express
```

---

### 完整安装命令

```bash
# 一键安装所有依赖
pnpm add @nestjs/typeorm typeorm pg \
         @nestjs/jwt @nestjs/passport passport passport-jwt passport-local bcrypt \
         @nestjs/config joi \
         class-validator class-transformer \
         helmet compression @nestjs/throttler \
         @nestjs/swagger swagger-ui-express

# 开发依赖
pnpm add -D @types/passport-jwt @types/passport-local @types/bcrypt \
            @types/compression
```

---

## 🎯 总结

本文档提供了一个完整的 JWT 认证和 RBAC 系统实现计划，包含:

✅ **完整的技术栈选型**
✅ **详细的数据库设计**
✅ **清晰的模块架构**
✅ **完善的认证流程**
✅ **强大的授权机制**
✅ **最佳安全实践**
✅ **全面的测试策略**
✅ **详细的实施计划**

---

## 📚 参考资源

- [NestJS 官方文档](https://docs.nestjs.com/)
- [TypeORM 文档](https://typeorm.io/)
- [Passport.js 文档](http://www.passportjs.org/)
- [JWT 规范](https://jwt.io/)
- [OWASP 安全指南](https://owasp.org/)

---

**文档版本:** 1.0.0
**最后更新:** 2026-01-14
**维护者:** Claude Code

---

## 下一步行动

1. ✅ 审阅本文档，确认技术选型和架构设计
2. ⏭️ 开始阶段 1: 安装依赖和配置基础设施
3. ⏭️ 按照实施计划逐步实现功能
4. ⏭️ 每个阶段完成后进行验收测试

**准备好开始实施了吗？** 🚀