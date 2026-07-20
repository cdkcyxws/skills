# FastAPI Patterns

本参考文件记录 FastAPI 后端落地模式。需要新建 feature、重构接口、审查代码边界或补测试时读取。

## 1. 应用入口模式

应用级资源在 `app/core/lifespan.py` 中统一管理：

```python
from collections.abc import AsyncIterator
from contextlib import AsyncExitStack, asynccontextmanager

from fastapi import FastAPI
from loguru import logger

from app.client.object_storage import ObjectStorageClient
from app.client.redis import RedisClient
from app.core.config import get_settings
from app.core.log import setup_logging


@asynccontextmanager
async def lifespan(application: FastAPI) -> AsyncIterator[None]:
    """
    管理应用级共享资源。

    :param application: FastAPI 应用实例。
    :return: 应用生命周期异步迭代器。
    """
    settings = get_settings()
    setup_logging(component="api")

    try:
        async with AsyncExitStack() as stack:
            redis_client = RedisClient(settings.redis)
            await redis_client.connect()
            stack.push_async_callback(redis_client.close)

            object_storage_client = ObjectStorageClient(settings.object_storage)
            await object_storage_client.connect()
            stack.push_async_callback(object_storage_client.close)

            application.state.redis_client = redis_client
            application.state.object_storage_client = object_storage_client
            yield
    finally:
        await logger.complete()
```

`app/main.py` 只做应用组装：

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.core.config import get_settings
from app.core.exception import register_exception_handler
from app.core.lifespan import lifespan
from app.core.middleware import RequestContextMiddleware
from app.router import include_router


