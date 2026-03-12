[首页](../README.md)

# 1.ACPs协议细节
所有ACPs协议文档均可通过 [协议文档汇总](../README.md) 访问。

# 2.服务端搭建

如果您想要搭建您自己的智能体注册服务器，请参考：[ ACPs Registry Server](https://github.com/AIP-PUB/ACPs-Registry-Server)

如果您想要搭建您自己的智能体发现服务器，请参考：[ACPs Discovery Server](https://github.com/AIP-PUB/ACPs-Discovery-Server)

如果您想要搭建您自己的CA服务器，请参考：[ACPs CA Server](https://github.com/AIP-PUB/ACPs-CA-Server)

这三个服务的关系如下：

1. `Registry Server` 是基础服务，负责账户、智能体的可信注册，提供可信注册相关接口和数据同步相关接口。
2. `Discovery Server` 依赖 `Registry Server` 的数据同步相关接口同步 ACS 数据，再对外提供发现能力。
3. `CA Server` 依赖 `Registry Server` 的可信注册相关接口，负责证书签发、吊销和状态查询。

三个服务都需要用到数据库服务，您需要根据自身情况提前启动数据库，本教程以PostgreSQL为例。

## 2.1 Registry Server 搭建

项目路径：[ACPs Registry Server](https://github.com/AIP-PUB/ACPs-Registry-Server)

### 2.1.1. 服务职责

Registry Server 是 ACPs 服务端体系的基础组件，至少承担以下职责：

- 提供账户与认证接口
- 提供智能体注册与管理接口
- 提供 ATR 接口
- 提供 DRC 数据同步接口，供 Discovery Server 拉取或接收 webhook

其主入口位于 `Registry Server/main.py`，当前对外主要暴露：

- 普通 API 前缀：`/api`
- ATR 前缀：`/acps-atr-v2`
- DRC 前缀：`/acps-drc-v2`
- 文档入口：`/docs`

### 2.1.2. 创建虚拟环境并安装依赖

```bash
cd registry-server
python3.13 -m venv venv     # 创建python虚拟环境，python的路径和名字根据实际情况调整
source venv/bin/activate    # 激活虚拟环境
pip install poetry      # 如果尚未全局安装 Poetry，可以在虚拟环境中安装
poetry install          # 安装依赖
```

### 2.1.3. 准备环境变量

```bash
cp .env.example .env
# 编辑 .env 文件，修改必要的配置项
```

### 2.1.4. 初始化数据库

启动服务前，先进行数据库迁移，这样可以保证结构和初始化数据完整。

```bash
alembic upgrade head
```


### 2.1.5. 启动服务

后台启动：

```bash
./run.sh # 会直接使用 ./venv/bin/python（或 .venv/bin/python）启动服务，因此执行前需要先完成上面的虚拟环境创建和依赖安装。
```

前台调试：

```bash
source venv/bin/activate
python main.py
```

### 2.1.6. 启动后验证

假设端口使用默认值 `8001`，可验证以下地址：

- 根路径：`http://localhost:8001/`
- OpenAPI 文档：`http://localhost:8001/docs`
- DRC 信息接口：`http://localhost:8001/acps-drc-v2/info`


## 2.2 Discovery Server 搭建

项目路径：[Discovery Server](https://github.com/AIP-PUB/ACPs-Discovery-Server)

### 2.2.1. 服务职责

Discovery Server 可以解析自然语言的请求，返回以Agent的skill为主体的rank列表。它本身不直接维护 ACS 主数据，而是通过 DRC 协议与 Registry Server 同步。

其主入口位于 `main.py`，当前对外主要暴露：

- 发现接口前缀：`/api/discovery`
- DRC 管理接口前缀：`/admin/drc`
- 文档入口：`/docs`

启动时会自动执行以下动作：

1. 创建数据库表
2. 启动 DRC 同步任务
3. 启动 GPU/模型健康检查任务

### 2.2.2. 创建虚拟环境并安装依赖

```bash
cd discovery-server-public
python3.13 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 2.2.3. 准备环境变量

```bash
cp .env.example .env
```

需要重点检查以下配置：

- `DATABASE_URL`
  例如 `postgresql+asyncpg://user:password@localhost:5432/agent_discovery`
- `DRC_BASE_URL`
  必须指向 Registry Server 的 DRC 根路径，例如`http://localhost:8001/acps-drc-v2`
- `DRC_WEBHOOK_RECEIVE_URL`
  必须是 Discovery Server 自身可被 Registry 回调的地址
- `OPENAI_API_KEY`
  用于 embedding
- `OPENAI_BASE_URL`
  如果不是调用官方 OpenAI 端点，这里需要填写实际端点
- `EMBEDDING_MODEL_NAME`
  默认是 `text-embedding-3-small`
- `DASHSCOPE_API_KEY`
  如果发现逻辑使用该能力，则也需要配置
- `GPU_SERVER_ENABLED`、`GPU_SERVER_URL`
  仅在启用 GPU 转发时需要

### 2.2.4. 初始化数据库

```bash
alembic upgrade head
```

### 2.2.5. 启动服务

后台启动：

```bash
./start.sh
```

前台调试：

```bash
python main.py
```

### 2.2.6. 启动后验证

假设端口使用默认值 `8005`，可验证以下地址：

- 根路径：`http://localhost:8005/`
- OpenAPI 文档：`http://localhost:8005/docs`


## 2.3 CA Server 搭建

项目路径：[`CA Server`]()

### 2.3.1. 服务职责

CA Server 提供证书相关能力。

其主入口位于 `main.py`，当前对外主要暴露：

- ACME 前缀：`/acps-atr-v2/acme`
- CA 前缀：`/acps-atr-v2/ca`
- CRL 前缀：`/acps-atr-v2/crl`
- OCSP 前缀：`/acps-atr-v2/ocsp`
- 管理接口：`/admin/certificates`
- 健康检查：`/health`
- 文档入口：`/docs`

### 2.3.2. 准备环境变量与证书文件

创建 `.env` 文件并配置以下环境变量：

```bash
# 复制环境变量模板
cp .env.example .env
```

在 `.env` 文件中根据实际情况修改各个配置项。

```bash
cp .env.example .env
```

需要检查以下配置：

- `DATABASE_URL`
  例如 `postgresql://postgres@localhost:5432/agent_ca`
- `HOST`、`PORT`
  默认端口是 `8003`
- `CA_CERT_PATH`
  必须指向实际存在的 CA 证书文件
- `CA_KEY_PATH`
  必须指向实际存在的 CA 私钥文件
- `AGENT_REGISTRY_URL`
  必须指向 Registry 的 ATR 根路径，默认示例为 `http://localhost:8001/acps-atr-v2`
- `AGENT_REGISTRY_MOCK`
  若不联调真实 Registry，可临时启用 mock；正式联调应保持 `false`
- `HTTP01_VALIDATION_MOCK`
  仅在开发阶段可按需开启

如果 `CA_CERT_PATH` 或 `CA_KEY_PATH` 指向的文件不存在，服务即使启动，也无法完成真实证书签发相关流程。

### 2.3.3. 创建虚拟环境

```bash
python3 -m venv venv # 创建虚拟环境（推荐使用 Python 3.13+）
source venv/bin/activate # 激活虚拟环境
pip install poetry # 安装 Poetry（如果未全局安装）
poetry install # 安装项目依赖
```

### 2.3.4. 初始化数据库

```bash
alembic upgrade head
```

### 2.3.5. 启动服务器

启动开发服务器

```bash
# 使用 uvicorn 直接启动（reload推荐开发时使用）
uvicorn main:app --reload
```

后台运行

可以使用 `./run.sh` 脚本来管理后台服务：

### 2.3.6. 验证安装

启动服务器后，访问以下 URL 验证安装：

- **API 文档**: http://localhost:8003/docs
- **ReDoc 文档**: http://localhost:8003/redoc
- **健康检查**: http://localhost:8003/health (如果实现了健康检查端点)


## 2.4 消息队列服务的部署

以下是RabbitMQ的安装与配置教程

```bash
# 更新 Ubuntu 的包列表，确保依赖库最新
sudo apt update && sudo apt upgrade -y
# 安装 erlang
sudo apt-get install erlang-nox
# （可选）如果安装时遇到依赖冲突，可尝试运行下面的命令修复依赖，再重新安装erlang
sudo apt --fix-broken install
 # 检查安装是否成功，输出类似 "Erlang (SMP,ASYNC_THREADS) (BEAM) emulator version X.Y.Z" 即成功
erl -version 
# 安装rbmq
sudo apt-get install rabbitmq-server
# 启动服务
sudo systemctl start rabbitmq-server
# 设置开机自启
sudo systemctl enable rabbitmq-server   
# 检查服务状态
sudo systemctl status rabbitmq-server   
```

