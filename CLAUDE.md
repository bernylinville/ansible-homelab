# Ansible Homelab 项目约束

## 核心原则

**主要指令**: "中文优先 | 质量至上 | 安全第一 | 最佳实践"

## 语言约束

### 必须使用中文
- **文档说明**: 所有任务描述、注释、文档均使用中文
- **变量命名**: 使用英文但需要中文注释说明
- **提交信息**: Git commit 消息使用中文
- **错误处理**: 错误信息和日志使用中文描述

## Ansible 编码规范

### FQCN (完全限定集合名称) - 强制要求
```yaml
# ✅ 正确 - 使用 FQCN
- name: 安装软件包
  ansible.builtin.package:
    name: vim
    state: present

# ❌ 错误 - 缺少 FQCN
- name: 安装软件包
  package:
    name: vim
    state: present
```

### 布尔值规范 - 强制要求
```yaml
# ✅ 正确 - 使用 true/false
- name: 配置服务
  ansible.builtin.systemd:
    name: nginx
    enabled: true
    state: started
  become: true

# ❌ 错误 - 使用 yes/no
- name: 配置服务
  systemd:
    name: nginx
    enabled: yes
    state: started
  become: yes
```

### 文件权限规范 - 强制要求
```yaml
# ✅ 正确 - 指定文件权限
- name: 创建配置文件
  ansible.builtin.copy:
    dest: /etc/myapp/config.conf
    content: "{{ config_content }}"
    mode: '0644'
    owner: root
    group: root

# ❌ 错误 - 缺少权限设置
- name: 创建配置文件
  ansible.builtin.copy:
    dest: /etc/myapp/config.conf
    content: "{{ config_content }}"
```

## 项目特定约束

### Ansible 角色设计原则 - 核心约束
**重要理念**: "角色专注安装 | 安装时解决问题 | 避免后续修复"

#### 角色职责边界
```yaml
# ✅ 角色应该做的事情
- 创建目录结构和设置正确权限
- 安装和配置服务
- 验证安装结果
- 提供完整的工作环境

# ❌ 角色不应该包含的内容
- 单独的修复任务（如修复权限问题）
- 故障排查任务
- 临时性的补丁操作
- 针对特定问题的后续修复
```

#### SSH 操作约束 - 严格限制
**操作边界**: SSH 执行命令仅限于以下场景

```bash
# ✅ 允许的 SSH 操作
ssh host "docker logs container-name"          # 查看日志
ssh host "docker restart container-name"       # 重启服务
ssh host "docker stop container-name"          # 停止服务
ssh host "docker rm container-name"            # 删除容器
ssh host "docker network rm network-name"      # 删除网络
ssh host "sudo rm -rf /path/to/old/data"       # 必要的删除操作

# ❌ 禁止的 SSH 操作
ssh host "chmod 755 /some/directory"           # 权限修复应在角色中
ssh host "chown user:group /some/path"         # 用户组修复应在角色中
ssh host "mkdir -p /some/new/directory"        # 目录创建应在角色中
ssh host "echo 'config' > /etc/file.conf"     # 配置修改应在角色中
```

#### 权限处理最佳实践
**核心原则**: 在安装阶段就处理好所有权限问题

```yaml
# ✅ 正确做法 - 在安装时设置权限
- name: 创建应用数据目录
  ansible.builtin.file:
    path: "{{ app_data_directory }}"
    state: directory
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: '0755'
  become: true

- name: 部署应用服务
  community.docker.docker_container:
    name: "app"
    volumes:
      - "{{ app_data_directory }}:/data"
    # 其他配置...

# ❌ 错误做法 - 先部署再修复权限
- name: 部署应用服务
  community.docker.docker_container:
    name: "app"
    volumes:
      - "{{ app_data_directory }}:/data"

- name: 修复数据目录权限  # 这是修复任务，不应该存在
  ansible.builtin.file:
    path: "{{ app_data_directory }}"
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: '0755'
    recurse: true
  become: true
```