def create_app() -> FastAPI:
    """
    创建 FastAPI 应用实例。

    :return: 组装并配置完成的应用实例。
    """
    settings = get_settings()
    application = FastAPI(
        title="项目 API",
        description="项目后端 API。",
        version=settings.app.version,
        debug=settings.app.debug,
        docs_url="/docs" if settings.app.docs_enabled else None,
        redoc_url="/redoc" if settings.app.docs_enabled else None,
        openapi_url="/openapi.json" if settings.app.docs_enabled else None,
        lifespan=lifespan,
    )
    application.add_middleware(
        CORSMiddleware,
        allow_origins=settings.app.cors_allow_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    application.add_middleware(RequestContextMiddleware)
    include_router(application)
    register_exception_handler(application)
    return application


app = create_app()
```

`app/router.py` 只注册 feature router：

```python
from fastapi import APIRouter, FastAPI

from app.core.config import get_settings
from app.feature.health.router import router as health_router
from app.feature.user.router import router as user_router


def include_router(app: FastAPI) -> None:
    """
    注册应用路由。

    :param app: FastAPI 应用实例。
    """
    settings = get_settings()

    # 运维探活端点按部署需要保留在根路径。
    app.include_router(health_router)

    api_router = APIRouter(prefix=settings.app.api_prefix)
    api_router.include_router(user_router)
    app.include_router(api_router)
```

全局 `/api` 或 `/api/v1` 前缀应集中在这里处理，不要散落到每个 feature router。

生命周期规则：

- 日志必须先于外部客户端初始化，确保启动失败可以被记录。
- `AsyncExitStack` 按注册的相反顺序释放资源；后续客户端初始化失败时，已成功初始化的客户端仍会关闭。
- 客户端对象放入 `app.state` 或通过依赖注入提供，不在模块导入阶段建立连接。
- 启用 `enqueue=True` 时，在所有客户端关闭后执行 `await logger.complete()`，避免退出时丢失排队日志。
- `docs_enabled`、`debug` 由配置决定，生产环境默认均为 False。

## 2. 配置模式

配置文件使用 `config/config.toml`，配置模型和 TOML 结构保持一致。

关键约定：

- 每个配置模型设置 `model_config = ConfigDict(extra="forbid")`。
- 密码、secret、access key 使用 `SecretStr`。
- `environment` 使用 `Literal["dev", "test", "prod"]` 等受约束类型。
- 正式 TOML 只存放非敏感默认值，凭据由环境变量、secret mount 或密钥服务注入。
- `SecretStr` 只减少 repr、异常和调试输出中的明文暴露，不代表配置文件可以提交真实凭据。
- 衍生 URL 使用 property 生成，不散落在业务代码。
- `load_settings(config_path: Path | None = None)` 支持测试传入配置路径。
- `get_settings()` 使用 `@lru_cache`。

应用和日志配置示例：

```python
class AppConfig(BaseModel):
    """
    应用运行配置。
    """

    model_config = ConfigDict(extra="forbid")

    environment: Literal["dev", "test", "prod"] = "dev"
    version: str
    api_prefix: str = "/api"
    debug: bool = False
    docs_enabled: bool = False
    cors_allow_origins: list[str] = Field(default_factory=list)


class LogConfig(BaseModel):
    """
    日志输出配置。
    """

    model_config = ConfigDict(extra="forbid")

    level: str = "INFO"
    rotation: str = "00:00"
    retention: str = "30 days"
    dir: str = "logs"
    file_enabled: bool = True
    enqueue: bool = True
    backtrace: bool = False
    diagnose: bool = False
```

示例：

```python
class MySQLConfig(BaseModel):
    """
    MySQL 连接配置。
    """

    model_config = ConfigDict(extra="forbid")

    host: str
    port: int
    username: str
    password: SecretStr
    database: str
    charset: str

    @property
    def database_url(self) -> str:
        username = quote_plus(self.username)
        password = quote_plus(self.password.get_secret_value())
        database = quote_plus(self.database)
        return f"mysql+asyncmy://{username}:{password}@{self.host}:{self.port}/{database}?charset={self.charset}"
```

配置加载后还要强制生产安全默认值：生产环境不得开启 `debug` 或 Loguru `diagnose`。若部署需要开放 API 文档，必须显式配置并记录访问控制方案，不能仅依赖路径隐藏。

## 3. 路径工具模式

项目内文件路径统一通过 `app/utils/path_utils.py`：

```python
BASE_DIR = Path(__file__).resolve().parent.parent.parent


def _resolve_project_path(rel_path: str | None = None) -> Path:
    """
    解析项目内相对路径为绝对路径。

    :param rel_path: 相对于项目根目录的路径，为 None 时返回项目根目录。
    :return: 解析后的绝对路径。
    :raises ValueError: 路径为绝对路径或超出项目根目录时抛出。
    """
    if rel_path is None:
        return BASE_DIR
    candidate = Path(rel_path)
    if candidate.is_absolute():
        raise ValueError("项目路径必须使用相对路径")
    resolved_path = (BASE_DIR / candidate).resolve()
    if resolved_path != BASE_DIR and BASE_DIR not in resolved_path.parents:
        raise ValueError("项目路径不能超出项目根目录")
    return resolved_path
```

保留两个入口：

- `get_os_base_join()` 给只接受字符串路径的第三方库。
- `get_pathlib_base_join()` 给现代文件读写和路径判断。

## 4. 数据库模式

`app/core/database.py`：

```python
async_engine = create_async_engine(
    settings.mysql.database_url,
    echo=settings.app.debug,
    pool_pre_ping=True,
)

AsyncSessionLocal = async_sessionmaker(
    bind=async_engine,
    class_=AsyncSession,
    expire_on_commit=False,
)


async def get_async_session() -> AsyncGenerator[AsyncSession]:
    """
    获取异步数据库会话。

    :return: 异步 SQLAlchemy 会话生成器。
    """
    async with AsyncSessionLocal() as session:
        yield session
```

基础模型 mixin：

```python
class Base(AsyncAttrs, DeclarativeBase):
    """
    SQLAlchemy 声明式模型基类。
    """


class UUIDPrimaryKeyMixin:
    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=get_uuid)


