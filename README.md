# 叢集監控專案

這個專案使用 Ansible 自動化部署一套完整的叢集監控解決方案，包括：

- 在所有節點上安裝 Node Exporter
- 在所有節點上安裝 DCGM Exporter & Docker
- 在主要監控伺服器上使用 Docker 安裝 Prometheus 和 Grafana
- 自動配置所有組件互相連接

## 專案結構

```plaintext
cluster-monitoring/
├── ansible.cfg           - Ansible 配置設定
├── copy_ssh_key.yml      - 自動複製 SSH 金鑰的腳本
├── inventory/
│   └── hosts.ini         - 伺服器庫存檔案
├── group_vars/
│   ├── all.yml           - 全域變數定義
│   └── monitoring/
│       └── vault.yml     - 加密的敏感資訊
├── site.yml              - 主要部署 playbook
└── roles/
    ├── node_exporter/    - Node Exporter 角色
    │   ├── tasks/
    │   │   └── main.yml  - Node Exporter 安裝任務
    │   ├── handlers/
    │   │   └── main.yml  - Node Exporter 事件處理器
    │   └── templates/
    │       └── node_exporter.service.j2 - systemd 服務模板
    ├── dcgm_exporter/    - DCGM Exporter 角色
    │   ├── tasks/
    │   │   └── main.yml  - DCGM Exporter 安裝任務
    │   ├── handlers/
    │   │   └── main.yml  - DCGM Exporter 事件處理器
    │   └── templates/
    │       └── dcgm_exporter.service.j2 - systemd 服務模板
    └── monitoring/       - 監控伺服器角色
        ├── tasks/
        │   └── main.yml  - 監控伺服器安裝任務
        ├── handlers/
        │   └── main.yml  - 監控伺服器事件處理器
        └── templates/
            ├── prometheus.yml.j2     - Prometheus 配置模板
            ├── docker-compose.yml.j2 - Docker-Compose 配置模板
            ├── datasource.yml.j2     - Grafana 資料源配置
            └── web.yml.j2            - Prometheus 身份驗證配置
```

## Pre-requisites

- 監控伺服器上安裝有 Python 3.x
- 安裝 pipx (Python 包管理器)

    ```bash
    python3 -m pip install --user pipx
    python3 -m pipx ensurepath
    ```

- Ansible 2.9+ 已安裝在控制主機上，且安裝以下插件(collection)
    1. 安裝 Ansible:

        ```bash
            pipx install ansible
        ```

    2. 安裝 Ansible 的插件:

        ```bash
            ansible-galaxy collection install community.docker
            ansible-galaxy collection install community.general
        ```

- 安裝passlib:
  
  ```bash
    /home/<username>/.local/pipx/venvs/ansible-core/bin/pip3 install passlib
  ```

- 目標伺服器為 Linux 系統 (Debian/Ubuntu)
- 監控伺服器上有足夠的存儲空間用於 Prometheus 數據

## 安裝步驟

### 1. 安裝 ssh key到所有伺服器

更改ansible.cfg 檔案中的remote user：

```ini
remote_user = <your_username>
```

確保您可以無密碼 SSH 登入所有伺服器：

```bash
ssh-keygen -t rsa
```

然後將公鑰複製到所有伺服器：

```bash
ansible-playbook copy_ssh_key.yml --ask-pass
```

### 2. 配置庫存檔案

編輯 `inventory/hosts.ini` 檔案，添加您的監控伺服器和被監控節點：

```ini
[monitoring]
monitoring-server ansible_host=<監控伺服器IP>

[nodes]
node1 ansible_host=<節點1 IP>
node2 ansible_host=<節點2 IP>
node3 ansible_host=<節點3 IP>
```

### 3. 配置變數（Optional）

如需調整配置，編輯 `group_vars/all.yml` 檔案：

### 4. 修改 Grafana 以及 Prometheus 密碼

1. 修改 Vault.yml 中的密碼（可選）：

   ```yaml
   # group_vars/monitoring/vault.yml
   vault_prometheus_password: PASSWORD
   vault_grafana_password: PASSWORD
   ```

2. 加密 vault.yml：

   ```bash
   ansible-vault encrypt group_vars/monitoring/vault.yml
   ```

   Note: 這邊會要求輸入密碼，這個密碼會在執行 playbook 時使用。

### 5. 執行部署

```bash
ansible-playbook site.yml --ask-become-pass -K --ask-vault-pass
```

Note: 這邊會要求輸入 sudo 密碼和 vault 密碼。

