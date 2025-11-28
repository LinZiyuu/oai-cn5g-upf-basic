# OAI CN5G UPF Basic

这是一个用于测试 OAI CN5G UPF 和 SMF 的 Docker Compose 项目。项目提供了两种部署模式：完整 5G 核心网部署和仅 UPF+SMF 的简化部署。

## 项目结构

```
oai-cn5g-upf-basic/
├── docker-compose.yaml              # 完整 5G 核心网部署配置
├── docker-compose-upf-only.yaml    # 仅 UPF+SMF 部署配置
├── conf/                            # 配置文件目录
│   ├── config.yaml                  # 完整部署的配置文件
│   ├── config-upf-only.yaml        # UPF-only 部署的配置文件
│   ├── sip.conf                     # IMS SIP 配置
│   └── users.conf                   # IMS 用户配置
├── database/                        # 数据库初始化脚本
│   └── oai_db.sql                   # OAI 数据库初始化 SQL
└── healthscripts/                   # 健康检查脚本
    └── mysql-healthcheck.sh         # MySQL 健康检查脚本
```

## 前置要求

- Docker Engine 20.10+
- Docker Compose 2.0+
- Linux 系统（推荐 Ubuntu 20.04+）
- 足够的系统资源（建议至少 4GB RAM）

## 快速开始

### 方式一：仅启动 UPF 和 SMF（推荐用于测试）

这种方式只启动 SMF 和 UPF 两个核心组件，适合快速测试 UPF 功能。

```bash
# 启动服务
docker-compose -f docker-compose-upf-only.yaml up -d

# 查看服务状态
docker-compose -f docker-compose-upf-only.yaml ps

# 查看日志
docker-compose -f docker-compose-upf-only.yaml logs -f oai-upf
docker-compose -f docker-compose-upf-only.yaml logs -f oai-smf

# 停止服务
docker-compose -f docker-compose-upf-only.yaml down
```

### 方式二：完整 5G 核心网部署

这种方式启动完整的 5G 核心网组件，包括：
- MySQL 数据库
- NRF (Network Repository Function)
- UDR (Unified Data Repository)
- UDM (Unified Data Management)
- AUSF (Authentication Server Function)
- AMF (Access and Mobility Management Function)
- SMF (Session Management Function)
- UPF (User Plane Function)
- IMS (IP Multimedia Subsystem)
- External DN (Data Network)

```bash
# 启动所有服务
docker-compose up -d

# 查看服务状态
docker-compose ps

# 查看特定服务日志
docker-compose logs -f oai-upf
docker-compose logs -f oai-smf
docker-compose logs -f oai-amf

# 停止所有服务
docker-compose down
```

## 网络配置

项目使用自定义的 Docker 网络 `oai-cn5g-public-net`，子网为 `192.168.70.128/26`。

### 服务 IP 地址分配（完整部署）

| 服务 | 容器名 | IP 地址 |
|------|--------|---------|
| MySQL | mysql | 192.168.70.131 |
| NRF | oai-nrf | 192.168.70.130 |
| AMF | oai-amf | 192.168.70.132 |
| SMF | oai-smf | 192.168.70.133 |
| UPF | oai-upf | 192.168.70.134 |
| External DN | oai-ext-dn | 192.168.70.135 |
| UDR | oai-udr | 192.168.70.136 |
| UDM | oai-udm | 192.168.70.137 |
| AUSF | oai-ausf | 192.168.70.138 |
| IMS | ims | 192.168.70.139 |

### UPF-only 部署 IP 地址

| 服务 | 容器名 | IP 地址 |
|------|--------|---------|
| SMF | oai-smf | 192.168.70.133 |
| UPF | oai-upf | 192.168.70.134 |

**注意**：在 UPF-only 模式下，UPF 容器会自动创建 `eth1` dummy 接口并配置 IP 地址 `192.168.70.240/26`，用于 N6 接口，避免 XDP 程序冲突。

## 端口说明

### SMF 端口
- `80/tcp`: HTTP 接口
- `8080/tcp`: SBI 接口
- `8805/udp`: PFCP 接口（与 UPF 通信）

### UPF 端口
- `2152/udp`: N3 接口（与 gNB 通信）
- `8805/udp`: PFCP 接口（与 SMF 通信）

## 配置文件说明

### config.yaml / config-upf-only.yaml

主要的配置文件，包含所有网络功能的配置。主要区别：
- `config.yaml`: `register_nf: yes`，启用 NRF 注册
- `config-upf-only.yaml`: `register_nf: no`，禁用 NRF 注册（因为不启动 NRF）

### 数据库配置

MySQL 数据库配置：
- 数据库名: `oai_db`
- 用户名: `test`
- 密码: `test`
- Root 密码: `linux`

## 常用操作

### 查看容器日志

```bash
# 查看所有服务日志
docker-compose logs -f

# 查看特定服务日志
docker-compose logs -f oai-upf
docker-compose logs -f oai-smf
```

### 进入容器调试

```bash
# 进入 UPF 容器
docker exec -it oai-upf bash

# 进入 SMF 容器
docker exec -it oai-smf bash
```

### 检查网络连接

```bash
# 检查 Docker 网络
docker network inspect oai-cn5g-public-net

# 在容器内测试网络连通性
docker exec oai-upf ping -c 3 192.168.70.133  # 测试 UPF 到 SMF
docker exec oai-smf ping -c 3 192.168.70.134  # 测试 SMF 到 UPF
```

### 重启服务

```bash
# 重启所有服务
docker-compose restart

# 重启特定服务
docker-compose restart oai-upf oai-smf
```

## 故障排查

### 1. 容器无法启动

检查 Docker 日志：
```bash
docker-compose logs <service-name>
```

### 2. 网络问题

确保 Docker 网络已创建：
```bash
docker network ls | grep oai-cn5g-public-net
```

如果网络不存在，重新创建：
```bash
docker-compose down
docker-compose up -d
```

### 3. UPF 权限问题

UPF 需要 `NET_ADMIN` 和 `SYS_ADMIN` 权限，以及 `privileged: true`。如果遇到权限问题，检查 docker-compose 配置。

### 4. 端口冲突

检查端口是否被占用：
```bash
# 检查 8805 端口
sudo netstat -tulpn | grep 8805

# 检查 2152 端口
sudo netstat -tulpn | grep 2152
```

## 测试建议

1. **启动顺序**：先启动 SMF，再启动 UPF（UPF 依赖 SMF）
2. **健康检查**：等待容器健康检查通过后再进行测试
3. **日志监控**：启动后持续监控日志，确保没有错误
4. **网络验证**：验证容器间网络连通性

## 注意事项

- UPF 容器需要特权模式运行（`privileged: true`），因为它需要创建网络接口和运行 XDP 程序
- 确保系统支持 XDP（eBPF），否则 UPF 可能无法正常工作
- 在生产环境中，请修改默认密码和配置
- 网络配置可能需要根据实际环境调整

## 相关资源

- [OAI 官方文档](https://openairinterface.org/)
- [OAI GitHub](https://github.com/OPENAIRINTERFACE)

## 许可证

本项目基于 OAI Public License, Version 1.1。

