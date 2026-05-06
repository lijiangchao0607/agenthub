# AgentHub 开发规范文档

> 最后更新: 2026-05-06  
> 适用项目: AgentHub AI Agent 管理平台  
> 部署地址: https://agent.suncentgroup.com

---

## 目录

1. [项目概述](#1-项目概述)
2. [接口规范](#2-接口规范)
3. [认证与权限](#3-认证与权限)
4. [部署流程](#4-部署流程)
5. [测试验证清单](#5-测试验证清单)
6. [常见问题及解决方案](#6-常见问题及解决方案)

---

## 1. 项目概述

### 1.1 项目架构

AgentHub 采用 **前后端分离** 架构，通过 Docker 容器化部署：

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│   Nginx     │────▶│   Frontend   │     │    Backend      │
│  (宿主机)    │────▶│  (静态文件)   │────▶│  (FastAPI容器)   │
└─────────────┘     └──────────────┘     └────────┬────────┘
                                                   │
                                          ┌────────┴────────┐
                                          │                 │
                                   ┌──────▼──────┐  ┌──────▼──────┐
                                   │ PostgreSQL  │  │    Redis    │
                                   │  (Docker)   │  │  (Docker)   │
                                   └─────────────┘  └─────────────┘
```

| 组件 | 说明 | 部署位置 |
|------|------|----------|
| 前端 | Vue 3 + TypeScript SPA | 宿主机 `/opt/app/frontend/`，构建输出到 `dist/` |
| 后端 | FastAPI REST API | Docker 容器 `agenthub-backend`，代码 `/app/app/` |
| 数据库 | PostgreSQL | Docker 容器 `agenthub-postgres` |
| 缓存 | Redis | Docker 容器 `agenthub-redis` |
| 反向代理 | Nginx | 宿主机，域名 `agent.suncentgroup.com` |

### 1.2 技术栈

#### 后端技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| Python | 3.12 | 运行时 |
| FastAPI | - | Web 框架 |
| SQLAlchemy | async | ORM（异步模式） |
| asyncpg | - | PostgreSQL 异步驱动 |
| python-jose | - | JWT Token 生成与验证 |
| passlib | - | 密码哈希 |
| Pydantic | v2 | 数据校验与 Schema 定义 |
| Uvicorn | - | ASGI 服务器 |

#### 前端技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| Vue | 3 | UI 框架（Composition API） |
| TypeScript | - | 类型系统 |
| Ant Design Vue | - | UI 组件库 |
| Vite | - | 构建工具 |
| Pinia | - | 状态管理 |
| Axios | - | HTTP 客户端（封装于 `src/api/request.ts`） |

---

## 2. 接口规范

> ⚠️ **这是最重要的章节！** 前后端联调问题 90% 源于不遵守本规范。请每次开发时严格对照检查。

### 2.1 后端统一响应格式

所有 API 接口必须使用以下统一响应格式：

**成功响应：**
```json
{
    "code": 0,
    "data": { /* 业务数据 */ },
    "message": "ok",
    "timestamp": 1715012345.123
}
```

**失败响应：**
```json
{
    "code": 40001,
    "data": null,
    "message": "用户名或密码错误",
    "timestamp": 1715012345.123
}
```

**标准错误码定义：**

| 错误码 | 含义 |
|--------|------|
| 0 | 成功 |
| 40000 | 请求参数错误 |
| 40100 | 未认证（Token 缺失或无效） |
| 40300 | 无权限 |
| 40400 | 资源不存在 |
| 50000 | 服务器内部错误 |

> **后端开发者注意：** 不要返回裸数据，必须包裹在上述格式中。建议在 FastAPI 中实现统一的响应包装工具函数或中间件。

### 2.2 前端响应拦截器行为（关键！）

前端 Axios 响应拦截器（位于 `src/api/request.ts`）已修改为：

```typescript
// 响应拦截器
response => {
    const res = response.data;
    // res 的结构: { code: 0, data: ..., message: "ok", timestamp: ... }
    if (res.code !== 0) {
        // 处理错误
        return Promise.reject(new Error(res.message));
    }
    return res.data;  // ← 注意：直接返回 data 字段
}
```

**这意味着：**

```typescript
// ❌ 错误写法 — 不要再从 data 中取值
const res = await api.login(params);
const token = res.data.access_token;  // res.data 是 undefined！

// ✅ 正确写法 — 拦截器已经解包，直接使用返回值
const data = await api.login(params);
const token = data.access_token;  // 直接取值
```

**规则：前端 API 调用者拿到的是 `response.data` 中的 `data` 字段值，而非完整响应对象。**

### 2.3 字段命名规则

后端和前端 **必须** 使用统一的字段命名，特别是用户信息相关字段。

**规范要求：**

1. **先约定，后开发：** 新增接口前，前后端开发者必须先确认字段名称，写入接口文档
2. **使用 snake_case：** 后端数据库和 API 统一使用 `snake_case` 命名
3. **前端 TypeScript interface 必须与后端返回的字段名一一对应**

**用户信息字段约定（已有规范）：**

| 后端字段名 | 前端字段名 | 说明 |
|-----------|-----------|------|
| `display_name` | `display_name` | 显示名称（已统一） |
| `avatar_url` | `avatar_url` | 头像地址（已统一） |
| `nickname` | `nickname` | 昵称 |
| `roles` | `roles` | 角色列表 |

> ⚠️ **历史教训：** 早期版本曾出现后端返回 `display_name`/`avatar_url` 但前端类型定义为 `nickname`/`avatar` 的不一致问题，导致用户信息显示异常。务必在开发前对齐字段名。

### 2.4 API 路径规范

| 规则 | 说明 |
|------|------|
| 路由前缀 | 所有后端 API 统一使用 `/api/v1/` 前缀 |
| 命名风格 | 使用小写字母 + 下划线，使用复数名词（如 `/api/v1/users`、`/api/v1/depts`） |
| 路径一致性 | **前端调用路径必须和后端注册路径完全一致**，包括大小写和分隔符 |
| 嵌套资源 | 使用路径参数，如 `/api/v1/depts/{dept_id}/agents` |

**已有路由映射表：**

| 功能模块 | 后端路由 | 前端 API 文件 |
|---------|---------|-------------|
| 认证 | `/api/v1/auth/*` | `src/api/auth.ts` |
| 用户 | `/api/v1/users/*` | `src/api/users.ts` |
| 部门 | `/api/v1/depts/*` | `src/api/depts.ts` |
| Agent | `/api/v1/agents/*` | `src/api/agents.ts` |
| 监控 | `/api/v1/monitor/*` | `src/api/monitor.ts` |

> ⚠️ **历史教训：** 前端曾使用 `/api/v1/departments` 但后端注册的是 `/api/v1/depts`，导致 404。务必对照上表确认路径。

### 2.5 新接口开发检查清单

每次新增或修改接口时，**必须** 逐项检查以下清单：

- [ ] **1. 后端路由注册：** 新增的路由文件必须在 `main.py` 的 `include_router` 中注册，且 prefix 正确
- [ ] **2. 前端路径匹配：** `src/api/*.ts` 中新增的接口调用路径，必须与后端注册路径完全一致（逐字符对比）
- [ ] **3. 字段名一致性：** 后端返回的 JSON 字段名，必须与前端 TypeScript `interface` 定义完全一致
- [ ] **4. 响应格式同步：** 如果修改了统一响应格式（如改变包装结构），必须同步更新前端 `src/api/request.ts` 中的拦截器
- [ ] **5. 错误码使用：** 使用标准错误码，不要随意自定义未记录的错误码
- [ ] **6. 本地联调验证：** 使用 curl 或 Postman 验证接口返回格式正确
- [ ] **7. 前端类型检查：** 确保前端 TypeScript 无类型错误（`npm run build` 通过）

---

## 3. 认证与权限

### 3.1 JWT Token 机制

系统采用双 Token 认证方案：

| Token 类型 | 用途 | 有效期 | 存储位置 |
|-----------|------|--------|---------|
| `access_token` | API 请求认证 | 较短（建议 30 分钟） | 前端 localStorage |
| `refresh_token` | 刷新 access_token | 较长（建议 7 天） | 前端 localStorage |

**认证流程：**

```
1. 用户登录 → 后端返回 access_token + refresh_token
2. 前端在请求头中携带: Authorization: Bearer <access_token>
3. access_token 过期 → 前端自动使用 refresh_token 刷新
4. refresh_token 过期 → 跳转登录页面
```

**后端实现：**
- Token 生成：使用 `python-jose` 的 `jwt.encode()`
- Token 验证：通过 FastAPI 依赖注入 `Depends(get_current_user)`
- Token 刷新：`/api/v1/auth/refresh` 接口

### 3.2 钉钉 SSO 登录流程

```
1. 用户访问登录页 → 点击"钉钉登录"
2. 前端重定向到钉钉授权页面
3. 用户在钉钉中授权 → 钉钉回调到后端
4. 后端使用钉钉授权码换取用户信息
5. 后端在本地数据库创建/更新用户记录
6. 后端生成 JWT Token 并返回前端
7. 前端存储 Token 并跳转到首页
```

### 3.3 角色定义

| 角色 | 标识 | 权限范围 |
|------|------|---------|
| 超级管理员 | `super_admin` | 系统全部权限，可管理所有部门和用户 |
| Agent 管理员 | `agent_admin` | 管理 Agent 的创建、配置、发布 |
| 部门管理员 | `dept_admin` | 管理本部门的用户和 Agent |
| 开发者 | `developer` | 创建和开发 Agent，查看监控数据 |
| 普通用户 | `user` | 使用已发布的 Agent |

### 3.4 RBAC 中间件使用方式

**后端使用方法：**

```python
from app.dependencies import get_current_user, require_roles

# 方式1: 仅验证登录（任何已登录用户可访问）
@router.get("/profile")
async def get_profile(user=Depends(get_current_user)):
    return user

# 方式2: 要求特定角色
@router.get("/admin/dashboard")
async def admin_dashboard(user=Depends(require_roles(["super_admin", "agent_admin"]))):
    return {"dashboard": "admin data"}
```

**前端使用方法：**

```typescript
// 在路由守卫中检查权限
router.beforeEach((to, from, next) => {
    const userStore = useUserStore()
    const requiredRoles = to.meta.roles as string[]
    
    if (requiredRoles && !requiredRoles.some(role => userStore.roles.includes(role))) {
        next('/403')
    } else {
        next()
    }
})
```

> ⚠️ **历史教训：** 确保 `roles` 表中有初始数据。如果角色表为空，所有 RBAC 检查都会失败，导致所有需要权限的接口返回 403。

---

## 4. 部署流程

### 4.1 服务器信息

| 项目 | 值 |
|------|-----|
| 服务器 IP | 120.77.223.91 |
| SSH 连接 | `ssh root@120.77.223.91` |
| 域名 | agent.suncentgroup.com |
| 前端路径 | `/opt/app/frontend/` |
| 后端代码（容器内） | `/app/app/` |
| Nginx 配置 | `/etc/nginx/conf.d/` 或 `/etc/nginx/sites-available/` |

### 4.2 前端部署

```bash
# 1. 进入前端项目目录
cd /opt/app/frontend

# 2. 拉取最新代码（如使用 Git）
git pull origin main

# 3. 安装依赖（如有新增）
npm install

# 4. 构建
npm run build

# 5. 构建产物自动输出到 dist/ 目录，Nginx 直接引用该目录
```

### 4.3 后端部署

```bash
# 1. 重启后端容器
docker restart agenthub-backend

# 2. 查看后端日志（确认启动成功）
docker logs -f agenthub-backend --tail 100

# 3. 如果更新了代码，需要重新构建镜像
docker build -t agenthub-backend:latest /path/to/backend/
docker restart agenthub-backend
```

### 4.4 Nginx 配置

Nginx 配置文件位于宿主机：

```
/etc/nginx/conf.d/agenthub.conf
```

关键配置要点：
- 前端静态文件指向 `/opt/app/frontend/dist/`
- API 请求反向代理到 `http://127.0.0.1:8000`（后端容器映射端口）
- WebSocket 代理（如有实时通信功能）

```bash
# 修改 Nginx 配置后重新加载
nginx -t          # 检查配置语法
nginx -s reload   # 热重载
```

### 4.5 数据库操作

```bash
# 进入 PostgreSQL 容器
docker exec -it agenthub-postgres psql -U postgres -d agenthub

# 常用 SQL 操作
\dt                  -- 列出所有表
SELECT * FROM users;  -- 查询用户
\d users              -- 查看表结构

# 退出
\q
```

### 4.6 Redis 操作

```bash
# 进入 Redis 容器
docker exec -it agenthub-redis redis-cli

# 常用命令
KEYS *              -- 查看所有 key
GET <key>           -- 获取值
DEL <key>           -- 删除 key
FLUSHDB             -- 清空当前数据库（慎用！）
```

---

## 5. 测试验证清单

### 5.1 新功能上线前验证

每次上线前，**必须** 完成以下检查：

- [ ] **本地开发环境测试通过**
- [ ] **接口联调测试**：使用 curl 测试所有新增/修改的接口
- [ ] **前端构建无错误**：`npm run build` 成功
- [ ] **前端 TypeScript 无报错**：无类型错误
- [ ] **后端启动无异常**：`docker logs` 无 ERROR
- [ ] **认证流程正常**：登录、Token 刷新、退出登录
- [ ] **权限控制正常**：不同角色的用户访问权限正确
- [ ] **浏览器兼容性**：Chrome 和 Edge 测试通过
- [ ] **Nginx 配置正确**：静态文件和 API 代理均正常

### 5.2 接口联调方法（curl 测试）

**登录测试：**
```bash
# 获取 Token
curl -X POST https://agent.suncentgroup.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "xxx"}'

# 预期返回:
# {"code": 0, "data": {"access_token": "xxx", "refresh_token": "xxx"}, "message": "ok", "timestamp": ...}
```

**认证接口测试：**
```bash
# 使用 Token 访问受保护接口
TOKEN="your_access_token_here"

curl -X GET https://agent.suncentgroup.com/api/v1/users/profile \
  -H "Authorization: Bearer $TOKEN"

# 预期返回:
# {"code": 0, "data": {"id": 1, "display_name": "管理员", ...}, "message": "ok", "timestamp": ...}
```

**检查响应格式：**
```bash
# 验证返回的是统一格式
curl -s https://agent.suncentgroup.com/api/v1/auth/login \
  -X POST -H "Content-Type: application/json" \
  -d '{}' | python3 -m json.tool

# 确认字段: code, data, message, timestamp 都存在
```

### 5.3 前端构建验证

```bash
cd /opt/app/frontend

# TypeScript 类型检查
npx vue-tsc --noEmit

# 构建
npm run build

# 检查构建产物
ls -la dist/
```

---

## 6. 常见问题及解决方案

> 以下是实际开发中遇到的问题案例，记录于此以避免重复踩坑。

### 问题 1：前端获取 Token 失败 — 响应未正确解包

**现象：** 前端 store 中 `res.access_token` 为 `undefined`，无法保存登录状态。

**原因：** 前端 Axios 响应拦截器未解包。后端返回格式为 `{code: 0, data: {access_token: "...", refresh_token: "..."}, ...}`，但前端 store 直接使用 `res.access_token`，而实际 Token 在 `res.data.access_token` 中。

**解决方案：** 修改 `src/api/request.ts` 中的响应拦截器，在 `code === 0` 时 `return res.data`，使调用者直接获取业务数据。

**预防：** 所有前端 API 调用代码必须基于"拦截器已解包"的前提编写，不要二次取 `.data`。

---

### 问题 2：用户角色格式不匹配 — 对象 vs 字符串

**现象：** 前端权限判断失败，所有用户都被视为无角色。

**原因：** 后端 `UserOut` schema 返回的 `roles` 为对象数组 `[{name: "super_admin"}, {name: "developer"}]`，但前端期望的是字符串数组 `["super_admin", "developer"]`。

**解决方案：** 统一为字符串数组。修改后端 `UserOut` 的 `roles` 字段，在序列化时只提取角色名称：

```python
# 后端修改
class UserOut(BaseModel):
    roles: list[str]  # 返回 ["super_admin", "developer"]
```

或在前端做转换：

```typescript
// 前端转换
const roles = data.roles.map((r: any) => r.name || r)
```

**预防：** 接口文档中明确写出 `roles` 的数据格式。建议使用简单的字符串数组，减少嵌套复杂度。

---

### 问题 3：用户信息字段名不一致 — display_name vs nickname

**现象：** 用户个人信息页面显示空白，头像加载失败。

**原因：** 后端返回的字段为 `display_name` 和 `avatar_url`，但前端 TypeScript 类型定义为 `nickname` 和 `avatar`，导致字段取值为 `undefined`。

**解决方案：** 统一字段命名。修改前端 TypeScript interface 与后端保持一致：

```typescript
// 统一使用后端的字段名
interface UserInfo {
    display_name: string   // 不是 nickname
    avatar_url: string     // 不是 avatar
    // ...
}
```

**预防：** 开发新接口时，前后端先约定字段名并写入文档。优先参考数据库字段命名。

---

### 问题 4：API 路径不一致 — departments vs depts

**现象：** 部门列表接口返回 404。

**原因：** 前端 `src/api/depts.ts` 中请求路径为 `/api/v1/departments`，但后端路由注册为 `/api/v1/depts`。

**解决方案：** 修改前端请求路径与后端一致：

```typescript
// 修改前（错误）
export const getDepartments = () => request.get('/api/v1/departments')

// 修改后（正确）
export const getDepartments = () => request.get('/api/v1/depts')
```

**预防：** 严格遵守 [2.4 API 路径规范](#24-api-路径规范)。新增路由后必须对照后端 `main.py` 中的 `include_router` 注册路径，逐字符确认前端调用路径匹配。

---

### 问题 5：RBAC 权限全部失败 — 角色表为空

**现象：** 所有需要权限的接口都返回 403，包括超级管理员。

**原因：** 数据库 `roles` 表为空，没有任何角色记录。RBAC 中间件查询角色时返回空结果，导致权限校验始终失败。

**解决方案：** 初始化角色数据：

```sql
INSERT INTO roles (id, name, description, created_at, updated_at) VALUES
(1, 'super_admin', '超级管理员', NOW(), NOW()),
(2, 'agent_admin', 'Agent管理员', NOW(), NOW()),
(3, 'dept_admin', '部门管理员', NOW(), NOW()),
(4, 'developer', '开发者', NOW(), NOW()),
(5, 'user', '普通用户', NOW(), NOW());
```

**预防：** 
1. 数据库迁移脚本中必须包含初始角色数据的种子数据（seed data）
2. 部署后首次启动时执行健康检查，确认角色表非空
3. 在部署文档中增加"初始化数据"步骤

---

### 问题 6：前端请求了不存在的监控接口

**现象：** 监控页面部分功能报 404 错误。

**原因：** 后端 `monitor` 模块只实现了 3 个接口，但前端代码中请求了 9 个接口（6 个不存在的接口返回 404）。

**解决方案：**
1. **短期方案：** 前端移除对未实现接口的调用，或添加接口存在性检查
2. **长期方案：** 后端补齐剩余 6 个监控接口

**预防：**
1. 前后端接口开发需同步对齐，使用接口文档（如 Swagger/OpenAPI）作为唯一事实来源
2. 前端访问不存在的接口时，应给出友好提示而非静默失败
3. 在 `request.ts` 中添加 404 的统一处理逻辑

---

## 附录

### A. 快速命令参考

```bash
# SSH 连接
sshpass -p '***' ssh -o StrictHostKeyChecking=no root@120.77.223.91

# 前端构建
cd /opt/app/frontend && npm run build

# 后端重启
docker restart agenthub-backend

# 查看后端日志
docker logs -f agenthub-backend --tail 200

# 查看容器状态
docker ps -a | grep agenthub

# Nginx 重载
nginx -t && nginx -s reload

# 数据库连接
docker exec -it agenthub-postgres psql -U postgres -d agenthub

# Redis 连接
docker exec -it agenthub-redis redis-cli
```

### B. 文件路径速查

| 文件/目录 | 路径 | 说明 |
|----------|------|------|
| 后端入口 | `/app/app/main.py` | FastAPI 应用入口，路由注册 |
| 前端请求封装 | `src/api/request.ts` | Axios 拦截器配置 |
| 前端 API 定义 | `src/api/*.ts` | 各模块 API 调用 |
| 前端类型定义 | `src/types/*.ts` | TypeScript 接口定义 |
| Nginx 配置 | `/etc/nginx/conf.d/` | 反向代理配置 |
| Docker Compose | 项目根目录 `docker-compose.yml` | 容器编排（如使用） |

### C. 相关文档

- FastAPI 官方文档: https://fastapi.tiangolo.com/
- Vue 3 官方文档: https://vuejs.org/
- Ant Design Vue: https://antdv.com/
- Pinia 状态管理: https://pinia.vuejs.org/
