## WSL 上安装 Aztec 测试网节点和rpc的步骤：

### 0. 打开你的 WSL 终端

确保你使用的是 WSL2，因为它对 Docker 的支持更好，性能也更优。

### 1. 安装依赖

这些命令与指南中一致，在你的 WSL 终端中运行：

```bash
sudo apt update
sudo apt install git curl -y
```

### 2. 安装 Docker 和 Docker Compose 插件

- 如果没有安装过桌面docker:
    
    ```bash
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh get-docker.sh
    ```
    
    删除安装脚本并将当前用户添加到 `docker` 组，以便无需 `sudo` 即可使用 Docker：
    
    ```bash
    sudo rm -r get-docker.sh
    sudo usermod -aG docker $USER
    ```
    
- 如果安装过桌面docker，并且启用了 WSL 集成功能与你当前的 Ubuntu WSL 环境对接，可跳过安装docker的步骤
    
    直接将当前用户添加到 `docker` 组，以便无需 `sudo` 即可使用 Docker：
    
    ```bash
    sudo usermod -aG docker $USER
    ```
    
    **重要**：为了使组更改生效，你需要**关闭所有 WSL 终端窗口，然后重新打开一个新的 WSL 终端**，或者重启你的 WSL 实例 (`wsl --shutdown` 在 PowerShell/CMD 中执行，然后重新打开 WSL)。你可以通过运行 `groups` 命令检查 `$USER` 是否在 `docker` 组中。
    

### 3. 设置 Ethereum 节点 - Sepolia 网络

如果你还没有自己的以太坊全节点，可以按照指南中的说明使用 `eth-docker` 在 WSL 内部署一个。

- **下载 eth-docker**:
    
    ```bash
    cd ~ 
    git clone https://github.com/eth-educators/eth-docker.git 
    cd eth-docker
    ```
    
- **配置**:
按照指南选择：`Sepolia Network > Ethereum RPC node >` (选择 Geth + Nimbus 或 Geth + Lodestar 作为执行层和共识层客户端组合)。
    
    ```bash
    ./ethd config
    ```
    
- **启动 Eth 节点**:
    
    ```bash
    ./ethd up -d
    ```
    

- **暴露端口**:
指南中提到修改 `geth.yml` 和 `nimbus-cl-only.yml` (或你选择的共识客户端的 yml 文件) 来暴露端口 `8545` (执行层 RPC) 和 `5052` (共识层 REST API，通常用于 beacon API)。
例如，在 `geth.yml` 中，找到 `ports` 部分，确保它看起来像这样 (如果不存在就添加 `ports` 部分)：
    
    ```bash
    # 在 geth.yml 文件中 (或者类似 services.geth.ports 下)
    ports:
      - "8545:8545" # RPC
      - "30303:30303/tcp" # p2p
      - "30303:30303/udp" # p2p`
    
    # 在 nimbus-cl-only.yml 文件中 (或者类似 services.nimbus.ports 下)
    ports:
      - "5052:5052" # Beacon API
      - "9000:9000/tcp" # p2p
      - "9000:9000/udp" # p2p`
    ```
    
- 对于共识层客户端 (如 Nimbus)，在对应的 `.yml` 文件 (例如 `nimbus-cl-only.yml`) 中，找到 `ports` 部分，确保包含：
    
    ```bash
    # 在 geth.yml 文件中 (或者类似 services.geth.ports 下)
    ports:
      - "8545:8545" # RPC
      - "30303:30303/tcp" # p2p
      - "30303:30303/udp" # p2p`
    
    # 在 nimbus-cl-only.yml 文件中 (或者类似 services.nimbus.ports 下)
    ports:
      - "5052:5052" # Beacon API
      - "9000:9000/tcp" # p2p
      - "9000:9000/udp" # p2p`
    ```
    
    修改后，你可能需要重启 `eth-docker` 服务：`./ethd down && ./ethd up -d`。
    **注意**：在 WSL2 中，`localhost` 或 `127.0.0.1` 通常可以直接从 WSL 内部访问。如果 Aztec 容器与以太坊节点容器在同一个 Docker 网络中（`eth-docker` 默认会创建自己的网络），它们可以通过容器名互相访问。但如果 Aztec 节点作为独立 Docker Compose 部署（如下一步），并且以太坊节点端口已映射到 WSL 的 `localhost`，那么 Aztec 节点可以使用 `http://localhost:8545` 访问。
    
- **等待同步**:
使用以下命令检查同步状态 (当返回 `false` 时表示已同步)：
    
    ```bash
    curl -X POST -H "Content-type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' http://localhost:8545
    ```
    

### 4. 设置 Aztec 工作文件夹

这些命令在你希望存放 Aztec 数据的目录中运行（例如用户主目录）：

```bash
cd ~ # 或者你选择的其他位置
mkdir -p aztec/data
sudo chmod 700 aztec/data # 指南用了 chmod 700，这意味着只有所有者有权读/写/执行
cd aztec
nano .env
```