class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=get_utc_now, nullable=False)
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=get_utc_now, onupdate=get_utc_now, nullable=False)
```

模型规则：

- 表名使用单数，例如 `user`、`workspace`。
- 关联表使用业务单数组合，例如 `workspace_member`。
- 外键字段显式 `index=True`。
- 需要响应带关系字段时，查询使用 `selectinload` 并在 refresh 时指定 `attribute_names`。

## 5. Alembic 模式

Alembic 环境要自动导入所有 feature model，避免自动迁移遗漏：

```python
def import_feature_model() -> None:
    """
    导入所有 feature 模型。

    避免 Alembic 自动迁移遗漏模型。
    """
    feature_dir = get_pathlib_base_join("app/feature")
    for model_path in feature_dir.glob("*/model.py"):
        feature_name = model_path.parent.name
        import_module(f"app.feature.{feature_name}.model")
```

异步数据库迁移使用 `async_engine_from_config`，数据库 URL 从正式配置读取。

迁移文件是应用发布的一部分，必须纳入 Git。禁止在 `.gitignore` 中忽略 `migrations/` 或 `alembic/versions/`；生产启动只执行经过审查的迁移，不调用 `create_all()` 或 ORM 自动建表替代迁移。

## 6. Feature 新增流程

新增一个业务域时按顺序做：

1. 创建 `app/feature/<feature>/`。
2. 写 `schema.py`：请求、响应、字段校验、公开 DTO。
3. 写 `model.py`：SQLAlchemy 2.0 async 风格模型和关系。
4. 写 `service.py`：查询、权限、业务规则、状态变更。
5. 写 `router.py`：URL、依赖、OpenAPI、状态码、响应模型。
6. 在 `app/router.py` 注册 router。
7. 新增 Alembic 迁移。
8. 补 `tests/test_<feature>.py`。
9. 更新 `md/api/README.md`，必要时更新架构、数据库、部署和阶段文档。

不要只新增空 router 或占位 service。一个 feature 应尽量形成可运行闭环。

feature 可以按业务需要扩展：

- `dependency.py`：仅该业务域使用的 FastAPI 依赖。
- `constant.py`、`status.py`：领域常量、状态枚举和错误码定义。
- `prompt/`：AI feature 自己维护的 Prompt 模板和版本说明。
- `utils/`：只服务该 feature 的纯工具；被多个 feature 使用后再评估上移。

这些扩展不能替代 router/schema/service/model 的职责，也不能把业务规则重新堆进 `utils.py` 或 `status.py`。

## 7. Router 模式

路径前缀分三层：

```text
全局 API 前缀：/api 或 /api/v1，在 app/router.py 统一挂载
feature 前缀：/workspace、/user、/auth，在 feature router 中声明
方法路径：""、"/{workspace_id}"、"/{workspace_id}/member"，在具体 route 中声明
```

推荐写法：

```python
router = APIRouter(prefix="/workspace", tags=["工作空间"])


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
async def create_workspace_by_user(
    request: WorkspaceCreateRequest,
    session: Annotated[AsyncSession, Depends(get_async_session)],
    current_user: Annotated[User, Depends(get_current_user)],
) -> Workspace:
    """
    创建工作空间。

    :param request: 工作空间创建请求。
    :param session: 异步数据库会话。
    :param current_user: 当前登录用户。
    :return: 创建后的工作空间。
    """
    workspace = await create_workspace(session, request.name, request.description, current_user)
    await session.commit()
    await session.refresh(workspace)
    return workspace
```

`app/router.py` 再统一挂载：

```python
def include_router(app: FastAPI) -> None:
    settings = get_settings()
    api_router = APIRouter(prefix=settings.app.api_prefix)
    api_router.include_router(auth_router)
    api_router.include_router(user_router)
    api_router.include_router(workspace_router)
    app.include_router(api_router)
