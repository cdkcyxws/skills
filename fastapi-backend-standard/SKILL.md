---
name: fastapi-backend-standard
description: 规范 Python FastAPI 后端项目的代码结构、异步开发、配置管理、路由/service/schema/model 分层、APIRouter prefix、ReDoc 友好接口注释、SQLAlchemy 2.0 async、Pydantic v2、统一异常、测试、文档同步、Git 跟踪提交和质量检查。Use when building, reviewing, refactoring, or extending FastAPI API projects with a feature-based Python backend structure.
---

# FastAPI Backend Standard Skill

## 目标

当处理 Python FastAPI 后端项目时，使用本 Skill 统一代码组织和交付质量。

本规范适合需要长期维护、AI 托管开发、接口文档同步、数据库迁移和测试闭环的 FastAPI 项目。

## 开发前检查

接手项目后先读：

1. `AGENTS.md` 或同类协作规则。
2. `README.md`。
3. 和任务相关的 `md/<topic>/README.md`。
4. 已有 feature、测试和迁移。

开发前先判断当前任务属于：

```text
新增业务闭环
修复接口行为
调整配置或基础设施
修改数据模型或迁移
补文档或测试
```

涉及产品效果、权限边界、数据模型、存储策略、部署方式或外部服务行为时，先用产品语言确认预期。

## 技术栈规则

优先使用：

```text
FastAPI
Pydantic v2
SQLAlchemy 2.0 async
Alembic
loguru
pytest + pytest-asyncio
httpx.AsyncClient + ASGITransport
ruff
uv
```

异步优先：

- API 路由优先写 `async def`。
- 数据库使用 `AsyncSession`、`async_sessionmaker`、`create_async_engine`。
- HTTP 调用使用 `httpx.AsyncClient`。
- Redis 使用 `redis.asyncio`。
- 对象存储或第三方 I/O 优先使用 async client 或 async wrapper。
- 配置模型、SQLAlchemy 模型定义、纯函数和 CPU 密集小函数可以保持同步。

依赖管理优先使用：

```bash
uv add <package>
uv add --dev <package>
uv remove <package>
uv sync
uv run <command>
```

不要直接用 `pip install` 维护项目依赖，不要手动编辑锁文件。

## 目录结构

优先采用 feature-based 结构：

```text
app/
├── main.py
├── router.py
├── core/
│   ├── config.py
│   ├── database.py
│   ├── exception.py
│   ├── model.py
│   ├── security.py
│   └── dependency.py
├── feature/
│   ├── health/
│   │   ├── router.py
│   │   ├── schema.py
│   │   └── service.py
│   └── user/
│       ├── model.py
│       ├── router.py
│       ├── schema.py
│       └── service.py
└── utils/
    └── path_utils.py
```

规则：

- `app/main.py` 只创建 FastAPI app、配置中间件、注册路由和异常处理。
- `app/router.py` 统一 include feature router。
- `app/core/` 只放全局基础能力，例如配置、数据库、异常、安全、缓存、对象存储和基础模型 mixin。
- `app/feature/<feature>/` 自己维护 `model.py`、`schema.py`、`router.py`、`service.py`。
- 不创建 `app/model`、`app/schema`、`app/api`、`app/service` 这种集中式技术分层目录。
- 除 `utils` 目录外，项目代码命名优先使用单数名词。

## 分层职责

router：

- 声明 URL、HTTP 方法、依赖、请求体、响应模型、状态码和 OpenAPI 文案。
- 正确使用 `APIRouter(prefix=...)` 分层管理路径，避免把 `/api`、版本号和业务域混在每个 feature router 中。
- 使用 `Annotated[..., Depends(...)]` 注入 session、当前用户、权限依赖。
- 只做轻量请求转换，例如 `request.model_dump(exclude_unset=True)`。
- 不写复杂业务规则、数据库查询和跨实体事务。

路由 prefix 分层规则：

- 全局 API 前缀只配置一次，例如 `settings.app.api_prefix = "/api"` 或 `"/api/v1"`。
- `app/router.py` 负责把全局前缀挂到统一 API router 上。
- feature router 只写业务域前缀，例如 `APIRouter(prefix="/workspace")`，不要写 `"/api/workspace"`。
- 路由方法只写集合、详情和子资源路径，例如 `""`、`"/{workspace_id}"`、`"/{workspace_id}/member"`。
- `/health`、`/metrics` 这类运维端点可以按产品和部署需要留在根路径，但要显式说明，不要和业务 API 前缀混用。
- 如果需要 API 版本，优先把版本放进全局前缀或版本 router，例如 `/api/v1`，不要在每个 feature 里重复写 `/v1`。

