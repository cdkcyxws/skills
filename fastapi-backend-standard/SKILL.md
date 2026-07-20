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
│   ├── lifespan.py
│   ├── log.py
│   ├── model.py
│   ├── security.py
│   └── dependency.py
├── client/
│   ├── redis.py
│   └── object_storage.py
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
├── worker/
│   └── cleanup.py
└── utils/
    └── path_utils.py
```

规则：

- `app/main.py` 只创建 FastAPI app、配置中间件、注册路由和异常处理。
- `app/router.py` 统一 include feature router。
- `app/core/` 只放全局基础能力，例如配置、数据库会话、异常、安全、依赖、日志、生命周期和基础模型 mixin。
- `app/feature/<feature>/` 自己维护 `model.py`、`schema.py`、`router.py`、`service.py`。
- feature 可按需要增加 `dependency.py`、`constant.py`、`status.py`、`prompt/`、`utils/`；只服务单一业务域的内容必须留在该 feature 内。
- `app/client/` 用于封装 Redis、对象存储、外部 HTTP、模型服务等第三方客户端；客户端必须显式初始化和关闭，不在模块导入时建立连接。
- `app/worker/` 用于独立进程运行的队列消费者、定时任务和长耗时后台任务；简单项目不需要为了目录完整而创建空目录。
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
- 运行环境使用受约束字段，例如 `Literal["dev", "test", "prod"]`，由配置统一决定 `debug`、`docs_enabled` 和日志行为，不在入口散落环境分支。
- 正式配置文件只保存非敏感默认值；密码、token、API Key、私钥通过环境变量、secret mount 或密钥服务注入。
- `SecretStr` 只避免配置对象意外展示明文，不能替代安全存储和凭据轮换。
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
- 日志默认每天 `00:00` 轮换，可在 `config/config.toml` 中配置。
- 控制台 sink 默认开启，便于 Docker、Kubernetes 和进程管理器采集；文件 sink 由配置控制，默认按项目需要开启。
- 当前活动日志固定使用 `{component}.log`，例如 `api.log`；sink 路径不包含 `{time}`，轮换时由 Loguru 自动给历史文件追加日期，确保当前文件与归档文件一眼可区分。
- API、worker、定时任务使用不同 `component` 名称和文件名，不让多个独立进程轮换同一个日志文件。
- 多副本容器共享同一日志目录时关闭文件 sink，或把实例标识加入文件名；不能依靠 `enqueue=True` 协调彼此独立启动的进程。
- sink 默认使用 `enqueue=True`，避免异步请求被同步写盘阻塞；启用队列后在进程退出前调用 `await logger.complete()`。
- 生产环境强制 `diagnose=False`，避免异常诊断把局部变量中的敏感数据写入日志；`backtrace` 同样通过配置控制。
- 日志格式、级别、输出目标在统一日志模块初始化，不在各 feature 中分散配置。
- 重复调用日志初始化函数不能叠加 sink；初始化前使用 `logger.remove()`，并为缺省上下文设置默认值。
- Uvicorn、Gunicorn 或第三方库继续使用标准库 `logging` 时，统一接管到 Loguru 或明确保留独立 sink；不得同时开启两套 access log 导致重复记录。
- 业务日志使用参数化占位符和字段白名单，不用 f-string 拼接完整对象。

配置示例：

```toml
# config/config.toml
[log]
level = "INFO"
rotation = "00:00"
retention = "30 days"
dir = "logs"
file_enabled = true
enqueue = true
backtrace = false
diagnose = false
```

```python
# app/core/log.py
import sys

from loguru import logger

from app.core.config import get_settings
from app.utils.path_utils import get_pathlib_base_join


def setup_logging(component: str = "api") -> None:
    """
    配置应用日志。

    :param component: 当前进程组件名，用于区分 API 和不同 worker。
    """
    settings = get_settings()
    is_production = settings.app.environment == "prod"
    diagnose = False if is_production else settings.log.diagnose
    backtrace = False if is_production else settings.log.backtrace
    log_format = (
        "{time:YYYY-MM-DD HH:mm:ss.SSS} | {level:<8} | "
        "{extra[component]} | request_id={extra[request_id]} | "
        "{name}:{function}:{line} - {message}"
    )

    logger.remove()
    logger.configure(extra={"component": component, "request_id": "-"})
    logger.add(
        sys.stdout,
        level=settings.log.level,
        format=log_format,
        colorize=None,
        enqueue=settings.log.enqueue,
        backtrace=backtrace,
        diagnose=diagnose,
    )
    if settings.log.file_enabled:
        log_dir = get_pathlib_base_join(settings.log.dir)
        log_dir.mkdir(parents=True, exist_ok=True)
        logger.add(
            log_dir / f"{component}.log",
            level=settings.log.level,
            format=log_format,
            rotation=settings.log.rotation,
            retention=settings.log.retention,
            encoding="utf-8",
            enqueue=settings.log.enqueue,
            backtrace=backtrace,
            diagnose=diagnose,
        )