```

router 可以做：

- `model_dump(exclude_unset=True)` 区分未传字段和显式传 `null`。
- `commit`、最终 `refresh` 和 204 空响应。
- 构造仅本次返回的响应字段，例如明文 token。
- 写完整的 ReDoc 友好接口元信息。

router 不要做：

- 权限矩阵判断。
- 复杂数据库查询。
- 密码哈希、token 解析等安全细节。
- 大段跨实体业务流程。

prefix 避免写法：

```python
router = APIRouter(prefix="/api/workspace", tags=["工作空间"])
router = APIRouter(prefix="/api/v1/workspace", tags=["工作空间"])

@router.get("/api/workspace/{workspace_id}")
async def get_workspace_by_user(...): ...
```

例外规则：

- `/health`、`/health/ready`、`/metrics` 可按部署需求放在根路径，方便负载均衡和监控直接探测。
- 业务接口统一进入 API 前缀，不要一部分是 `/api/user`，一部分是 `/workspace`。
- 如果项目采用多版本并存，可以在 `app/router.py` 建 `v1_router`、`v2_router`，再分别 include feature router；不要让每个 feature 自己拼版本号。

## 8. ReDoc 注释模式

FastAPI 的 ReDoc 页面是接口协作面，不只是装饰。新增或修改接口时同步维护路由元信息。

必填或优先补齐：

- `tags`：业务域中文名，保持稳定。
- `summary`：短动作，通常是动词 + 业务对象。
- `description`：说明用户效果、权限要求、关键规则和典型失败原因。
- `response_description`：说明成功响应是什么。
- `responses`：补充 401、403、404、409、422、503 等重要失败状态。
- `response_model`：使用公开响应 schema，避免泄露内部字段。

推荐描述：

```python
@router.patch(
    "/{workspace_id}/member/{user_id}",
    response_model=WorkspaceMemberPublic,
    summary="修改工作空间成员角色",
    description="工作空间 Owner/Admin 或超级管理员将成员调整为 Admin、Editor 或 Viewer。",
    response_description="修改后的工作空间成员。",
    responses={
        401: {"description": "用户未登录。"},
        403: {"description": "没有权限管理该工作空间。"},
        404: {"description": "工作空间或成员不存在。"},
        422: {"description": "角色值不允许用于该操作。"},
    },
)
async def update_workspace_member_role_by_user(...) -> WorkspaceMember:
    """
    修改工作空间成员角色。

    :return: 修改后的工作空间成员。
    """
```

避免：

```python
@router.patch("/{workspace_id}/member/{user_id}")
async def update(...): ...
```

ReDoc 注释不要替代业务文档。接口变更仍需同步 `md/api/README.md` 或项目约定的 API 文档。

## 9. Schema 模式

响应读 ORM：

```python
class UserPublic(BaseModel):
    """
    用户公开信息。
    """

    model_config = ConfigDict(from_attributes=True)

    id: str
    email: str
    display_name: str
    created_at: datetime
    updated_at: datetime
```

更新请求至少传一个字段：

```python
class WorkspaceUpdateRequest(BaseModel):
    name: str | None = Field(default=None, min_length=1, max_length=100)
    description: str | None = Field(default=None, max_length=500)

    @model_validator(mode="after")
    def validate_has_update(self) -> WorkspaceUpdateRequest:
        if not self.model_fields_set:
            raise ValueError("至少需要传入一个更新字段")
        return self
```

角色值优先用 `Literal` 暴露给 API，用 `StrEnum` 放模型或 service 常量中表达内部规则。

成功响应默认直接返回公开资源 schema。只有现有协议明确要求统一 envelope 时才使用 `ApiResponse[T]` 等泛型 schema；无论是否使用 envelope，都要保留 201、204、400、401、403、404、409、422、503 等真实 HTTP 语义，不能把所有业务失败包装成 HTTP 200。

## 10. Service 模式

service 中常见函数形态：

```python
async def get_active_workspace(session: AsyncSession, workspace_id: str) -> Workspace:
    """
    获取未删除工作空间。

    :param session: 异步数据库会话。
    :param workspace_id: 工作空间 ID。
    :return: 未删除的工作空间。
    :raises AppError: 工作空间不存在或已删除时抛出。
    """
    workspace = await get_workspace_by_id(session, workspace_id)
    if not workspace or workspace.deleted_at is not None:
        raise AppError("workspace.not_found", "工作空间不存在", status_code=404)
    return workspace