#### 环境配置一次到位原则
```yaml
# ✅ 在部署时就包含所有必要的环境变量
- name: 部署 API 服务
  community.docker.docker_container:
    name: "app-api"
    env:
      DB_HOST: "{{ db_host }}"
      REDIS_HOST: "{{ redis_host }}"
      PLUGIN_DAEMON_URL: "http://plugin-daemon:5002"  # 关键配置一次到位
      # 所有必要的环境变量...

# ❌ 避免后续添加缺失的环境变量
- name: 添加缺失的插件守护进程配置  # 这表明初始安装不完整
  community.docker.docker_container:
    name: "app-api"
    env:
      PLUGIN_DAEMON_URL: "http://plugin-daemon:5002"
```

#### 重新部署策略
**最佳实践**: 遇到配置问题时，优先选择完全重新部署

```bash
# ✅ 推荐的解决流程
1. 完全清理现有部署: ssh host "docker stop && docker rm && sudo rm -rf"
2. 修正角色配置文件
3. 重新执行 ansible-playbook

# ❌ 避免的修复流程  
1. 保留现有部署
2. 通过 SSH 手动修复权限/配置
3. 添加单独的修复任务到角色中
```

### NFS 配置约束
- **服务器**: b360m (192.168.50.94) 作为 NFS 服务器
- **客户端**: 仅 y9000p 挂载 NFS 共享
- **安全**: 限制网络访问范围为 192.168.50.0/24
- **性能**: 使用 NFSv4 协议和优化的挂载选项

```yaml
# NFS 挂载任务模板
- name: 创建 NFS 挂载点目录
  ansible.builtin.file:
    path: "{{ nfs_mount_point }}"
    state: directory
    mode: '0755'
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
  when: inventory_hostname == 'y9000p'
  become: true

- name: 挂载 NFS 共享
  ansible.posix.mount:
    path: "{{ nfs_mount_point }}"
    src: "{{ nfs_server_host }}:{{ nfs_export_path }}"
    fstype: nfs4
    opts: "{{ nfs_mount_options }}"
    state: mounted
  when: inventory_hostname == 'y9000p'
  become: true
  register: nfs_mount_result
  retries: 3
  delay: 5
  until: nfs_mount_result is succeeded
```

### Docker 配置约束
- **用户**: 将 {{ ansible_user }} 添加到 docker 组
- **存储驱动**: 使用 overlay2
- **日志限制**: 最大 100MB，保留 3 个文件
- **网络**: 使用自定义网络进行容器隔离

#### Docker 容器依赖管理 - 重要约束
**核心原则**: `community.docker.docker_container` 模块不支持 `depends_on` 参数

```yaml
# ❌ 错误 - docker_container 不支持 depends_on
- name: 部署应用服务
  community.docker.docker_container:
    name: "app"
    image: "myapp:latest"
    depends_on:  # 这个参数不被支持！
      - database
      - redis

# ✅ 正确 - 使用 Ansible 任务检查依赖
- name: 等待数据库容器就绪
  community.docker.docker_container_info:
    name: "database"
  register: dify_db_container_info
  retries: 30
  delay: 5
  until: dify_db_container_info.container.State.Status == "running" and dify_db_container_info.container.State.Health.Status == "healthy"

- name: 等待 Redis 容器就绪
  community.docker.docker_container_info:
    name: "redis"
  register: dify_redis_container_info
  retries: 30
  delay: 5
  until: dify_redis_container_info.container.State.Status == "running"

- name: 部署应用服务
  community.docker.docker_container:
    name: "app"
    image: "myapp:latest"
    # 其他配置参数
```

#### Docker 变量命名规范
**强制要求**: 角色内所有 `register` 变量必须使用角色前缀

```yaml
# ✅ 正确 - 使用角色前缀
- name: 检查容器状态
  community.docker.docker_container_info:
    name: "my-container"
  register: dify_container_info  # 使用 dify_ 前缀

# ❌ 错误 - 缺少角色前缀
- name: 检查容器状态
  community.docker.docker_container_info:
    name: "my-container"
  register: container_info  # 会被 ansible-lint 标记为错误
```

