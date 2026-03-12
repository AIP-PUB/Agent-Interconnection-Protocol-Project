[首页](../Tutorials.md)

# RabbitMQ安装教程

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