```

权限封装成 `ensure_*`：

```python
async def ensure_workspace_manager(
    session: AsyncSession,
    workspace_id: str,
    operator: User,
) -> Workspace:
    """
    确保操作用户有工作空间管理权限。

    :param session: 异步数据库会话。
    :param workspace_id: 工作空间 ID。
    :param operator: 操作用户。
    :return: 活跃的工作空间。
    :raises AppError: 工作空间不存在或没有管理权限时抛出。
    """
    workspace = await get_active_workspace(session, workspace_id)
    if operator.is_super_admin:
        return workspace
    member = await get_workspace_member(session, workspace_id, operator.id)
    if member is None or member.role not in MANAGE_WORKSPACE_ROLE:
        raise AppError("workspace.permission_denied", "没有权限管理该工作空间", status_code=403)
    return workspace
```

状态变更后：

- 需要 ID 或默认值时先 `await session.flush()`。
- 返回前需要 ORM 最新状态时 `await session.refresh(instance)`。
- 关系字段需要响应时 `await session.refresh(instance, attribute_names=["user"])`。
- 删除使用 `await session.delete(instance)` 后 `flush`。

## 11. 安全模式

密码：

- 使用成熟密码哈希库，例如 `pwdlib[argon2]`。
- 校验异常返回 False，避免泄露细节。

JWT：

- payload 包含 `sub`、`type`、`iat`、`exp` 和必要权限快照。
- access 和 refresh 用 `Literal["access", "refresh"]` 区分。
- decode 时校验 token 类型和 `sub`。
- 过期、无效、类型错误都转换为 `AppError`。

FastAPI 登录依赖：

```python
bearer_scheme = HTTPBearer(auto_error=False)


async def get_current_user(
    credentials: Annotated[HTTPAuthorizationCredentials | None, Depends(bearer_scheme)],
    session: Annotated[AsyncSession, Depends(get_async_session)],
) -> User:
    """
    从请求中解析当前登录用户。

    :param credentials: Bearer 认证凭证。
    :param session: 异步数据库会话。
    :return: 当前登录用户。
    :raises AppError: 未登录或 token 无效时抛出。
    """
    if not credentials:
        raise AppError("auth.not_authenticated", "请先登录", status_code=401)
    payload = decode_token(credentials.credentials, "access")
    user = await get_user_by_id(session, str(payload["sub"]))
    return await ensure_active_user(user)
```

## 12. 外部客户端模式

Redis、对象存储、外部 HTTP、模型服务等适配器优先封装在 `app/client/`。`app/core/` 只负责配置、依赖和生命周期组装，feature 通过依赖获取客户端。

客户端规则：

- 构造函数只保存配置，不发起网络连接、模型加载或数据库访问。
- 提供显式 `connect/init` 与 `close/disconnect`，重复调用保持幂等。
- 由 API lifespan 或 worker 启停钩子初始化和释放，不创建导入即连接的模块级单例。
- 请求期间复用连接池和 `httpx.AsyncClient`，为连接、读取、写入和总耗时设置明确 timeout。
- 外部失败转换为稳定的 `AppError` 或 worker 可识别的暂时性/永久性错误，不向上层泄露客户端内部异常和响应正文。
- 客户端对象可以放入 `app.state`，再通过 FastAPI dependency 暴露；测试使用 `dependency_overrides` 或测试 app state 替换。

推荐形态：

```python
class RedisClient:
    """
    Redis 客户端适配器。
    """

    def __init__(self, config: RedisConfig) -> None:
        """
        创建未连接的 Redis 客户端适配器。

        :param config: Redis 配置。
        """
        self._config = config
        self._client: Redis | None = None

    async def connect(self) -> None:
        """
        建立 Redis 连接池。
        """
        if self._client is None:
            self._client = Redis.from_url(self._config.url.get_secret_value())
            await self._client.ping()

    async def close(self) -> None:
        """
        关闭 Redis 连接池。
        """
        client, self._client = self._client, None
        if client is not None:
            await client.aclose()

    def get_client(self) -> Redis:
        """
        获取已初始化的 Redis 客户端。

        :return: Redis 客户端。
        :raises RuntimeError: 客户端尚未初始化时抛出。
        """
        if self._client is None:
            raise RuntimeError("Redis 客户端尚未初始化")
        return self._client