### 安全配置约束
- **SSH**: 禁用密码认证，仅允许密钥认证
- **防火墙**: 使用 UFW 管理防火墙规则
- **用户权限**: 最小权限原则，按需使用 become
- **密钥管理**: 使用 Ansible Vault 管理敏感数据

### 主机配置约束
- **时区**: 统一设置为 Asia/Shanghai
- **主机名**: 与 inventory 中的名称保持一致
- **包管理**: 定期更新系统包，但生产环境避免使用 latest

## 任务命名规范

### 中文任务名称
```yaml
- name: 更新软件包缓存
- name: 安装必需的软件包
- name: 配置 SSH 安全设置
- name: 创建应用程序目录
- name: 挂载 NFS 共享存储
- name: 重启相关服务
```

### 变量命名规范
```yaml
# 全局变量 - 中文注释
ansible_user: kchou              # 系统用户名
timezone: "Asia/Shanghai"        # 系统时区
homelab_appdata_directory: "/home/{{ ansible_user }}/appdata"  # 应用数据目录

# NFS 变量 - 中文注释
nfs_server_host: "192.168.50.94"     # NFS 服务器地址
nfs_export_path: "/mnt/nas"           # NFS 导出路径
nfs_mount_point: "/mnt/shared"        # NFS 挂载点
nfs_mount_options: "defaults,nfsvers=4,rsize=1048576,wsize=1048576,hard,intr"  # NFS 挂载选项
```

## 验证机制规范 - 强制要求

### 验证原则
**核心理念**: "配置完成 ≠ 功能可用 | 验证是配置的必要组成部分"

#### 验证层次结构
1. **状态验证**: 确认资源存在且状态正确
2. **功能验证**: 测试实际功能是否工作
3. **权限验证**: 确认访问权限和安全性
4. **清理验证**: 自动清理测试数据

### 验证任务模板

#### 服务验证模板
```yaml
# 1. 状态验证
- name: 验证服务状态
  ansible.builtin.systemd:
    name: "{{ service_name }}"
    state: started
  register: service_status

# 2. 功能验证
- name: 测试服务端点响应
  ansible.builtin.uri:
    url: "http://{{ ansible_default_ipv4.address }}:{{ service_port }}/health"
    method: GET
    status_code: 200
  delegate_to: localhost
  register: service_health_check

# 3. 显示验证结果
- name: 显示服务验证结果
  ansible.builtin.debug:
    msg: "{{ service_name }} 服务验证成功，状态: {{ service_status.state }}"
```

#### 挂载点验证模板
```yaml
# 1. 状态验证
- name: 验证挂载点状态
  ansible.builtin.stat:
    path: "{{ mount_point }}"
  register: mount_stat

# 2. 功能验证 - 写入测试
- name: 测试挂载点写入权限
  ansible.builtin.copy:
    content: |
      功能测试文件
      创建时间: {{ ansible_date_time.iso8601 }}
      主机: {{ inventory_hostname }}
    dest: "{{ mount_point }}/.ansible_test"
    mode: '0644'
  register: write_test

# 3. 功能验证 - 读取测试
- name: 验证测试文件内容
  ansible.builtin.slurp:
    src: "{{ mount_point }}/.ansible_test"
  register: read_test

# 4. 清理测试数据
- name: 清理测试文件
  ansible.builtin.file:
    path: "{{ mount_point }}/.ansible_test"
    state: absent
  always:
    - name: 确保测试文件被清理
      ansible.builtin.file:
        path: "{{ mount_point }}/.ansible_test"
        state: absent
      ignore_errors: true
```

#### 网络连接验证模板
```yaml
# 1. 基础连接验证
- name: 验证网络连通性
  ansible.builtin.wait_for:
    host: "{{ target_host }}"
    port: "{{ target_port }}"
    state: started
    timeout: 30

# 2. 功能验证
- name: 测试应用程序响应
  ansible.builtin.uri:
    url: "{{ target_url }}/api/status"
    method: GET
    return_content: true
  register: app_response

# 3. 内容验证
- name: 验证响应内容
  ansible.builtin.assert:
    that:
      - app_response.json.status == "healthy"
    fail_msg: "应用程序状态检查失败"
    success_msg: "应用程序状态正常"
```

