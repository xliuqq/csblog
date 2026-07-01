# 包管理 

## UV

rust 编写的新一代 Python 包管理器，使用pyproject.toml和lock文件管理依赖：

- 创建标准兼容的Python虚拟环境
- 与`venv`模块兼容，但速度更快
- 环境激活方式与标准Python虚拟环境相同

## Conda

开源的**包管理和环境管理系统**，支持 Python、R 、NodJs 等

- 创建独立的环境，包含完整的Python解释器副本
- 环境之间完全隔离，包括**系统库**
- 需要特定的conda命令来激活环境

## Pip

### 源修改

pip install jpype1 -i https://pypi.tuna.tsinghua.edu.cn/simple

### pip install 默认安装路径修改

```
python -m site

USER_BASE python.exe启动程序路径
USER_SITE 依赖安装包基础路径

# 查看对应配置文件路径
python -m site -help

```