推荐：

```python
# app/router.py
api_router = APIRouter(prefix=settings.app.api_prefix)
api_router.include_router(workspace_router)
app.include_router(api_router)

# app/feature/workspace/router.py
router = APIRouter(prefix="/workspace", tags=["工作空间"])
```

避免：

```python
router = APIRouter(prefix="/api/workspace", tags=["工作空间"])
router = APIRouter(prefix="/api/v1/workspace", tags=["工作空间"])
```

schema：

- 放 Pydantic v2 请求、响应和公开 DTO。
- 响应模型读取 ORM 时使用 `ConfigDict(from_attributes=True)`。
- 输入校验使用 `Field`、`field_validator`、`model_validator`。
- 可复用的小标准化函数可以放在 schema，例如邮箱小写清洗。

service：

- 承载业务规则、权限判断、数据库查询和状态变更。
- 业务失败抛 `AppError`，不要在 service 返回模糊 boolean。
- 事务边界保持清晰：常见做法是 service 内 `flush/refresh`，router 或上层闭环负责 `commit`。
- 跨 feature 协作通过 service 函数调用完成，避免 router 串业务。

model：

- 使用 SQLAlchemy 2.0 类型注解：`Mapped[...]` + `mapped_column(...)`。
- 表名、目录名、API 前缀优先单数。
- 通用字段用 mixin，例如 UUID 主键、时间戳、软删除。
- 关系加载要显式考虑 `selectinload`，避免响应序列化时隐式懒加载失败。

## 配置和路径

项目配置建议集中在一个正式配置文件，例如：

```text
config/config.toml
```

规则：

- 使用 Pydantic v2 定义配置模型。
- 配置模型使用 `ConfigDict(extra="forbid")`。
- 密钥和密码字段使用 `SecretStr`。
- 使用 `@lru_cache` 缓存 `get_settings()`。
- 新增配置项时同步更新配置文件、部署说明和必要架构文档。
- 不把数据库、Redis、对象存储 endpoint、bucket、密钥写入业务表。

路径规则：

- 项目内路径统一通过公共路径工具生成。
- `BASE_DIR` 只在路径工具模块定义。
- 传入相对项目根目录的路径。
- 不硬编码绝对路径，不依赖当前工作目录，不手写 Windows 或 Linux 分隔符。

如需更细路径规范，配合使用 `python-path-standard`。

## 日志管理

使用 `loguru` 进行日志管理，在 `app/core/` 中集中配置。

规则：

- 使用 `loguru` 替代标准库 `logging`。
- 日志轮换周期默认 30 天，可在 `config/config.toml` 中配置。
- 日志格式、级别、输出目标在统一日志模块初始化，不在各 feature 中分散配置。
- 不将密钥、密码、token 写入日志。

配置示例：

```toml
# config/config.toml
[log]
level = "INFO"
rotation = "30 days"
retention = "90 days"
dir = "logs"
```

```python
# app/core/log.py
from loguru import logger

from app.core.config import get_settings


def setup_logging() -> None:
    settings = get_settings()
    logger.remove()
    logger.add(
        get_pathlib_base_join(settings.log.dir) / "app_{time:YYYY-MM-DD}.log",
        level=settings.log.level,
        rotation=settings.log.rotation,
        retention=settings.log.retention,
        encoding="utf-8",
    )
```

业务代码中直接引入使用：

```python
from loguru import logger

logger.info("用户登录成功 user_id={}", user_id)
logger.error("数据库连接失败", exc_info=True)
```

## 异常响应

定义统一业务异常：

```python
class AppError(Exception):
    """项目业务异常。"""

    def __init__(self, code: str, message: str, status_code: int = 400, detail: dict | None = None):
        self.code = code
        self.message = message
        self.status_code = status_code
        self.detail = detail or {}
        super().__init__(message)
```

错误响应统一为：

```json
{
  "code": "feature.error_name",
  "message": "用户可读错误信息",
  "detail": {}
}
```

规则：

- 业务错误码使用 `feature.reason` 形式，例如 `auth.invalid_token`。
- 请求校验错误统一转换为固定响应结构。
- 未捕获异常不要泄露堆栈和密钥。
- 测试中断言错误码，避免只断言 HTTP 状态码。

## OpenAPI 和 ReDoc 注释

Python 注释和 docstring 使用简体中文，专有名词可保留英文。

docstring 使用 reST 风格：