#### 文件系统验证模板
```yaml
# 1. 目录存在性验证
- name: 验证目录结构
  ansible.builtin.stat:
    path: "{{ item }}"
  register: dir_stats
  loop:
    - "{{ app_directory }}"
    - "{{ data_directory }}"
    - "{{ log_directory }}"

# 2. 权限验证
- name: 验证目录权限
  ansible.builtin.file:
    path: "{{ item.path }}"
    owner: "{{ expected_owner }}"
    group: "{{ expected_group }}"
    mode: "{{ expected_mode }}"
    state: directory
  loop: "{{ dir_stats.results }}"
  when: item.stat.exists

# 3. 磁盘空间验证
- name: 检查磁盘空间
  ansible.builtin.setup:
    filter: ansible_mounts
  register: disk_info

- name: 验证磁盘空间充足
  ansible.builtin.assert:
    that:
      - item.size_available > minimum_free_space
    fail_msg: "磁盘空间不足: {{ item.mount }}"
  loop: "{{ ansible_mounts }}"
  when: item.mount in critical_mount_points
```

### 错误处理和重试机制

#### 重试机制标准
```yaml
- name: 执行可能失败的操作
  ansible.builtin.command:
    cmd: some-command
  register: command_result
  retries: 3
  delay: 5
  until: command_result is succeeded
  failed_when: command_result.rc not in [0]
```

#### 条件检查
```yaml
- name: 验证先决条件
  ansible.builtin.assert:
    that:
      - prerequisite_service is defined
      - prerequisite_service | length > 0
    fail_msg: "先决条件未满足"
    success_msg: "先决条件验证通过"
```

### 验证最佳实践

#### ✅ 推荐做法
```yaml
# 使用专用的 Ansible 模块
- name: 验证服务状态
  ansible.builtin.systemd:
    name: nginx
    state: started

# 使用 stat 模块验证文件存在
- name: 验证配置文件存在
  ansible.builtin.stat:
    path: /etc/app/config.yml
  register: config_stat

# 使用 assert 模块进行断言
- name: 确认配置文件存在
  ansible.builtin.assert:
    that: config_stat.stat.exists
    fail_msg: "配置文件不存在"

# 使用 wait_for 模块等待服务就绪
- name: 等待服务端口就绪
  ansible.builtin.wait_for:
    port: 8080
    host: localhost
    state: started
    timeout: 60
```

#### ❌ 避免的反模式
```yaml
# 不要使用 shell/command 来检查文件存在
- name: 检查文件存在 (错误做法)
  ansible.builtin.shell: test -f /etc/config.yml

# 不要忽略验证失败
- name: 危险的验证 (错误做法)
  ansible.builtin.uri:
    url: http://localhost:8080/health
  ignore_errors: true

# 不要跳过清理步骤
- name: 测试文件 (错误做法)
  ansible.builtin.copy:
    content: "test"
    dest: /tmp/test
  # 缺少清理步骤
```

### 验证任务组织

#### 任务分组结构
```yaml
# 主要配置任务
- name: 配置 NFS 挂载
  ansible.posix.mount:
    path: "{{ nfs_mount_point }}"
    src: "{{ nfs_server }}:{{ nfs_export }}"
    fstype: nfs4
    state: mounted

# 验证任务组 - 使用 tags 组织
- name: 验证 NFS 挂载状态
  ansible.builtin.stat:
    path: "{{ nfs_mount_point }}"
  register: nfs_mount_stat
  tags: [validation, nfs]

- name: 测试 NFS 读写功能
  ansible.builtin.copy:
    content: "test"
    dest: "{{ nfs_mount_point }}/.test"
  register: nfs_write_test
  tags: [validation, nfs]

- name: 清理 NFS 测试文件
  ansible.builtin.file:
    path: "{{ nfs_mount_point }}/.test"
    state: absent
  tags: [validation, nfs, cleanup]
```

### 验证报告

