# 叢集監控專案

這個專案使用 Ansible 自動化部署一套完整的叢集監控解決方案，包括：

- 在所有節點上安裝 Node Exporter
- 在所有節點上安裝 DCGM Exporter & Docker
- 在主要監控伺服器上使用 Docker 安裝 Prometheus 和 Grafana
- 可選的 NUT Exporter（用於 UPS 監控）
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
            ansible-galaxy install -r requirements.yml
        ```

- 安裝passlib:

  ```bash
    pipx inject ansible-core passlib
  ```

- 目標伺服器為 Linux 系統 (Debian/Ubuntu)
- 監控伺服器上有足夠的存儲空間用於 Prometheus 數據
- 開發環境需要有 Python 3.x 和 pip (用於 pre-commit 檢查)

### 開發環境設置

開發此專案需要安裝 pre-commit 工具來進行程式碼檢查和格式化：

```bash
# 安裝 pre-commit
pip install pre-commit

# 在專案資料夾中安裝 git hooks
cd cluster-monitoring
pre-commit install

# 可以手動運行檢查所有檔案
pre-commit run --all-files
```

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

如需調整配置，編輯 `group_vars/all.yml` 檔案。


### 4. 修改 Grafana 以及 Prometheus 密碼

1. 修改 Vault.yml 中的密碼（可選）：

   ```yaml
   # group_vars/monitoring/vault.yml
   vault_prometheus_password: PASSWORD
   vault_grafana_password: PASSWORD
   # 如果啟用了 NUT Exporter，請同時設定以下憑證：
   vault_nut_exporter_user: "your_ups_username"
   vault_nut_exporter_password: "your_ups_password"

   # group_vars/nodes/vault.yml
   vault_monitoring_public_ip: ""
   ```

2. 加密 vault.yml：

   ```bash
   ansible-vault encrypt group_vars/monitoring/vault.yml
   ansible-vault encrypt group_vars/nodes/vault.yml
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

### NUT Exporter（UPS 監控）

- 可選功能：監控 UPS（不斷電系統）的狀態和指標
- 部署在監控伺服器上的 Docker 容器中
- 監控端口：9199

#### 啟用 NUT Exporter

1. 在 `group_vars/monitoring/vault.yml` 中配置 UPS 憑證：

   ```yaml
   vault_nut_exporter_user: "your_ups_username"
   vault_nut_exporter_password: "your_ups_password"
   ```

#### NUT Exporter 監控指標

NUT Exporter 會收集以下 UPS 指標：

- 電池電量百分比 (battery.charge)
- 電池剩餘時間 (battery.runtime)
- 電池電壓 (battery.voltage)
- 輸入電壓 (input.voltage)
- 輸出電壓 (output.voltage)
- UPS 負載百分比 (ups.load)
- UPS 狀態 (ups.status)
- UPS 額定功率 (ups.realpower.nominal)

#### 注意事項

- 確保 UPS 伺服器上已安裝和配置 NUT (Network UPS Tools)
- UPS 必須支援 SNMP 或其他 NUT 支援的通訊協定
- 參考文件: [NUT Exporter](https://github.com/DRuggeri/nut_exporter)

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
- **NUT Exporter**: `http://<監控伺服器IP>:9199` (如果已啟用)
  - 提供 UPS 狀態指標，通常透過 Prometheus 和 Grafana 查看

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

4. **NUT Exporter 無法收集 UPS 數據**
   - 確認 `nut_exporter_enabled: true` 已在 `group_vars/all.yml` 中設定
   - 檢查 NUT Exporter 容器狀態: `docker ps | grep nut-exporter`
   - 檢查 NUT Exporter 日誌: `docker logs <nut-exporter-container-id>`
   - 確認 UPS 伺服器上的 NUT 服務正在運行
   - 驗證 UPS 憑證設定正確

## 自訂與擴展

### 添加更多節點

只需在 `inventory/hosts.ini` 中的 `[nodes]` 部分添加新的伺服器條目，然後重新運行 playbook。

### 添加 AlertManager

編輯 `roles/monitoring/files/docker-compose.yml` 添加 AlertManager 服務，並相應地更新 Prometheus 配置。

## 程式碼品質控制

本專案使用了多種工具來確保程式碼品質和一致性：

### Pre-commit 鉤子

pre-commit 工具在每次 git commit 之前自動執行以下檢查：

1. **基本格式檢查**：
   - 移除行末空格
   - 確保文件以空行結束
   - 檢查 YAML 語法
   - 檢查大型文件
   - 檢測合併衝突
   - 檢測私鑰洩漏

2. **YAML 格式檢查**：
   - 使用 yamllint 檢查 YAML 文件格式
   - 設置了適合 Ansible 專案的自訂規則

3. **Ansible 程式碼檢查**：
   - 使用 ansible-lint 檢查 Ansible 最佳實踐
   - 檢查變數引用格式
   - 檢查危險的 shell 指令
   - 檢查密碼安全性

### 手動運行檢查

除了提交時自動執行外，您也可以手動運行檢查：

```bash
# 檢查所有檔案
pre-commit run --all-files

# 檢查特定檔案
pre-commit run --files path/to/file.yml

# 跳過某些檢查
SKIP=ansible-lint pre-commit run
```

### 配置檔案

- `.pre-commit-config.yaml`: 定義了 git pre-commit 鉤子的配置
- `.yamllint`: 自訂的 YAML 格式規則，適合 Ansible 專案
- `.ansible-lint`: Ansible Lint 的配置，設定了特定的規則

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
