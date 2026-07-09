---
name: python-path-standard
description: 规范 Python 项目中的文件路径写法，统一使用基于项目根目录的跨系统路径封装，避免 Windows 和 Linux 路径不兼容问题。
---

# Python Path Standard Skill

## 目标

当 Python 项目中涉及文件路径、目录路径、模型路径、配置文件路径、上传文件路径、日志路径、数据集路径、缓存路径时，使用本 Skill。

目标：

1. 避免硬编码绝对路径。
2. 避免直接拼接 Windows 或 Linux 风格路径。
3. 统一从项目根目录生成绝对路径。
4. 保证 Windows、Mac、Linux 下路径稳定。
5. 优先使用封装函数处理项目内文件路径。
6. 涉及路径改动时，自动补充简单测试。

## 核心规则

Python 项目中不要直接写这种路径：

```python
path = "data/test.json"
path = "E:\\project\\data\\test.json"
path = "/home/user/project/data/test.json"
path = os.getcwd() + "/data/test.json"
path = BASE_DIR + "/data/test.json"
```

应该统一使用：

```python
path = get_os_base_join("data/test.json")
```

或：

```python
path = get_pathlib_base_join("data/test.json")
```

## 路径工具函数

项目中应封装统一的路径工具函数，例如放在：

```text
utils/path_utils.py
```

或项目已有的公共工具模块中。

推荐代码：

```python
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent.parent


def get_os_base_join(rel_path: str | None = None):
    """
    基于项目相对路径获取绝对路径，支持 Windows / Mac / Linux

    :param rel_path: 相对路径，为空时返回项目根目录。
    :return: 规范化后的字符串路径。
    """
    if rel_path is None:
        return os.path.normpath(BASE_DIR)
    return os.path.normpath(os.path.join(BASE_DIR, rel_path))


def get_pathlib_base_join(rel_path: str | None = None) -> Path:
    """
    基于项目相对路径获取绝对路径，支持 Windows / Mac / Linux

    :param rel_path: 相对路径，为空时返回项目根目录。
    :return: Path 对象。
    """
    if rel_path is None:
        return BASE_DIR
    return (BASE_DIR / rel_path).resolve()
```

## BASE_DIR 规则

`BASE_DIR` 必须根据当前 `utils.py` 或 `path_utils.py` 的实际位置决定。

示例：

如果文件在项目根目录：

```text
project/
  utils.py
```

使用：

```python
BASE_DIR = Path(__file__).resolve().parent
```

如果文件在一层目录下：

```text
project/
  utils/
    path_utils.py
```

使用：

```python
BASE_DIR = Path(__file__).resolve().parent.parent
```

如果文件在两层目录下：

```text
project/
  app/
    common/
      path_utils.py
```

使用：

```python
BASE_DIR = Path(__file__).resolve().parent.parent.parent
```

判断原则：

```text
BASE_DIR 必须指向项目根目录。
```

不要照抄 parent 层级，应根据工具文件实际位置调整。

## 使用规则

项目内路径统一传入相对于项目根目录的路径。

推荐：

```python
config_path = get_os_base_join("config/settings.yaml")
data_path = get_os_base_join("data/input.json")
model_path = get_os_base_join("models/qwen")
log_path = get_os_base_join("logs/app.log")
```

pathlib 版本推荐用于文件读写：

```python
config_path = get_pathlib_base_join("config/settings.yaml")
content = config_path.read_text(encoding="utf-8")
```

创建目录时推荐：

```python
log_dir = get_pathlib_base_join("logs")
log_dir.mkdir(parents=True, exist_ok=True)
```

写文件时推荐：

```python
output_path = get_pathlib_base_join("outputs/result.json")
output_path.parent.mkdir(parents=True, exist_ok=True)
output_path.write_text("{}", encoding="utf-8")
```

## 选择规则

需要字符串路径时，使用：

```python
get_os_base_join("xxx")
```

常见场景：

```text
第三方库只接受 str 路径
老代码使用 os.path
配置项要求字符串
```