将以下内容复制到 `.env` 文件中，并填充 `< >` 中的部分：

```bash
DATA_DIRECTORY=./data
COINBASE=<你的新以太坊0x地址>
LOG_LEVEL=debug
YOUR_RPC_ENDPOINT="http://localhost:8545" # 如果以太坊节点运行在同一 WSL 实例且端口已映射
YOUR_CONSENSUS_ENDPOINT="http://localhost:5052" # 如果共识节点运行在同一 WSL 实例且端口已映射
YOUR_VALIDATOR_PRIVATE_KEY=<你的新以太坊账户私钥字符串，不带0x前缀>
YOUR_IP_ADDRESS=<你的 Windows 主机的公网 IP 地址>
```

**关于 `YOUR_IP_ADDRESS`**: 这应该是你的**公网 IP 地址**。因为 P2P 节点需要从外部被发现。你需要确保你的路由器已将端口 `40400` (TCP/UDP) 转发到运行 WSL 的 Windows 机器的**内网 IP 地址**，并且 Windows 防火墙允许该端口的入站连接。

保存并关闭文件 (`Ctrl + O` 保存, `Ctrl + X` 退出 nano 编辑器)。

### 5. 设置 Docker Compose

在 `aztec` 文件夹内创建 `docker-compose.yml` 文件：

```bash
nano docker-compose.yml
```

粘贴以下内容：

```bash
version: '3.8'

name: aztec-node

services:
  node:
    image: aztecprotocol/aztec:0.87.2 # 注意：版本号可能会变，请检查官方文档或 Discord
    restart: unless-stopped
    environment:
      ETHEREUM_HOSTS: "${YOUR_RPC_ENDPOINT}"
      L1_CONSENSUS_HOST_URLS: "${YOUR_CONSENSUS_ENDPOINT}"
      DATA_DIRECTORY: /data
      VALIDATOR_PRIVATE_KEY: "${YOUR_VALIDATOR_PRIVATE_KEY}"
      P2P_IP: "${YOUR_IP_ADDRESS}"
      LOG_LEVEL: "${LOG_LEVEL:-debug}"
    entrypoint: >
      sh -c 'node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js start --network alpha-testnet start --node --archiver --sequencer'
    ports:
      - "40400:40400/tcp"
      - "40400:40400/udp"
      - "8080:8080" # Aztec 节点的 RPC 端口
    volumes:
      - ./data:/data
    network_mode: "host" # 使用主机网络模式简化 WSL 中的端口暴露
```

**关于 `network_mode: "host"`**:
在 WSL2 环境下，使用 `network_mode: "host"` 可以让容器直接使用 WSL 虚拟机的网络栈。这意味着容器内监听的端口会直接暴露在 WSL 虚拟机的 IP 地址上。对于从 WSL 内部访问这些端口（比如 Aztec 节点访问同一 WSL 实例中的以太坊节点，或者在 WSL 内部运行 `cast call`），这通常是最简单的方式。
**端口转发和防火墙**:

- 你需要将外部路由器的 `40400` (TCP/UDP) 端口转发到你的 Windows 主机的**内网 IP**。
- 你需要在 Windows 防火墙中为 `40400` (TCP/UDP) 创建入站规则，允许流量通过。
- 由于 `network_mode: "host"`，WSL2 会自动将 Windows 主机上监听的端口映射到 WSL2 虚拟机。

### 6. 运行节点

在你的 `aztec` 工作目录中启动节点：

```bash
docker compose up -d
```

检查日志：

```bash
docker compose logs -f node
```

### 7. 检查节点同步状态

- **检查 Rollup 合约以获取最新区块**:

如果你的以太坊节点也运行在 WSL 中并且 RPC 端口 (8545) 映射到了 `localhost`，那么 `http://localhost:8545` 应该能工作。
    
    ```bash
    docker run --rm --network host \
      aztecprotocol/foundry:25f24e677a6a32a62512ad4f561995589ac2c7dc-amd64 \
      cast call 0xee6d4e937f0493fb461f28a75cf591f1dba8704e \
                "getPendingBlockNumber()(uint256)" \
                --rpc-url http://localhost:8545 # 确保这里的 RPC URL 能从该 Docker 容器访问到你的以太坊节点
    ```
    
- **检查你自己的节点日志以获取最新区块**:
或者，如果你只想看最新的日志，可以直接用 `docker compose logs -f node` 然后观察是否有类似 "Cannot propose block X" (其中 X 是一个区块号) 的信息，这通常表示你的节点在尝试提议一个新区块。
    
    ```bash
    docker logs --tail 2000 aztec-node-node-1 2>&1 \
    | grep -Eo 'Cannot propose block [0-9]+' \
    | awk '{print $4}' \
    | tail -n1
    ```
    
    你的节点区块号应该比合约结果大1，这表示它是你的序列器正在尝试提出的区块。
