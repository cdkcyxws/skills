---
name: git-commit-standard
description: 规范 Git 提交信息、仓库初始化、.gitattributes 和 .gitignore 默认配置。
---

# Git Commit Standard Skill

## 目标

当用户需要生成 Git 提交信息、检查提交说明、阶段性保存代码、初始化 Git 仓库时，使用本 Skill。

目标：

1. 提交信息简短清楚。
2. 统一使用 `<type>: <description>` 格式。
3. 不使用 `feat(auth): xxx` 这种模块范围写法。
4. 初始化仓库默认使用 `main` 分支。
5. 初始化仓库时自动检查 `.gitattributes` 和 `.gitignore`。
6. 支持 `wip` 阶段性提交，用于保存未完成代码。
7. 区分配置、依赖、构建、部署等不同类型。
8. 避免 `update code`、`fix bug`、`提交代码` 这类模糊说明。

## 提交格式

统一使用：

```text
<type>: <description>
```

示例：

```text
feat: 新增用户登录功能
fix: 修复语音重复播放问题
docs: 更新部署说明
config: 调整模型服务参数配置
deps: 升级 Python 项目依赖版本
build: 优化 Dockerfile 构建流程
deploy: 调整 docker-compose 服务配置
wip: 接入语音播放队列核心流程
chore: 清理无用文件
```

要求：

1. 使用英文冒号 `:`。
2. 冒号后必须有一个空格。
3. 不使用中文冒号 `：`。
4. 不添加模块括号，如 `feat(auth): xxx`。
5. 描述使用简体中文。
6. 描述要具体、简短，结尾不加句号。

## type 类型

常用类型：

```text
feat: 新增功能
fix: 修复问题
docs: 文档修改
style: 格式调整
refactor: 代码重构
perf: 性能优化
test: 测试相关
config: 配置变更
deps: 依赖变更
build: 构建相关
deploy: 部署运行配置
ci: 流水线相关
chore: 日常维护
wip: 阶段性提交
revert: 回滚提交
release: 版本发布
```

选择规则：

```text
新增功能 -> feat
修复问题 -> fix
修改文档 -> docs
调整格式 -> style
代码重构 -> refactor
性能优化 -> perf
测试相关 -> test
修改 toml、yaml、json、ini、conf、env.example 等配置文件 -> config
修改 requirements、pyproject、poetry.lock、uv.lock、pom.xml、build.gradle 等依赖文件 -> deps
修改 Dockerfile、构建脚本、打包流程 -> build
修改 docker-compose、nginx、systemd、k8s、部署启动配置 -> deploy
流水线修改 -> ci
清理、整理、杂项维护 -> chore
功能未完成但需要阶段性保存或同步 -> wip
回滚提交 -> revert
版本发布 -> release
```

简单判断：

```text
业务能力新增 -> feat
错误行为修复 -> fix
配置参数变化 -> config
依赖版本变化 -> deps
怎么构建变化 -> build
怎么部署或运行变化 -> deploy
不属于以上类型的维护工作 -> chore
```

## wip 阶段性提交规则

如果功能研发还没有完成，但需要保存当前阶段代码、跨设备继续开发或临时同步分支，使用：

```text
wip: <阶段性变更描述>
```

`wip` 的描述不能只写“同步 xxx 进度”，应说明当前阶段实际完成了什么。

推荐写法：

```text
wip: 搭建用户登录接口基础结构
wip: 接入语音播放队列核心流程
wip: 完成权限重构初版实现
wip: 增加实时转写 WebSocket 草稿
wip: 调整模型服务启动脚本
wip: 补充知识库导入流程雏形
wip: 临时保存语音识别联调代码
```

不推荐写法：

```text
wip: 同步代码
wip: 同步开发进度
wip: 同步用户登录进度
wip: 临时提交
wip: 改了一些东西
```

规则：