```

业务代码中直接引入使用：

```python
from loguru import logger

logger.info("用户登录成功 user_id={}", user_id)


async def connect_database_on_startup() -> None:
    """
    在应用启动阶段连接数据库。
    """
    try:
        await connect_database()
    except Exception:
        logger.exception("数据库连接失败")
        raise
```

敏感信息规则：

- 不记录 `Authorization`、Cookie、密码、token、OTT、API Key、私钥或完整连接串。
- 默认不记录完整请求体、响应体、查询参数、上传文件内容、Prompt、模型消息和模型回答。
- 外部服务失败只记录服务名、状态码、耗时、重试次数和脱敏业务 ID；响应正文必须经过明确白名单和脱敏后才能记录。
- 调试问题优先记录类型、长度、数量、哈希摘要或脱敏 ID，不把完整业务对象交给 logger。
- WebSocket 只记录连接 ID、动作类型、状态和耗时，不记录鉴权信息和消息体。

请求链路规则：

- 请求中间件读取合法的 `X-Request-ID`，缺失或不合法时生成 UUID，并在响应头中回传。
- 使用 `logger.contextualize(request_id=...)` 隔离线程和异步任务上下文，禁止手写无法正确 reset 的全局变量。
- access log 只记录 method、path、status code、duration 和 request ID，不记录 query string、headers 或 body。
- 未捕获异常日志必须带 request ID；异常响应仍使用统一安全结构，不向客户端返回堆栈。

## 应用生命周期和后台任务

使用 FastAPI `lifespan` 统一管理应用级资源。

规则：

- 启动时先配置日志，再初始化数据库、Redis、对象存储、模型客户端等共享资源；关闭时按相反顺序释放。
- 禁止在模块导入阶段发起数据库、Redis、模型加载或外部网络连接，避免测试导入和多 worker 启动产生副作用。
- 客户端提供显式 `init/connect` 与 `close/disconnect` 接口，通过 lifespan、依赖或 `app.state` 管理，不依赖隐式全局连接。
- 长耗时、需要重试、要求持久化或定时执行的任务使用独立 worker；FastAPI `BackgroundTasks` 只处理短时、进程内、允许随进程退出丢失的工作。
- worker 任务必须定义超时、最大重试次数、退避策略和幂等键，并记录 job ID、attempt、duration 和最终状态。
- worker 客户端优先在 worker 启停钩子中初始化和释放，不在每个 job 中重复建立连接。
- `/health` 只证明 API 进程存活，`/health/ready` 检查关键外部依赖，`/health/worker` 按需报告 worker 心跳和队列状态。
- worker 心跳必须有 TTL 和过期判定；Redis 查询使用 `SCAN` 或显式实例登记，不在生产使用 `KEYS` 扫描。

## 异常响应

定义统一业务异常：

```python
class AppError(Exception):
    """
    项目业务异常。
    """

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
- 成功响应默认直接使用公开资源 schema；现有项目需要统一 envelope 时使用带泛型的响应 schema，并继续返回语义正确的 HTTP 状态码，不能用 HTTP 200 掩盖业务失败。
- 测试中断言错误码，避免只断言 HTTP 状态码。

## OpenAPI 和 ReDoc 注释

Python 注释和 docstring 使用简体中文，专有名词可保留英文。

docstring 使用 reST 风格（reStructuredText），所有模块、类、函数、方法均须编写 reST 风格 docstring。

reST docstring 写作规则：