```

## 13. 日志和请求链路模式

日志初始化遵循主 Skill 的双 sink、component 隔离和敏感字段白名单。默认 extra 至少包含 `component` 与 `request_id`，确保启动日志、worker 日志和请求日志使用同一格式。

文件 sink 使用固定活动文件名，不在路径中放 `{time}`：

```text
logs/
├── api.log                     # 当前自然日正在写入
├── api.<轮换时间>.log          # API 历史日志
├── cleanup_worker.log          # 当前清理 worker 日志
└── cleanup_worker.<轮换时间>.log
```

默认 `rotation="00:00"`，每天第一条跨零点日志写入前，Loguru 会把原活动文件重命名为带日期的历史文件，再创建同名活动文件。`retention="30 days"` 只清理历史文件，排查当天问题时始终先打开 `{component}.log`。

HTTP 请求使用纯 ASGI 中间件绑定 request ID：

```python
import re
from time import monotonic
from uuid import uuid4

from loguru import logger
from starlette.datastructures import Headers, MutableHeaders
from starlette.types import ASGIApp, Message, Receive, Scope, Send

REQUEST_ID_PATTERN = re.compile(r"^[A-Za-z0-9][A-Za-z0-9._:-]{0,127}$")


def resolve_request_id(scope: Scope) -> str:
    """
    解析或生成请求 ID。

    :param scope: ASGI 请求作用域。
    :return: 经过校验的请求 ID。
    """
    candidate = Headers(scope=scope).get("x-request-id", "")
    if REQUEST_ID_PATTERN.fullmatch(candidate):
        return candidate
    return str(uuid4())


class RequestContextMiddleware:
    """
    为 HTTP 请求绑定日志上下文并写入响应请求 ID。
    """

    def __init__(self, app: ASGIApp) -> None:
        """
        初始化请求上下文中间件。

        :param app: 下游 ASGI 应用。
        """
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        """
        处理一次 ASGI 调用。

        :param scope: ASGI 请求作用域。
        :param receive: ASGI 消息接收函数。
        :param send: ASGI 消息发送函数。
        """
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        request_id = resolve_request_id(scope)
        started_at = monotonic()
        status_code = 500

        async def send_with_request_id(message: Message) -> None:
            """
            在响应头中加入请求 ID 并捕获状态码。

            :param message: ASGI 响应消息。
            """
            nonlocal status_code
            if message["type"] == "http.response.start":
                status_code = message["status"]
                MutableHeaders(scope=message)["X-Request-ID"] = request_id
            await send(message)

        with logger.contextualize(request_id=request_id):
            try:
                await self.app(scope, receive, send_with_request_id)
            except Exception:
                duration_ms = (monotonic() - started_at) * 1000
                logger.exception(
                    "请求处理失败 method={} path={} duration_ms={:.2f}",
                    scope["method"],
                    scope["path"],
                    duration_ms,
                )
                raise
            else:
                duration_ms = (monotonic() - started_at) * 1000
                logger.info(
                    "请求完成 method={} path={} status_code={} duration_ms={:.2f}",
                    scope["method"],
                    scope["path"],
                    status_code,
                    duration_ms,
                )
