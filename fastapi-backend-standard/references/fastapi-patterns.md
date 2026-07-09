# FastAPI Patterns

本参考文件记录 FastAPI 后端落地模式。需要新建 feature、重构接口、审查代码边界或补测试时读取。

## 1. 应用入口模式

`app/main.py` 只做应用组装：

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.core.config import get_settings
from app.core.exception import register_exception_handler
from app.router import include_router


def create_app() -> FastAPI:
    """创建 FastAPI 应用实例。"""
    settings = get_settings()
    application = FastAPI(
        title="项目 API",
        description="项目后端 API。",
        version=settings.app.version,
        docs_url="/docs",
        redoc_url="/redoc",
        openapi_url="/openapi.json",
    )
    application.add_middleware(
        CORSMiddleware,
        allow_origins=settings.app.cors_allow_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
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
    """注册应用路由。"""
    settings = get_settings()

    # 运维探活端点按部署需要保留在根路径。
    app.include_router(health_router)

    api_router = APIRouter(prefix=settings.app.api_prefix)
    api_router.include_router(user_router)
    app.include_router(api_router)
```

全局 `/api` 或 `/api/v1` 前缀应集中在这里处理，不要散落到每个 feature router。

## 2. 配置模式

配置文件使用 `config/config.toml`，配置模型和 TOML 结构保持一致。

关键约定：

- 每个配置模型设置 `model_config = ConfigDict(extra="forbid")`。
- 密码、secret、access key 使用 `SecretStr`。
- 衍生 URL 使用 property 生成，不散落在业务代码。
- `load_settings(config_path: Path | None = None)` 支持测试传入配置路径。
- `get_settings()` 使用 `@lru_cache`。

示例：

```python
class MySQLConfig(BaseModel):
    """MySQL 连接配置。"""

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

## 3. 路径工具模式

项目内文件路径统一通过 `app/utils/os_utils.py`：

```python
BASE_DIR = Path(__file__).resolve().parent.parent.parent


def _resolve_project_path(rel_path: str | None = None) -> Path:
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
    async with AsyncSessionLocal() as session:
        yield session
```

基础模型 mixin：

```python
class Base(AsyncAttrs, DeclarativeBase):
    """SQLAlchemy 声明式模型基类。"""


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
    """导入所有 feature 模型，避免 Alembic 自动迁移遗漏模型。"""
    feature_dir = get_pathlib_base_join("app/feature")
    for model_path in feature_dir.glob("*/model.py"):
        feature_name = model_path.parent.name
        import_module(f"app.feature.{feature_name}.model")
```

异步数据库迁移使用 `async_engine_from_config`，数据库 URL 从正式配置读取。

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
    """创建工作空间。"""
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
    """修改工作空间成员角色。"""
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
    """用户公开信息。"""

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

## 10. Service 模式

service 中常见函数形态：

```python
async def get_active_workspace(session: AsyncSession, workspace_id: str) -> Workspace:
    """获取未删除工作空间。"""
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
    if not credentials:
        raise AppError("auth.not_authenticated", "请先登录", status_code=401)
    payload = decode_token(credentials.credentials, "access")
    user = await get_user_by_id(session, str(payload["sub"]))
    return await ensure_active_user(user)
```

## 12. 外部依赖模式

Redis、对象存储、外部 HTTP 等封装在 `app/core/`。

健康检查和就绪检查分开：

- `/health` 只证明 API 进程可响应。
- `/health/ready` 检查 MySQL、Redis、对象存储等外部依赖，失败返回 503。

多个依赖可用 `asyncio.gather` 并发探活。

## 13. 测试模式

`tests/conftest.py` 推荐提供隔离客户端：

```python
@pytest_asyncio.fixture
async def client() -> AsyncGenerator[AsyncClient]:
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

## 14. 文档维护模式

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

## 15. 质量门禁

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
- 没有提交 `.env`、缓存、虚拟环境、日志、密钥。

## 16. Git 跟踪和提交模式

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

不要默认使用 `git add .`。先确认没有 `.env`、日志、虚拟环境、缓存目录、密钥、token 或无关文件。

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
