# 包管理 

## Pip

### 源修改

pip install jpype1 -i https://pypi.tuna.tsinghua.edu.cn/simple

https://pypi.douban.com/simple/ 

### pip install 默认安装路径修改

```
python -m site

USER_BASE python.exe启动程序路径
USER_SITE 依赖安装包基础路径

# 查看对应配置文件路径
python -m site -help

```

##  Poetry

Python 包管理工具

### 环境管理

默认使用 virtualenv 创建虚拟环境。

- 关闭自动创建 env

```shell
poetry config virtualenvs.create false
```

缓存默认位置`cache-dir`

```
macOS: ~/Library/Caches/pypoetry
Windows: C:\Users\<username>\AppData\Local\pypoetry\Cache
Unix: ~/.cache/pypoetry
```

### 使用

```bash
poetry new poetry-demo
```

目录结构组织如下：

```text
poetry-demo
├── pyproject.toml
├── README.md
├── poetry_demo
│   └── __init__.py
└── tests
    └── __init__.py
```

配置说明文件

```toml
[tool.poetry]
name = "poetry-demo"
version = "0.1.0"
description = ""
authors = ["Sébastien Eustace <sebastien@eustace.io>"]
readme = "README.md"
packages = [{include = "poetry_demo"}]

[tool.poetry.dependencies]
python = "^3.7"


[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

安装依赖

```shell
poetry install
```