#### 结果汇总
```yaml
- name: 汇总验证结果
  ansible.builtin.debug:
    msg: |
      验证完成摘要:
      - NFS 挂载: {{ 'SUCCESS' if nfs_mount_stat.stat.exists else 'FAILED' }}
      - 写入权限: {{ 'SUCCESS' if nfs_write_test is succeeded else 'FAILED' }}
      - 服务状态: {{ 'SUCCESS' if service_status.state == 'started' else 'FAILED' }}
  tags: [validation, report]
```

## 强制验证要求

### 验证检查清单
- [ ] 每个配置任务都有对应的验证任务
- [ ] 验证包含状态、功能、权限三个层次
- [ ] 使用适当的 Ansible 模块而非 shell/command
- [ ] 包含错误处理和重试机制
- [ ] 自动清理测试数据
- [ ] 提供清晰的成功/失败反馈

### 验证覆盖要求
- **服务配置**: 状态 + 端点响应 + 功能测试
- **文件操作**: 存在性 + 权限 + 内容验证
- **网络配置**: 连通性 + 响应时间 + 功能测试
- **挂载操作**: 挂载状态 + 读写权限 + 空间检查
- **用户权限**: 身份验证 + 权限测试 + 安全检查

## 质量保证

### 自动验证要求 - 强制执行
**重要**: 每次修改 Ansible 文件后，必须自动运行以下验证命令

#### 虚拟环境要求
```bash
# 激活项目虚拟环境并配置 cowsay
source .venv/bin/activate
export ANSIBLE_COW_PATH=/opt/homebrew/bin/cowsay
```

#### 必须通过的检查 - 按顺序执行
```bash
# 1. Ansible Lint 检查 (生产环境配置) - 最高优先级
ansible-lint --profile production roles/ playbooks/ inventory/

# 2. YAML 格式检查
yamllint roles/ playbooks/ inventory/

# 3. 语法检查
ansible-playbook --syntax-check homelab.yml

# 4. 干运行测试 (可选，用于最终验证)
ansible-playbook --check homelab.yml
```

#### 验证结果要求
- ✅ **ansible-lint**: 必须显示 "Passed" 且 0 failure(s)
- ✅ **yamllint**: 无输出表示通过
- ✅ **syntax-check**: 显示 "playbook: homelab.yml" 表示通过
- ⚠️ **所有检查必须 100% 通过才能提交代码**

### 提交前检查清单
- [ ] 所有任务名称使用中文
- [ ] 使用 FQCN 格式的模块名
- [ ] 布尔值使用 true/false
- [ ] 文件操作指定权限
- [ ] Docker 容器使用正确的依赖管理（避免 depends_on）
- [ ] 角色变量使用正确前缀（如 dify_）
- [ ] 通过 ansible-lint 检查
- [ ] 通过语法检查
- [ ] 敏感信息使用 Vault 加密

### 模块使用验证
**重要提醒**: 使用新模块或不熟悉的参数前，务必：
1. 通过 Context7 查询官方文档
2. 验证参数支持情况
3. 查看示例用法
4. 测试基本功能

## 禁止的反模式

### ❌ 避免的做法
```yaml
# 不要使用 shell/command 替代专用模块
- name: 安装软件包
  ansible.builtin.shell: apt install -y nginx

# 不要硬编码路径
- name: 复制文件
  ansible.builtin.copy:
    src: /home/user/file.txt
    dest: /etc/file.txt

# 不要忽略错误
- name: 危险操作
  ansible.builtin.command: rm -rf /tmp/*
  ignore_errors: true

# 不要在生产环境使用 latest
- name: 安装软件包
  ansible.builtin.package:
    name: nginx
    state: latest
```

### ✅ 推荐的做法
```yaml
# 使用专用模块
- name: 安装软件包
  ansible.builtin.package:
    name: nginx
    state: present

# 使用变量管理路径
- name: 复制文件
  ansible.builtin.copy:
    src: "{{ source_file }}"
    dest: "{{ dest_path }}/file.txt"
    mode: '0644'

# 正确处理错误
- name: 清理临时文件
  ansible.builtin.find:
    paths: /tmp
    age: 7d
    file_type: file
  register: temp_files

- name: 删除过期临时文件
  ansible.builtin.file:
    path: "{{ item.path }}"
    state: absent
  loop: "{{ temp_files.files }}"

# 使用具体版本
- name: 安装软件包
  ansible.builtin.package:
    name: nginx
    state: present
```

