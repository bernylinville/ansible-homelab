# Dev 开发工具角色

这个 Ansible 角色用于在指定主机上安装开发工具，特别是 Charm 生态系统中的工具如 Crush 压缩工具。

## 功能特性

- ✅ **多节点控制**: 可配置在哪些主机上安装开发工具
- ✅ **安全安装**: 使用 GPG 密钥验证软件包来源
- ✅ **完整验证**: 安装后验证工具功能和配置完整性
- ✅ **自动清理**: 清理临时下载文件
- ✅ **可扩展**: 支持安装额外的开发工具包

## 支持的工具

### Crush AI 编码助手
- Charm 生态系统中的 AI 编码助手
- 在终端中提供智能编码支持
- 增强开发者的编程体验

## 使用方法

### 1. 在 Playbook 中使用

```yaml
- name: 安装开发工具
  hosts: all
  roles:
    - dev
```

### 2. 指定目标主机

```yaml
- name: 在特定主机上安装开发工具
  hosts: all
  vars:
    dev_target_hosts:
      - y9000p
      - dev-server
  roles:
    - dev
```

### 3. 安装额外工具

```yaml
- name: 安装开发工具和额外软件包
  hosts: all
  vars:
    dev_additional_packages:
      - git
      - vim
      - curl
      - jq
  roles:
    - dev
```

### 4. 禁用特定功能

```yaml
- name: 仅安装 Charm 仓库，不安装 Crush
  hosts: all
  vars:
    dev_crush_enabled: false
  roles:
    - dev
```

## 配置变量

### 主机控制
- `dev_target_hosts`: 目标主机列表 (默认: `["y9000p"]`)
- `dev_tools_enabled`: 是否启用开发工具安装 (默认: `true`)

### Crush 工具配置
- `dev_crush_enabled`: 是否安装 Crush (默认: `true`)
- `dev_crush_package`: Crush 软件包名 (默认: `"crush"`)

### 验证和清理
- `dev_verify_installation`: 是否验证安装 (默认: `true`)
- `dev_cleanup_temp_files`: 是否清理临时文件 (默认: `true`)

### 扩展配置
- `dev_additional_packages`: 额外安装的软件包列表 (默认: `[]`)

## 使用示例

### 基本使用
```bash
# 在 y9000p 主机上安装 Crush
ansible-playbook -i inventory/hosts.yml playbooks/dev-tools.yml
```

### 指定多个主机
```bash
# 在多个主机上安装
ansible-playbook -i inventory/hosts.yml playbooks/dev-tools.yml -e "dev_target_hosts=['y9000p','server2']"
```

### 仅安装特定工具
```bash
# 禁用 Crush，仅安装其他工具
ansible-playbook -i inventory/hosts.yml playbooks/dev-tools.yml -e "dev_crush_enabled=false"
```

## 安装后验证

角色执行完成后，可以通过以下命令验证安装：

```bash
# 检查 Crush 版本
crush --version

# 查看帮助信息
crush --help

# 启动 AI 编码助手
crush

# 在项目目录中使用
cd your-project
crush
```

## 文件位置

- **GPG 密钥**: `/etc/apt/keyrings/charm.gpg`
- **软件源配置**: `/etc/apt/sources.list.d/charm.list`
- **可执行文件**: `/usr/bin/crush`

## 故障排查

### 1. GPG 密钥下载失败
```bash
# 手动下载密钥
curl -fsSL https://repo.charm.sh/apt/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/charm.gpg
```

### 2. 软件源无法访问
```bash
# 检查网络连接
curl -I https://repo.charm.sh/apt/

# 检查软件源配置
cat /etc/apt/sources.list.d/charm.list
```

### 3. 安装失败
```bash
# 更新软件包缓存
sudo apt update

# 手动安装
sudo apt install crush
```

## 依赖要求

- **系统**: Ubuntu 20.04+ / Debian 10+
- **Ansible**: 2.12+
- **网络**: 需要访问 `repo.charm.sh`
- **权限**: 需要 sudo 权限安装软件包

## 许可证

MIT License