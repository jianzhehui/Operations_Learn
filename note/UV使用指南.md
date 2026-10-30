# UV使用指南

## 1、管理Python版本

```python
uv python list
#查看可用的Python版本
```

安装特定版本的 Python：

```python
uv python install 3.12

uv python install 3.11.6

uv python pin 3.12
#将项目固定为3.12
```

## 2、管理虚拟环境

创建虚拟环境

```python
uv venv

uv venv --python 3.12
#使用指定 Python 版本创建虚拟环境
```

激活虚拟环境

```python
.venv\Scripts\activate
```

## 3、项目管理 

初始化项目

```python
uv init
```

添加和移除依赖

```python
uv add requests

uv remove requests
```

安装项目所有依赖

```python
uv sync
```