1. `wip` 表示代码未完成，不代表功能可用。
2. `wip` 也要描述本次阶段性变更内容。
3. 不要把未完成的功能写成 `feat`。
4. 不要把未修完的问题写成 `fix`。
5. `wip` 不建议作为正式发布提交。
6. 合并到主分支前，优先整理、压缩或替换 `wip` 提交。

## 配置、依赖、构建和部署规则

配置文件变更使用：

```text
config: 更新 xxx 配置
```

示例：

```text
config: 更新 pyproject 配置
config: 调整日志配置
config: 更新环境变量示例文件
config: 调整模型服务参数配置
```

注意：

```text
不要提交包含真实密钥、token、密码的 .env 文件
如果需要提交环境变量模板，优先提交 .env.example
```

依赖变更使用：

```text
deps: 更新 xxx 依赖
```

示例：

```text
deps: 升级 fastapi 依赖版本
deps: 更新 Python 项目依赖锁文件
deps: 调整 Java 项目 Maven 依赖
```

构建变更使用：

```text
build: 优化 xxx 构建流程
```

示例：

```text
build: 优化 Dockerfile 构建流程
build: 调整镜像构建缓存策略
build: 更新前端打包配置
```

部署和运行配置变更使用：

```text
deploy: 调整 xxx 部署配置
```

示例：

```text
deploy: 调整 docker-compose 服务配置
deploy: 更新 nginx 反向代理配置
deploy: 调整 systemd 服务启动配置
deploy: 更新 Kubernetes 部署配置
```

## 初始化仓库规则

初始化 Git 仓库时，默认创建 `main` 分支：

```bash
git init -b main
```

如果当前 Git 版本不支持：

```bash
git init
git branch -M main
```

如果仓库已经是 `master` 分支，应改为：

```bash
git branch -M main
```

不要默认创建或推送 `master` 分支。

## .gitattributes 规则

初始化仓库时，检查项目根目录是否存在 `.gitattributes`。

如果不存在，使用默认模板创建：

```text
resources/default.gitattributes
```

规则：

1. 不覆盖已有 `.gitattributes`。
2. 不自动重写用户已有配置。
3. `.gitattributes` 应纳入首次提交。
4. 如果本次只新增 `.gitattributes`，提交信息使用：

```text
chore: 初始化 Git 属性配置
```

## .gitignore 规则

初始化仓库时，检查项目根目录是否存在 `.gitignore`。

如果不存在，根据项目类型选择默认模板。

Python 项目使用：

```text
resources/python.gitignore
```

Java 项目使用：

```text
resources/java.gitignore
```

项目类型判断：

```text
存在 pyproject.toml、requirements.txt、setup.py、Pipfile、poetry.lock、uv.lock、*.py -> Python 项目
存在 pom.xml、build.gradle、build.gradle.kts、settings.gradle、gradlew、src/main/java -> Java 项目
```

规则：

1. 不覆盖已有 `.gitignore`。
2. 不自动重写用户已有配置。
3. `.gitignore` 应纳入首次提交。
4. 如果无法判断项目类型，不要强行创建，先提示用户确认。
5. 如果同时明显存在 Python 和 Java 项目，先提示用户选择模板。
6. 如果本次只新增 `.gitignore`，提交信息使用：

```text
chore: 初始化忽略规则
```

## 推荐 Skill 资源结构

```text
skills/git-commit-standard/
├── SKILL.md
└── resources/
    ├── default.gitattributes
    ├── python.gitignore
    └── java.gitignore
```

## 关联远程仓库

关联远程仓库时，默认推送到 `main`：

```bash
git remote add origin <仓库地址>
git push -u origin main
```

不要默认推送到 `master`。

## 好的提交示例

```text
feat: 新增用户登录功能
fix: 修复登录状态失效问题
docs: 更新 Docker 部署说明
config: 调整模型服务参数配置
deps: 升级 Python 项目依赖版本
build: 优化 Dockerfile 构建流程
deploy: 调整 docker-compose 服务配置
refactor: 重构权限校验逻辑
perf: 优化知识库检索速度
test: 增加登录接口测试
wip: 增加实时语音识别接口草稿
chore: 更新 gitignore 规则
release: 发布 v1.0.0
```