## Context7 文档查询指南

### 使用场景
当遇到 Ansible 模块参数问题或需要查看最新文档时，使用 Context7 获取官方权威信息。

### 查询流程
```
1. 解析库 ID: mcp__context7__resolve-library-id
2. 获取文档: mcp__context7__get-library-docs
```

### 常用库 ID 映射
```yaml
# Ansible 官方集合
community.docker: "/ansible-collections/community.docker"
community.general: "/ansible-collections/community.general" 
community.postgresql: "/ansible-collections/community.postgresql"
ansible.posix: "/ansible-collections/ansible.posix"

# 查询示例
topic: "docker_container"  # 具体模块名
tokens: 8000  # 文档长度限制
```

### 实际应用案例
**问题**: `community.docker.docker_container` 模块报错 `deploy` 参数不支持

**解决过程**:
1. 使用 Context7 查询 `/ansible-collections/community.docker` 文档
2. 发现模块支持的参数列表中没有 `deploy` 和 `depends_on`  
3. 了解到这些是 Docker Compose 特有参数，不适用于单容器管理
4. 改用 `community.docker.docker_container_info` 实现依赖检查

### 最佳实践
- **验证参数**: 使用前通过 Context7 确认模块支持的参数
- **查看示例**: 获取官方使用示例和最佳实践
- **版本兼容**: 检查不同版本间的参数变化
- **错误排查**: 遇到模块错误时优先查询官方文档

## 常见问题排查指南

### Docker 模块问题

#### 问题: "Unsupported parameters for (community.docker.docker_container) module"
**症状**: 
```
fatal: [y9000p]: FAILED! => {"changed": false, "msg": "Unsupported parameters for (community.docker.docker_container) module: deploy. Supported parameters include: api_version, auto_remove, blkio_weight..."}
```

**原因分析**:
- `depends_on` 和 `deploy` 是 Docker Compose 特有参数
- `community.docker.docker_container` 模块管理单个容器，不支持编排参数

**解决方案**:
1. 移除不支持的参数（`depends_on`, `deploy` 等）
2. 使用 Ansible 任务实现依赖管理：
```yaml
- name: 等待依赖容器就绪
  community.docker.docker_container_info:
    name: "dependency-container"
  register: dify_dependency_info
  retries: 30
  delay: 5
  until: dify_dependency_info.container.State.Status == "running"
```

#### 问题: "Variables names from within roles should use [role]_ as a prefix"
**症状**:
```
var-naming[no-role-prefix]: Variables names from within roles should use dify_ as a prefix. (register: container_info)
```

**解决方案**:
```yaml
# ❌ 错误
register: container_info

# ✅ 正确  
register: dify_container_info
```

### 健康检查最佳实践
```yaml
# 检查容器运行状态和健康状态
until: |
  dify_container_info.container.State.Status == "running" and 
  (dify_container_info.container.State.Health.Status == "healthy" or 
   dify_container_info.container.State.Health is not defined)
```

### 验证检查要点
- ✅ 使用 `community.docker.docker_container_info` 检查容器状态
- ✅ 提供足够的重试次数和延迟时间
- ✅ 检查健康状态（如果容器定义了健康检查）
- ✅ 变量命名使用角色前缀

## 项目治理

### Git 提交规范
```
feat: 添加 NFS 挂载配置
fix: 修复 Docker 网络配置问题
docs: 更新 Ansible 约束文档
refactor: 重构角色目录结构
```

### 文档要求
- 所有角色必须包含中文 README
- 变量必须在 defaults/main.yml 中注释
- 复杂逻辑需要添加中文注释

---

**强制合规**: 所有 Ansible YAML 文件必须通过 `ansible-lint --profile production` 检查才能部署。

**持续改进**: 定期审查和更新约束文档，确保与最佳实践保持同步。