## 功能說明

### Node Exporter

- 在所有 `[nodes]` 群組中的伺服器上安裝 Node Exporter
- 配置 systemd 服務確保自動啟動
- 開放防火牆端口 9100 用於指標收集
- 配置基本收集器（文件系統、磁盤、CPU、記憶體等）

### DCGM Exporter

- 在所有 `[nodes]` 群組中的伺服器上安裝 DCGM Exporter
- 安裝在 Docker 中
- 配置 systemd 服務確保自動啟動
- 開放防火牆端口 9400 用於指標收集
- 配置基本收集器（GPU 使用率、記憶體使用率等）
- 注意：DCGM Exporter 需要 NVIDIA 驅動和 Docker 支持
- 參考文件: [DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)

### 監控伺服器部署

- 在 `[monitoring]` 群組中的伺服器上安裝 Docker 和 Docker Compose
- 部署 Prometheus 服務並配置數據保留期
- 部署 Grafana 並自動配置 Prometheus 資料源
- 開放必要的防火牆端口（9090 和 3000）

## 訪問監控界面

部署完成後，您可以通過以下 URL 訪問監控界面：

- **Prometheus**: `https://<監控伺服器IP>:9090`
  - 用戶名: `smil`
  - 密碼: <vault.yml 中配置的密碼>
- **Grafana**: `https://<監控伺服器IP>:3000`
  - 用戶名: `smil`
  - 密碼: <vault.yml 中配置的密碼>

## 故障排除

常見問題與解決方案：

1. **Node Exporter 無法連接**
   - 確認防火牆端口 9100 已開放
   - 檢查 Node Exporter 服務狀態: `systemctl status node_exporter`
   - 查看日誌: `journalctl -u node_exporter`

2. **Prometheus 無法搜集數據**
   - 確認 Prometheus 配置文件中的目標 IP 地址正確
   - 確認 Prometheus 容器運行狀態: `docker ps | grep prometheus`
   - 檢查 Prometheus 日誌: `docker logs <prometheus-container-id>`

3. **Grafana 無法顯示數據**
   - 確認 Prometheus 資料源已正確配置
   - 檢查 Grafana 日誌: `docker logs <grafana-container-id>`

## 自訂與擴展

### 添加更多節點

只需在 `inventory/hosts.ini` 中的 `[nodes]` 部分添加新的伺服器條目，然後重新運行 playbook。

### 添加 AlertManager

編輯 `roles/monitoring/files/docker-compose.yml` 添加 AlertManager 服務，並相應地更新 Prometheus 配置。

## 安全注意事項

本監控系統已實施以下安全增強措施：

### 加密密碼管理

- 所有敏感密碼都存儲在 `group_vars/vault.yml` 加密文件中
- 使用 Ansible Vault 保護重要憑證：
  ```bash
  # 加密 vault.yml 文件
  ansible-vault encrypt group_vars/monitoring/vault.yml
  
  # 執行部署時提供 vault 密碼
  ansible-playbook site.yml --ask-vault-pass
  ```

### Node Exporter 安全限制

- Node Exporter 監聽地址限制為本地地址 (127.0.0.1)
- 防火牆規則僅允許監控伺服器訪問 Node Exporter 端口
- 實現方式：
  ```yaml
  # 在 group_vars/all.yml 中設定
  node_exporter_web_listen_address: "127.0.0.1:9100"
  node_exporter_firewall_allow_from: 
    - "{{ hostvars[groups['monitoring'][0]].ansible_host }}"
  ```

### Prometheus 身份驗證與 TLS

- Prometheus 啟用了基本身份驗證機制
- 使用自簽或正式 TLS 證書保護 HTTPS 連接
- 配置說明：
  ```yaml
  # 啟用身份驗證
  prometheus_basic_auth_enabled: true
  prometheus_basic_auth_username: "prometheus_admin"
  prometheus_basic_auth_password: "{{ vault_prometheus_password }}"
  ```

### Grafana 安全配置

- 使用非默認管理員帳戶和強密碼
- 啟用安全 cookie 設定與 HTTPS 連接
- 禁用 Gravatar 和開放註冊
- 配置示例：
  ```yaml
  grafana_security_admin_user: "{{ grafana_admin_user }}"
  grafana_security_admin_password: "{{ grafana_admin_password }}"
  grafana_security_cookie_secure: "true"
  grafana_users_allow_sign_up: "false"
  ```

## 授權

MIT

## 貢獻

歡迎提交問題和請求合併！