## 不好的提交示例

```text
update code
fix bug
修改一下
提交代码
临时改动
feat：新增功能
fix：修复bug
feat(auth): 新增登录功能
fix(user): 修复用户问题
wip: 同步代码
wip: 同步开发进度
```

## 提交粒度

一次提交尽量只做一件事。

推荐：

```text
feat: 新增用户登录功能
test: 增加登录接口测试
docs: 更新登录接口文档
```

不推荐：

```text
feat: 新增登录功能并修复订单问题和更新文档
```

如果一次变更包含多个无关内容，应建议拆成多个 commit。

## 提交前检查

生成提交信息或提交前，优先查看：

```bash
git status --short
git diff
git diff --cached
```

重点检查：

1. 是否有无关文件。
2. 是否有临时文件。
3. 是否有日志文件。
4. 是否有 `.env`。
5. 是否有密钥、token、密码。
6. 是否有缓存目录或大文件。
7. 是否有调试代码。

不要轻易提交：

```text
.env
.env.local
*.log
*.tmp
*.bak
*.key
*.pem
id_rsa
id_ed25519
__pycache__/
node_modules/
dist/
build/
.cache/
```

## 暂存规则

不要默认使用：

```bash
git add .
```

优先使用：

```bash
git add <file>
git add <dir>
git add -p
```

如果用户明确要求提交全部变更，也要先检查是否存在敏感文件或无关文件。

## 生成提交信息规则

当用户要求生成 commit message 时：

1. 根据 diff 判断变更内容。
2. 选择合适的 type。
3. 生成简短、具体的中文描述。
4. 不添加模块括号。
5. 不编造 diff 中不存在的内容。
6. 如果代码明显未完成但用户需要保存或同步，优先使用 `wip`。
7. `wip` 也要描述实际阶段性变更，不要只写“同步进度”。

默认输出：

```text
推荐提交信息：

feat: 新增用户登录功能
```

如果用户要命令：

```bash
git commit -m "feat: 新增用户登录功能"
```

## 修正规则

如果用户给出：

```text
feat：新增登录
```

修正为：

```text
feat: 新增登录功能
```

如果用户给出：

```text
feat(auth): 新增登录功能
```

修正为：

```text
feat: 新增登录功能
```

如果用户给出：

```text
fix bug
```

应根据实际 diff 修正为具体描述，例如：

```text
fix: 修复登录失败问题
```

如果用户表示功能未完成但要保存代码，应使用：

```text
wip: <阶段性变更描述>
```

例如：

```text
wip: 搭建用户登录接口基础结构
wip: 增加实时转写 WebSocket 草稿
wip: 临时保存语音识别联调代码
```

如果无法判断具体内容，不要编造，应提示需要查看 diff。

## 禁止默认执行的命令

除非用户明确要求，不要执行：

```bash
git push
git push --force
git reset --hard
git clean -fd
git clean -fdx
git rebase
git commit --amend
git tag
```

这些命令可能影响远程代码或导致本地文件丢失。

## 最终原则

提交信息要短，但不能糊。

优先使用：

```text
feat: 新增 xxx
fix: 修复 xxx
docs: 更新 xxx
config: 更新 xxx 配置
deps: 更新 xxx 依赖
build: 优化 xxx 构建流程
deploy: 调整 xxx 部署配置
refactor: 重构 xxx
perf: 优化 xxx
test: 增加 xxx 测试
wip: 完成 xxx 阶段性实现
chore: 清理 xxx
```

不要使用：

```text
feat(xxx): xxx
fix(xxx): xxx
update code
fix bug
提交代码
feat：新增功能
wip: 同步代码
wip: 同步开发进度
```

初始化仓库时：

```text
默认分支使用 main
缺少 .gitattributes 时使用 resources/default.gitattributes
缺少 .gitignore 时按项目类型使用 resources/python.gitignore 或 resources/java.gitignore
```