- `"""` 独占首行，摘要从第二行开始。
- 摘要后空一行，再写详细说明或 reST 指令。
- `:param name: 描述。` 列出每个参数，顺序与函数签名一致。
- `:return: 描述。` 说明返回值。
- `:raises ExceptionType: 描述。` 列出可能抛出的异常。
- 参数、返回值、异常描述以句号结尾。
- 无参数、无返回值的函数可省略对应指令，仍保留摘要。
- 类 docstring 写于类定义下方（`class Foo:` 之后），方法 docstring 写于方法定义下方。
- 配置模型（Pydantic）和 ORM 模型（SQLAlchemy）的类 docstring 描述建模对象和业务含义。
- 模块级 docstring 放在文件顶部（import 之前），描述模块职责。

基本示例：

```python
async def get_user(user_id: str) -> User:
    """
    根据 ID 获取用户。

    :param user_id: 用户 ID。
    :return: 用户。
    :raises AppError: 用户不存在时抛出。
    """
```

多参数示例：

```python
def create_user(session: AsyncSession, email: str, password: str, display_name: str = "") -> User:
    """
    创建新用户。

    :param session: 异步数据库会话。
    :param email: 用户邮箱，自动转为小写。
    :param password: 原始密码，调用方传入明文，函数内哈希处理。
    :param display_name: 显示名称，默认空。
    :return: 创建后的用户 ORM 实例。
    :raises AppError: 邮箱已注册时抛出。
    """
```

类模型示例：

```python
class UserPublic(BaseModel):
    """
    用户公开信息响应模型。

    仅包含可暴露给 API 消费者的字段，不含密码哈希、内部标记等敏感字段。
    """

    model_config = ConfigDict(from_attributes=True)


class User(Base, UUIDPrimaryKeyMixin, TimestampMixin):
    """
    用户 ORM 模型。

    存储用户账户信息，关联用户认证、工作空间成员等。
    """

    __tablename__ = "user"
```

property 示例：

```python
@property
def is_super_admin(self) -> bool:
    """
    是否为超级管理员。

    :return: True 表示用户为超级管理员。
    """
    return "super_admin" in (self.roles or "")
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
    """
    创建工作空间。
    """
```

## 测试规则

常用测试形态：

- `httpx.AsyncClient` + `ASGITransport(app=app)` 测真实 ASGI 接口。
- 用 `app.dependency_overrides` 替换数据库 session。
- 用 SQLite 内存库或隔离测试数据库创建表。
- 用 helper 函数完成注册、登录、创建资源等重复流程。
- 对权限、重复提交、过期、撤销、不可见范围等失败路径做断言。
- 对统一异常格式、配置加载、路径工具、基础模型 mixin 单独测试。
- 日志测试覆盖重复初始化不产生重复 sink、每天零点轮换、活动文件固定为 `{component}.log`、历史文件带轮换日期、30 天保留、UTF-8 和生产环境 `diagnose=False`。
- 请求链路测试覆盖 request ID 的生成、合法值透传、响应头回传和并发请求上下文隔离。
- 敏感日志测试使用假的 token、Cookie、Prompt 和响应正文，断言日志输出不包含原值。
- lifespan 测试必须显式进入应用生命周期，断言共享客户端各初始化和关闭一次，并覆盖中途初始化失败时的清理。
- worker 测试覆盖成功、幂等重复执行、超时、重试退避和心跳过期。
- 配置测试覆盖 dev、test、prod 的文档开关、debug 和日志安全行为。

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
- Alembic 迁移文件必须纳入 Git，禁止在 `.gitignore` 中忽略整个迁移目录。
- 检查是否误入 `.env`、日志、虚拟环境、缓存、真实配置文件、密钥、token 或无关文件。
- 配置示例只保留占位值；提交前检查 staged diff，项目已配置 secret scanner 时必须运行并处理结果。
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
9. 所有函数、类、模块是否编写 reST 风格 docstring。
10. 配置、路径、密钥是否集中管理。
11. 日志是否集中初始化、区分 component，并在生产关闭 `diagnose`。
12. request ID、access log 和敏感信息脱敏是否有测试覆盖。
13. 共享客户端是否由 lifespan 显式初始化和反向释放，且无导入阶段 I/O。
14. worker 是否定义超时、重试、幂等和带 TTL 的健康心跳。
15. 是否有数据库迁移或迁移说明，且迁移文件已纳入 Git。
16. 是否补齐接口文档和相关 md 文档。
17. 是否运行 `uv run ruff check .` 和 `uv run pytest`。
18. 是否完成 staged diff、敏感信息、Git 状态和必要提交检查。

## 详细参考

需要更细的落地模式、示例代码和约定时，读取：

```text
references/fastapi-patterns.md
```