```python
async def get_user(user_id: str) -> User:
    """根据 ID 获取用户。

    :param user_id: 用户 ID。
    :return: 用户。
    :raises AppError: 用户不存在时抛出。
    """
```

FastAPI 路由要写：

- `summary`
- `description`
- `response_description`
- 必要的 `responses`
- 合理的 tags

ReDoc 友好规则：

- `summary` 写短动作，不写模糊标题，例如 `创建工作空间`、`撤销工作空间邀请`。
- `description` 写用户效果、权限要求和典型失败原因，面向接口使用者解释结果。
- `response_description` 写成功响应含义，例如 `创建后的工作空间。`。
- `responses` 至少补充重要错误状态，例如 401、403、404、409、422、503。
- tag 使用稳定业务域中文名，例如 `认证`、`用户后台`、`工作空间`。
- 路径参数、查询参数和复杂请求字段需要用 `Path`、`Query`、`Field` 补说明、长度和示例。
- `response_model` 必须使用公开响应 schema，不直接暴露密码哈希、token hash、密钥或内部字段。
- 不为了 ReDoc 展示把业务逻辑塞进 router；router 只保留 HTTP 声明和轻量转换。

推荐：

```python
@router.post(
    "",
    response_model=WorkspacePublic,
    status_code=status.HTTP_201_CREATED,
    summary="创建工作空间",
    description="当前登录用户创建工作空间，并自动成为该工作空间 Owner。",
    response_description="创建后的工作空间。",
    responses={
        401: {"description": "用户未登录。"},
        422: {"description": "请求参数校验失败。"},
    },
)
async def create_workspace_by_user(...) -> Workspace:
    """创建工作空间。"""
```

## 测试规则

常用测试形态：

- `httpx.AsyncClient` + `ASGITransport(app=app)` 测真实 ASGI 接口。
- 用 `app.dependency_overrides` 替换数据库 session。
- 用 SQLite 内存库或隔离测试数据库创建表。
- 用 helper 函数完成注册、登录、创建资源等重复流程。
- 对权限、重复提交、过期、撤销、不可见范围等失败路径做断言。
- 对统一异常格式、配置加载、路径工具、基础模型 mixin 单独测试。

至少运行：

```bash
uv run ruff check .
uv run pytest
```

格式化使用：

```bash
uv run ruff format .
```

## 文档同步

新增或修改接口时，同步更新接口文档。

改变以下内容时同步对应文档：

- 架构、目录、基础设施、错误响应、权限、存储策略。
- 数据表、迁移、实体关系。
- 配置、Docker、部署方式、外部服务连接。
- 阶段计划、产品范围、技术决策。

结束任务前做文档防漂移检查。即使无需修改文档，也在回复中说明已检查。

## Git 跟踪和提交

完成可运行功能闭环、规范沉淀、重要修复或文档同步后，主动做 Git 跟踪和提交。

提交前：

```bash
git status --short
git diff
git diff --cached
```

规则：

- 只暂存本次任务相关文件，优先使用 `git add <file>` 或 `git add <dir>`，不要默认 `git add .`。
- 检查是否误入 `.env`、日志、虚拟环境、缓存、密钥、token 或无关文件。
- 提交信息使用 `<type>: <description>`，描述使用简体中文，例如 `docs: 完善 FastAPI 后端规范`。
- 如果测试或校验无法运行，要在最终回复中说明原因，不要假装通过。
- 推送远端只在用户要求或项目规则授权时执行；推送失败时保留本地提交并说明原因。

## 交付清单

完成 FastAPI 后端任务前检查：

1. 目录是否按 feature-based 结构放置。
2. router 是否只负责 HTTP 声明和轻量转换。
3. router prefix 是否按全局 API 前缀、feature 前缀、方法路径三层拆分。
4. 路由是否具备 ReDoc 友好的 `summary`、`description`、`response_description` 和必要 `responses`。
5. service 是否包含业务规则、权限和事务状态变更。
6. schema 是否覆盖请求、响应和字段校验。
7. model 是否使用 SQLAlchemy 2.0 async 风格和单数命名。
8. 错误是否使用统一 `AppError` 和错误码。
9. 配置、路径、密钥是否集中管理。
10. 是否有数据库迁移或迁移说明。
11. 是否补齐接口文档和相关 md 文档。
12. 是否运行 `uv run ruff check .` 和 `uv run pytest`。
13. 是否完成 Git 状态检查、相关文件暂存和必要提交。

## 详细参考

需要更细的落地模式、示例代码和约定时，读取：

```text
references/fastapi-patterns.md
```
