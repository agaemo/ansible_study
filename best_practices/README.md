# ベストプラクティス

本番環境で安全に運用するために知っておくべきことを整理する。

---

## ディレクトリ構成の推奨パターン

Ansible公式が推奨する構成（本リポジトリも準拠）:

```
.
├── ansible.cfg
├── inventory/
│   ├── hosts.ini            # または production / staging など環境別
│   ├── group_vars/
│   │   ├── all.yml
│   │   └── webservers.yml
│   └── host_vars/
│       └── web1.yml
├── playbooks/
│   ├── site.yml             # すべてを束ねるマスターPlaybook
│   ├── webservers.yml
│   └── dbservers.yml
└── roles/
    ├── common/
    ├── webserver/
    └── database/
```

---

## 命名規則

```yaml
# タスク名: 動詞 + 対象
- name: nginxをインストール
- name: 設定ファイルを展開
- name: サービスを起動

# 変数名: スネークケース、ロール名プレフィックスで衝突回避
webserver_port: 80
webserver_document_root: /var/www/html

# ロール名: スネークケース
roles/web_server/     # ○
roles/webServer/      # ×
```

---

## セキュリティ

### SSH鍵認証を使う（本番必須）

```ini
# inventory/hosts.ini（本番環境）
[webservers]
web1.example.com ansible_ssh_private_key_file=~/.ssh/deploy_key
```

パスワード認証（`ansible_password`）は学習環境限定。本番では使わない。

### Ansible Vault でシークレットを暗号化

```bash
# ファイル全体を暗号化
ansible-vault encrypt group_vars/production/secrets.yml

# 暗号化ファイルを編集
ansible-vault edit group_vars/production/secrets.yml

# Playbook実行時にパスワードを渡す
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass
```

```yaml
# group_vars/production/secrets.yml（暗号化前の内容）
db_password: "supersecret"
api_key: "abc123"
```

**シークレットは絶対に平文でGitにコミットしない。**

---

## 冪等性を守るコツ

```yaml
# 悪い例: 実行するたびに changed になる
- ansible.builtin.shell: echo "hello" >> /tmp/log.txt

# 良い例: creates に指定したファイルが存在する場合はスキップ
# → 1回目: ファイルなし → 実行 → changed / 2回目: ファイルあり → スキップ → ok
- ansible.builtin.shell: echo "hello" >> /tmp/log.txt
  args:
    creates: /tmp/log.txt

# changed_when で変更判定を明示
- ansible.builtin.command: /usr/bin/myapp --status
  register: status
  changed_when: false           # このタスクは常に変更なし

- ansible.builtin.command: /usr/bin/myapp --migrate
  register: result
  changed_when: "'migrated' in result.stdout"
```

---

## タグで部分実行

```yaml
tasks:
  - name: nginxをインストール
    ansible.builtin.apt:
      name: nginx
    tags: [nginx, packages]

  - name: 設定ファイルを展開
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    tags: [nginx, config]
```

```bash
# 特定タグだけ実行
ansible-playbook site.yml --tags nginx

# 特定タグをスキップ
ansible-playbook site.yml --skip-tags packages
```

---

## Lint でコードを静的解析

```bash
pip install ansible-lint

ansible-lint playbooks/site.yml
```

CIに組み込んでコード品質を自動チェックするとよいです。

---

## よくあるミスと対処

| 症状 | 原因 | 対処 |
|---|---|---|
| `UNREACHABLE` | SSH接続失敗 | ホスト名/IP・ポート・鍵を確認 |
| `MODULE FAILURE` | Python未インストール | ターゲットに `python3` をインストール |
| 毎回 `changed` になる | 冪等性がない | `changed_when` や `creates` を使う |
| ハンドラーが動かない | `changed` が出ていない | タスクに実際の変更が起きているか確認 |
| 変数が空 | タイポ or スコープ外 | `debug` で変数の値を確認 |

---

## チェックリスト（本番適用前）

- [ ] `--check` でドライランを実行した
- [ ] シークレットはVaultで暗号化している
- [ ] SSH鍵認証を使っている
- [ ] `ansible-lint` でエラーがない
- [ ] ステージング環境で動作確認した
- [ ] ロールバック手順を用意している

---

### 公式ドキュメント

- [Ansible Vault ガイド](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
- [ansible-lint](https://ansible.readthedocs.io/projects/lint/)
- [ベストプラクティス（公式）](https://docs.ansible.com/ansible/latest/tips_tricks/index.html)

---

→ [学習を終えて](../README.md#学習を終えて) へ
