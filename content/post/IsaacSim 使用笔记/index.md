---
date: 2025-10-17
tags:
- 科研
- IsaacSim
publish: true
title: IsaacSim 使用笔记
---

## 容器安装
为了使得容器中运行的应用程序可以使用 GPU，必须安装英伟达容器工具包：
```bash
# Configure the repository
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
    && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list \
    && \
    sudo apt-get update

# Install the NVIDIA Container Toolkit packages
sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker

# Configure the container runtime
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Verify NVIDIA Container Toolkit
docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
```
### 容器安装 IsaacSim
下载镜像：
```bash
docker pull nvcr.io/nvidia/isaac-sim:5.1.0
```
创建挂载卷：
```bash
mkdir -p ~/docker/isaac-sim/cache/main/ov
mkdir -p ~/docker/isaac-sim/cache/main/warp
mkdir -p ~/docker/isaac-sim/cache/computecache
mkdir -p ~/docker/isaac-sim/config
mkdir -p ~/docker/isaac-sim/data/documents
mkdir -p ~/docker/isaac-sim/data/Kit
mkdir -p ~/docker/isaac-sim/logs
mkdir -p ~/docker/isaac-sim/pkg
sudo chown -R 1234:1234 ~/docker/isaac-sim
```
### 容器安装 IsaacLab
```bash
git clone git@github.com:isaac-sim/IsaacLab.git
cd IsaacLab && ./docker/container.py start
```
## 容器使用
### IsaacSim
运行容器：
```bash
docker run --name isaac-sim --entrypoint bash -it --gpus all -e "ACCEPT_EULA=Y" --rm --network=host \
    -e "PRIVACY_CONSENT=Y" \
    -v ~/docker/isaac-sim/cache/main:/isaac-sim/.cache:rw \
    -v ~/docker/isaac-sim/cache/computecache:/isaac-sim/.nv/ComputeCache:rw \
    -v ~/docker/isaac-sim/logs:/isaac-sim/.nvidia-omniverse/logs:rw \
    -v ~/docker/isaac-sim/config:/isaac-sim/.nvidia-omniverse/config:rw \
    -v ~/docker/isaac-sim/data:/isaac-sim/.local/share/ov/data:rw \
    -v ~/docker/isaac-sim/pkg:/isaac-sim/.local/share/ov/pkg:rw \
    -u 1234:1234 \
    nvcr.io/nvidia/isaac-sim:5.1.0
```
打开：
```bash
./runapp.sh
```
### IsaacLab
打开：
```bash
# Launch the container in detached mode
# We don't pass an image extension arg, so it defaults to 'base'
./docker/container.py start

# If we want to add .env or .yaml files to customize our compose config,
# we can simply specify them in the same manner as the compose cli
# ./docker/container.py start --file my-compose.yaml --env-file .env.my-vars

# Enter the container
# We pass 'base' explicitly, but if we hadn't it would default to 'base'
./docker/container.py enter base
```
## 在容器中使用 VSCode 编写代码
### 安装容器开发套件
打开 VSCode 之后，搜索 `ssh @pack`，选择 `Extensions Pack`，安装远程开发套件。注意，这会自动安装 Dev Container 扩展，这是在容器中编写代码的核心插件，如果你对其他插件感到困扰，可以只安装 Dev Container，尽管安装整个 pack 是笔者更推荐的做法。然后，搜索拓展包 `python @pack`，选择 `Extensions Pack`，安装 Python 开发套件。
### 启动容器
进入 IsaacLab 的容器安装目录，执行 `./docker/container.py start` 启动容器，再执行 `./docker/container.py enter base` 进入容器。
### 打开容器内的工作目录
在 VSCode 中按下 Ctrl+Shift+P，搜索 `Dev Containers: Open Folder in Container`，选择刚刚打开的容器。打开新工作区后，在目录 `/workspace/isaaclab` 下打开。
### 新建项目
执行 `./isaaclab.sh --new`，按照提示构建新项目，在新项目的目录下重新打开 VSCode，运行 VSCode Tasks ，通过按下 Ctrl+Shift+P ，选择 `Tasks: Run Task` 并在下拉菜单中运行 `setup_python_env`。如果一切执行正确，它应该创建以下文件:
- `.vscode/launch.json`: 包含用于调试 python 代码的启动配置。
- `.vscode/settings.json`: 包含 python 解释器和 python 环境的设置。
如果你不配置 `setup_python_env`，随便开启一个 Python 文件，输入 `from isaaclab.app`，你会发现没有任何智能补全。相信任何一个喜欢手写代码的老顽固，都不会想要自己手动输入所有的包名与类名。