```

请求日志规则：

- 只接受长度不超过 128 且由安全字符组成的上游 request ID；其他值重新生成，避免日志注入。
- 不记录 query string、headers、Cookie、请求体或响应体；需要排障时增加经过评审的字段白名单。
- 异常只记录一次。若此中间件负责 access log，应关闭 Uvicorn/Gunicorn 重复 access log；标准库 error log 可通过统一拦截器转发给 Loguru。
- WebSocket 使用独立 connection ID，并遵守相同的敏感信息规则，不记录鉴权子协议、token 或消息体。

## 14. 独立 Worker 模式

FastAPI `BackgroundTasks` 只适用于短时、进程内且允许随进程终止而丢失的任务。长耗时、定时、需要重试或必须持久化的工作使用独立 worker；项目已有 Redis 时可以按需使用 ARQ，但不将 ARQ 设为所有项目的强制依赖。

worker 约定：

- API 只校验请求、创建业务记录并入队，返回可查询的 job ID；不得在 router 中等待长任务完成。
- worker 启动时配置独立 component 日志并初始化客户端，关闭时反向释放客户端并等待 `logger.complete()`。
- 每个任务定义 `job_timeout`、`max_tries` 和退避策略，只对网络超时、限流、短暂不可用等暂时性错误重试。
- 任务按至少一次投递设计。使用数据库唯一约束、任务状态机或 Redis 原子键实现幂等，不依赖“通常只执行一次”。
- 日志记录 job ID、任务类型、attempt、duration、状态和脱敏资源 ID，不记录任务完整 payload。
- cron 任务设置稳定 job ID 或调度幂等键，避免多个调度实例重复执行。

ARQ 配置可以采用以下形态：

```python
class WorkerSettings:
    """
    清理任务 worker 配置。
    """

    functions = [cleanup_expired_upload]
    on_startup = startup_worker
    on_shutdown = shutdown_worker
    max_jobs = 10
    job_timeout = 300
    max_tries = 3
    health_check_interval = 30
    cron_jobs = [cron(cleanup_expired_upload, hour=3, minute=0, unique=True)]
