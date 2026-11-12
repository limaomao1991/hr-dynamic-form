# HR 动态表单与通用接口引擎

全栈开发工程师考核完整实现（禁止 Java，使用纯 Node.js + 原生前端）。

## 在线体验

```bash
cd backend
PORT=3000 node src/server.js
# 浏览器打开 http://localhost:3000
```

API Base: `http://localhost:3000`

---

## 考点覆盖清单

| 题目 | 实现位置 | 说明 |
|------|----------|------|
| **场景一 Q0 Schema 设计** | `docs/SCHEMA_DESIGN.md` + seed 数据 | 支持 key/label/type、校验、嵌套 object/array、defaultValue、version、deprecated |
| **Q1 前端渲染引擎** | `frontend/public/app.js` | 动态渲染、repeater 增删、Rule Engine、字段/表单错误、Dirty/Loading、Server Error |
| **Rule Engine 防循环** | `validator.js` + `app.js` | maxIterations + visited set，阻止 A→B→C→A |
| **Q2 Schema 生命周期** | `schemaService.js` | 已发布不可直接改 → 新 draft version；历史 submission 锁 schema_version |
| **场景二 Q0 通用提交** | `POST /api/v1/dynamic-data/:formId` | Schema 驱动校验，详细错误，持久化 SQLite JSON |
| **Q1 查询筛选** | `GET /api/v1/dynamic-data/:formId?filter=...` | json_extract 动态条件 + 分页 |
| **Q2 字段联动** | Schema `visibleWhen` / `requiredWhen` + rules | 应届生隐藏前公司、薪资>30000 显示审批提示 |
| **Q3 文件上传安全** | `fileService.js` + `POST /api/v1/files/upload` | 文件独立存储，Submission 只存 fileId |
| **Q4 钩子/插件** | `hooks.js` | beforeValidate/afterSave 等，内置钉钉/PDF/考勤 mock 插件 |
| **Q5 并发修改** | `PUT /api/v1/schemas/:formId` + expectedVersion / If-Match | 乐观锁，冲突返回 409 |
| **Q6 审计日志** | `audit_logs` 表 + 字段级 diff | 谁在何时改了什么（salary: 30000→35000） |

---

## 架构

```
backend/
  src/
    server.js          # HTTP 路由（零第三方依赖）
    db.js              # node:sqlite
    schemaService.js   # Schema 版本 / 发布 / 乐观锁
    submissionService.js
    validator.js       # 校验 + Rule Engine
    fileService.js
    hooks.js           # 插件系统
frontend/public/       # 原生 SPA 动态表单引擎
docs/SCHEMA_DESIGN.md
```

**技术选型**：
- 运行时：Node.js ≥ 22（内置 `node:sqlite`）
- 无 npm 依赖（便于部署）
- 前端：原生 JS + CSS（亦可替换为 React/Vue）

---

## 核心 API

```
# Schema
GET    /api/v1/schemas/:formId
GET    /api/v1/schemas/:formId/versions
GET    /api/v1/schemas/:formId/versions/:v
POST   /api/v1/schemas                    # 创建/更新 draft
PUT    /api/v1/schemas/:formId            # 更新 draft（乐观锁）
POST   /api/v1/schemas/:formId/publish
GET    /api/v1/schemas/:formId/audit

# Dynamic Data
POST   /api/v1/dynamic-data/:formId
GET    /api/v1/dynamic-data/:formId?filter=expectedSalary>30000&page=1
GET    /api/v1/dynamic-data/:formId/:id
PATCH  /api/v1/dynamic-data/:formId/:id
GET    /api/v1/dynamic-data/:formId/:id/audit

# Files
POST   /api/v1/files/upload
GET    /api/v1/files/:id
GET    /api/v1/files/:id/download
```

---

## 演示流程建议

1. 打开首页，填写「技术岗入职信息表」
   - 选择「是否应届生 = 否」→ 出现「前公司名称」且变为必填
   - 期望薪资填 35000 → 出现 HRBP 审批提示
   - 工作履历可动态添加多条
2. 提交后查看 Hooks 触发结果（钉钉/PDF/考勤）
3. 「提交列表」用 `expectedSalary>30000` 筛选
4. 「Schema 管理」修改 JSON → 保存 Draft → 发布新版本
5. 模拟并发：两个浏览器同时打开 Schema 管理，A 先保存，B 再保存应收到 409
6. 「审计日志」查看变更记录

---

## Schema 版本策略（Q2 详细）

```
v1 published  ──有 10000 条 submission 引用──
     │
HR 修改 employeeName → fullName，删 nickname，加 department
     │
创建 draft v2（可反复编辑）
     │
publish → v2 published，v1 变为 archived
     │
新 submission 使用 v2
历史 submission 仍绑定 schema_version=1，渲染时用旧 Schema
```

字段废弃：在 Schema 中标记 `deprecated: true`，新表单不渲染，但旧数据仍可读取。

---

## 部署说明

本实现零外部依赖，直接：

```bash
git clone <repo>
cd hr-dynamic-form/backend
PORT=3000 node src/server.js
```

生产建议：
- 反向代理 Nginx
- SQLite → PostgreSQL JSONB（仅需改 db 层）
- 文件存储换 S3/OSS
- 增加鉴权（JWT / Session）

提交物：本 repo + 在线地址。
```
