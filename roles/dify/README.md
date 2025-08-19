# Dify Ansible Role

基于 Ansible 的 Dify 部署角色，使用 `community.docker.docker_container` 进行容器管理。

## 功能特性

- ✅ **节点部署控制**: 精确控制每个服务部署在哪个节点
- ✅ **最小化生产配置**: 优化资源使用，适合个人部署
- ✅ **共享网络集成**: 使用 `docker_shared_network_name` 统一网络管理
- ✅ **安全配置**: 内置安全最佳实践和资源限制
- ✅ **中文界面**: 默认中文语言环境

## 部署的服务

- **dify-api**: Dify API 服务
- **dify-worker**: Celery 工作进程
- **dify-worker-beat**: Celery 定时任务
- **dify-web**: 前端界面
- **dify-db**: PostgreSQL 数据库
- **dify-redis**: Redis 缓存
- **dify-sandbox**: 代码执行沙箱
- **dify-plugin-daemon**: 插件守护进程
- **dify-ssrf-proxy**: SSRF 代理
- **dify-nginx**: 反向代理

## 配置说明

### 全局配置 (inventory/group_vars/all/all.yml)
```yaml
# Dify 版本控制
dify_version: "1.7.1"
dify_sandbox_version: "0.2.12"
dify_plugin_daemon_version: "0.2.0-local"

# 对外端口配置
dify_web_port: 18473  # HTTP 访问端口
dify_plugin_debug_port: 18474  # 调试端口（仅开发环境）
```

### 安全配置 (inventory/group_vars/all/vault.yml - 已加密)
所有敏感信息都存储在加密的 vault.yml 中：
- 数据库密码
- Redis 密码 
- API 密钥
- 初始管理员密码

## 使用方法

### 1. 部署 Dify
```bash
ansible-playbook homelab.yml --tags dify --vault-password-file .vault_pass
```

### 2. 仅部署在特定节点
```bash
ansible-playbook homelab.yml --tags dify --limit y9000p --vault-password-file .vault_pass
```

### 3. 自定义节点部署
在 `inventory/group_vars/all/all.yml` 中配置：
```yaml
dify_deployment_nodes:
  api: "y9000p"
  worker: "y9000p"
  web: "server2"
  db: "server3"
  # ... 其他服务
```

### 4. 访问 Dify
部署完成后访问：`http://your-server:18473`

## 安全配置

### Vault 管理
所有敏感信息已配置在加密的 `vault.yml` 中：

```bash
# 编辑加密的 vault 文件
ansible-vault edit inventory/group_vars/all/vault.yml --vault-password-file .vault_pass

# 查看加密文件内容
ansible-vault view inventory/group_vars/all/vault.yml --vault-password-file .vault_pass

# 重新加密文件
ansible-vault rekey inventory/group_vars/all/vault.yml --vault-password-file .vault_pass
```

### 默认登录信息
- **初始管理员密码**: 存储在 `vault_dify_init_password` 中
- **访问地址**: `http://your-server:18473`

## 资源配置

默认资源限制适合个人部署，可在 `dify_resource_limits` 中调整：

```yaml
dify_resource_limits:
  api:
    memory: "512m"
    cpus: "0.5"
  db:
    memory: "1g" 
    cpus: "1.0"
  # ... 其他服务
```

## 网络配置

- **对外端口**: 18473 (HTTP), 18474 (HTTPS)
- **内部服务**: Redis、PostgreSQL 等内部服务不对外暴露端口
- **内部网络**: 使用 `docker_shared_network_name` 和 `dify_internal` 网络
- **安全隔离**: SSRF Proxy 提供内部网络隔离
- **调试端口**: 仅在开发环境 (DEBUG=true) 暴露插件调试端口

## 故障排除

### 检查服务状态
```bash
docker ps | grep dify
```

### 查看日志
```bash
docker logs dify-api
docker logs dify-web
```

### 重新部署服务
```bash
ansible-playbook homelab.yml --tags dify --limit y9000p
```