需要 Path 对象时，使用：

```python
get_pathlib_base_join("xxx")
```

常见场景：

```text
读写文件
创建目录
判断文件是否存在
路径继续拼接
```

## 禁止写法

不要使用硬编码绝对路径：

```python
path = "/home/user/project/data/input.json"
path = "C:\\Users\\user\\project\\data\\input.json"
```

不要手写系统分隔符：

```python
path = BASE_DIR + "/data/input.json"
path = BASE_DIR + "\\data\\input.json"
```

不要依赖当前工作目录：

```python
path = os.getcwd() + "/data/input.json"
open("data/input.json")
```

不要在多个文件中重复定义不同的 `BASE_DIR`。

项目中应该只有一个统一的路径工具入口。

## 自动改造规则

当发现 Python 代码中存在文件路径问题时，应优先改造为路径工具函数。

原始写法：

```python
with open("data/input.json", "r", encoding="utf-8") as f:
    data = f.read()
```

推荐改造：

```python
from utils.path_utils import get_pathlib_base_join

input_path = get_pathlib_base_join("data/input.json")

with input_path.open("r", encoding="utf-8") as f:
    data = f.read()
```

原始写法：

```python
model_path = "./models/qwen"
```

推荐改造：

```python
from utils.path_utils import get_os_base_join

model_path = get_os_base_join("models/qwen")
```

原始写法：

```python
log_path = os.path.join(os.getcwd(), "logs", "app.log")
```

推荐改造：

```python
from utils.path_utils import get_pathlib_base_join

log_path = get_pathlib_base_join("logs/app.log")
log_path.parent.mkdir(parents=True, exist_ok=True)
```

## 测试规则

涉及路径工具函数新增或修改时，应补充简单测试。

推荐测试文件：

```text
tests/test_path_utils.py
```

测试示例：

```python
from pathlib import Path

from utils.path_utils import BASE_DIR, get_os_base_join, get_pathlib_base_join


def test_get_os_base_join_return_base_dir():
    path = get_os_base_join()
    assert isinstance(path, str)
    assert Path(path).exists()


def test_get_pathlib_base_join_return_base_dir():
    path = get_pathlib_base_join()
    assert isinstance(path, Path)
    assert path.exists()


def test_get_os_base_join_with_relative_path():
    path = get_os_base_join("data/test.json")
    assert isinstance(path, str)
    assert str(BASE_DIR) in path


def test_get_pathlib_base_join_with_relative_path():
    path = get_pathlib_base_join("data/test.json")
    assert isinstance(path, Path)
    assert path.is_absolute()
    assert BASE_DIR in path.parents or path == BASE_DIR
```

如果项目没有 pytest，也至少提供一个简单自检脚本：

```python
from utils.path_utils import BASE_DIR, get_os_base_join, get_pathlib_base_join

print("BASE_DIR:", BASE_DIR)
print("str path:", get_os_base_join("data/test.json"))
print("pathlib path:", get_pathlib_base_join("data/test.json"))
```

## 检查清单

处理 Python 文件路径时，应检查：

1. 是否有硬编码绝对路径。
2. 是否有 `./xxx`、`../xxx` 这类依赖运行目录的路径。
3. 是否有手写 `/` 或 `\\` 拼接路径。
4. 是否有多个重复的 `BASE_DIR`。
5. 是否能在 Windows 和 Linux 下正常运行。
6. 是否需要创建父目录。
7. 是否需要返回字符串路径还是 Path 对象。
8. 是否补充了路径函数测试。

## 最终原则

Python 项目中涉及项目内文件路径时，优先使用：

```python
get_os_base_join("相对项目根目录的路径")
```

或：

```python
get_pathlib_base_join("相对项目根目录的路径")
```

不要依赖当前工作目录。

不要硬编码操作系统路径。

不要手写路径分隔符。

`BASE_DIR` 必须根据路径工具文件所在位置正确指向项目根目录。

涉及路径工具改动时，应自动补充测试。