```

具体字段以项目锁定的 ARQ 版本为准。客户端应在 `startup_worker` 放入 worker context，在 `shutdown_worker` 统一关闭，不在每个 job 内重复创建连接池。

## 15. 健康检查模式

运维端点职责分开：

- `/health`：只证明 API 进程和事件循环可响应，不访问外部服务。
- `/health/ready`：带短 timeout 并发检查 MySQL、Redis、对象存储等关键依赖；任一关键依赖失败返回 503。
- `/health/worker`：按需报告 worker 实例心跳、队列深度和任务统计，不与 API liveness 混合。

worker 心跳使用结构化数据并设置 TTL，至少包含 component、instance ID 和更新时间。读取实例列表使用 Redis `SCAN`、显式实例集合或队列框架提供的健康接口，不使用会阻塞 Redis 的 `KEYS`，也不把日志文本当作健康协议解析。心跳超过约定阈值即视为失效，响应不得包含主机凭据、连接串或原始任务参数。

## 16. 测试模式

`tests/conftest.py` 推荐提供隔离客户端：

```python
@pytest_asyncio.fixture
async def client() -> AsyncGenerator[AsyncClient]:
    """
    隔离测试 HTTP 客户端。

    :return: 使用 SQLite 内存库和依赖覆盖的 HTTP 测试客户端。
    """
    engine = create_async_engine(
        "sqlite+aiosqlite://",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    async with engine.begin() as connection:
        await connection.run_sync(Base.metadata.create_all)

    session_local = async_sessionmaker(bind=engine, class_=AsyncSession, expire_on_commit=False)

    async def override_get_async_session() -> AsyncGenerator[AsyncSession]:
        async with session_local() as session:
            yield session

    app.dependency_overrides[get_async_session] = override_get_async_session
    try:
        transport = ASGITransport(app=app)
        async with AsyncClient(transport=transport, base_url="http://test") as test_client:
            yield test_client
    finally:
        app.dependency_overrides.clear()
        await engine.dispose()
```

测试建议：

- helper 函数放在测试文件顶部或测试工具模块，保持流程可读。
- 业务测试走 HTTP 接口，而不是只调用 service。
- 成功路径断言完整响应关键字段。
- 失败路径断言 `status_code` 和 `body["code"]`。
- 权限类接口覆盖未登录、无权限、角色变化、资源不可见。
- 时间类逻辑覆盖过期、未过期和时区。
- 安全类逻辑确认不返回 token hash、密码 hash、secret 等内部字段。
- 日志初始化调用两次后只保留预期 sink，并验证活动文件固定为 `{component}.log`、`rotation="00:00"`、历史文件带轮换日期、`retention="30 days"`、UTF-8 和生产 `diagnose=False`。
- 用内存 sink 捕获日志，传入假的 token、Cookie、Prompt、模型回答和外部响应正文，断言原值没有出现。
- 并发发起带不同 `X-Request-ID` 的请求，断言响应头和日志上下文一一对应；非法或缺失值应生成新 UUID。
- lifespan 测试显式进入生命周期。同步测试使用 `with TestClient(app)`；异步 `ASGITransport` fixture 需要显式配套 lifespan manager，不能假设 transport 自动触发启动和关闭。
- lifespan 覆盖全部客户端成功、后续客户端初始化失败、应用关闭三种情况，断言每个已初始化资源恰好释放一次。
- worker 覆盖成功、永久失败不重试、暂时失败退避、幂等重复投递、超时和心跳过期。
- 配置覆盖 dev、test、prod，断言生产关闭 debug、API 文档和 Loguru diagnose。

## 17. 文档维护模式

根 README 只放简介、快速启动和文档导航。

详细文档放：

```text
md/goal/README.md
md/product/README.md
md/api/README.md
md/todo/README.md
md/architecture/README.md
md/decision/README.md
md/database/README.md
md/deploy/README.md
```

接口变更同步 API 文档。表结构变更同步数据库文档。配置、Docker、外部服务变更同步部署文档。架构、权限、错误响应和存储策略变更同步架构或决策文档。

## 18. 质量门禁

常用命令：

```bash
uv run ruff check .
uv run pytest
uv run ruff format .
```

`pyproject.toml` 推荐包含：

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
pythonpath = ["."]
testpaths = ["tests"]

[tool.ruff]
line-length = 100
src = ["app", "tests"]

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]
```

完成任务前确认：

- ruff 检查通过。
- pytest 通过。
- 文档已同步或明确检查后无需更新。
- Alembic 迁移文件已生成、审查并纳入 Git，`.gitignore` 没有忽略迁移目录。
- 没有提交 `.env`、真实环境配置、缓存、虚拟环境、日志、密钥或 token。
- 检查 staged diff；项目配置了 Gitleaks、detect-secrets 等 secret scanner 时必须执行并处理结果。

## 19. Git 跟踪和提交模式

完成一轮可运行功能、规范沉淀、文档同步或重要修复后，进行 Git 跟踪和提交。

提交前检查：

```bash
git status --short
git diff
git diff --cached
```

暂存规则：

```bash
git add app/feature/workspace/router.py
git add app/feature/workspace/service.py
git add tests/test_workspace.py
git add md/api/README.md
```

不要默认使用 `git add .`。先确认没有 `.env`、真实环境配置、日志、虚拟环境、缓存目录、密钥、token 或无关文件，同时确认迁移文件没有被错误忽略。

提交信息使用：

```text
<type>: <description>
```

示例：

```text
feat: 新增工作空间邀请接口
fix: 修复工作空间成员权限判断
docs: 更新接口注释规范
test: 增加工作空间邀请测试
```

如果项目规则允许 AI 自主管理 Git，检查通过后主动提交。若测试无法运行、存在疑似用户未提交冲突或远端推送失败，在回复中说明当前 Git 状态和阻塞原因。
