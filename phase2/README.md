# フェーズ2：Playbookを書く

タスクを束ねたPlaybookを書き、冪等なコードを書く感覚を身につける。

---

## 教材

### Playbookの基本構造

```yaml
---                          # YAMLドキュメントの開始
- name: Play名               # 1つのPlayの開始
  hosts: webservers          # 対象ホスト/グループ
  gather_facts: yes          # ファクト収集するか
  become: yes                # sudo権限で実行するか

  vars:                      # Play内変数
    app_name: myapp

  tasks:                     # タスクのリスト
    - name: タスク名
      ansible.builtin.apt:
        name: nginx
        state: present

  handlers:                  # notify でトリガーされる処理
    - name: nginx を再起動
      ansible.builtin.service:
        name: nginx
        state: restarted
```

1つのPlaybookファイルに複数のPlayを書くこともできます（`- name:` を並べる）。

### よく使うモジュール

#### ファイル操作

```yaml
# ファイルコピー
- ansible.builtin.copy:
    src: files/nginx.conf
    dest: /etc/nginx/nginx.conf
    owner: root
    mode: '0644'

# テンプレート展開（Jinja2）
- ansible.builtin.template:
    src: templates/app.conf.j2
    dest: /etc/app/app.conf

# ファイル・ディレクトリの作成/削除/パーミッション変更
- ansible.builtin.file:
    path: /opt/myapp
    state: directory
    mode: '0755'

# 特定行を追加・変更
- ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PermitRootLogin'
    line: 'PermitRootLogin no'
```

#### コマンド実行

```yaml
# シェルを使わないコマンド（推奨）
- ansible.builtin.command:
    cmd: /usr/bin/myapp --init
  register: result

# シェルパイプやリダイレクトが必要なとき
- ansible.builtin.shell:
    cmd: "cat /etc/passwd | grep ansible"
```

> `command` と `shell` は何度実行しても `changed` になる（冪等性がない）モジュールです。以下のオプションと組み合わせて冪等にします。
> - `creates: /path/to/file` … 指定したファイルが**すでに存在する場合はスキップ**します。「このコマンドはそのファイルを生成する」という意味を持たせることで、生成済みなら実行不要と判断させます
> - `changed_when: false` … 常に `ok` 扱いにします（ステータス確認など副作用のないコマンドに使う）

### register と結果の利用

```yaml
- name: Pythonバージョンを確認
  ansible.builtin.command: python3 --version
  register: python_version
  changed_when: false       # コマンド実行だけなので changed にしない

- name: バージョンを表示
  ansible.builtin.debug:
    msg: "{{ python_version.stdout }}"
```

`register` で保存したオブジェクトの主なプロパティ:

| プロパティ | 内容 |
|---|---|
| `stdout` | 標準出力（文字列） |
| `stdout_lines` | 標準出力（リスト） |
| `stderr` | 標準エラー出力 |
| `rc` | 終了コード（0=成功） |
| `failed` | 失敗したかどうか（bool） |

### ループ

```yaml
- ansible.builtin.user:
    name: "{{ item }}"
    state: present
  loop:
    - alice
    - bob
    - carol
```

### ハンドラー（Handlers）

「設定ファイルを変えたらサービスを再起動する」のように、**変更があったときだけ実行したい処理**をハンドラーに書きます。

```yaml
tasks:
  - ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: nginx を再起動   # ← changed になったときだけ通知を送る

handlers:
  - name: nginx を再起動    # ← notify の文字列と一致させる
    ansible.builtin.service:
      name: nginx
      state: restarted
```

**ポイント:**

- `notify` はタスクが `changed`（実際に変更が発生した）ときだけ発火します。`ok`（変更なし）では発火しません
- ハンドラーは Play の**最後にまとめて1回だけ**実行されます。複数のタスクが同じハンドラーを notify しても重複しません
- これにより「設定ファイルに変更がなければ再起動しない」という冪等な動作になります

**実行の流れ（設定ファイルを変えた場合）:**

```
タスク: テンプレート展開 → changed → "nginx を再起動" に通知
タスク: シンボリックリンク作成 → ok → 通知なし
         ↓ Play 終了後
ハンドラー: nginx を再起動 → 1回だけ実行
```

**実行の流れ（2回目：何も変わっていない場合）:**

```
タスク: テンプレート展開 → ok → 通知なし
タスク: シンボリックリンク作成 → ok → 通知なし
         ↓ Play 終了後
ハンドラー: 通知がないので実行されない
```

### 変数の優先順位

Ansibleの変数は定義できる場所が多く、**優先順位（高いほど勝つ）** があります。

```
高 ← コマンドライン変数  ansible-playbook -e "key=value"
     タスクのvars
     Play vars
     ロール vars/main.yml
     インベントリグループ変数 (group_vars/)
     インベントリホスト変数 (host_vars/)
     ロール defaults/main.yml → 低
```

| 用途 | 置き場所 |
|---|---|
| デフォルト値（上書き前提） | `roles/xxx/defaults/main.yml` |
| ロール内の固定値 | `roles/xxx/vars/main.yml` |
| 環境ごとの差分 | `group_vars/production.yml` など |
| 本番だけ上書きしたい | `-e` オプション or `host_vars/` |

### ファクト (Facts)

`gather_facts: yes`（デフォルト）のとき、Ansibleが自動収集するシステム情報です。

```yaml
{{ ansible_facts['hostname'] }}                    # ホスト名
{{ ansible_facts['all_ipv4_addresses'] | first }}  # IPアドレス
{{ ansible_facts['distribution'] }}                # OS名 (Debian, Ubuntu...)
{{ ansible_facts['memtotal_mb'] }}                 # 合計メモリ(MB)
{{ ansible_facts['date_time']['iso8601'] }}        # 現在時刻
```

### Jinja2テンプレート

`{{ }}` で変数展開、`{% %}` でロジックを書きます。

```jinja2
server_name {{ server_name }};
listen {{ app_port | default(80) }};

{% if ansible_facts['distribution'] == "Debian" %}
apt_get=true
{% endif %}
```

よく使うフィルター:

```jinja2
{{ app_name | upper }}              {# MYAPP #}
{{ value | default('未設定') }}     {# 変数未定義時のデフォルト値 #}
{{ path | basename }}               {# /etc/nginx/nginx.conf → nginx.conf #}
```

### group_vars / host_vars

インベントリに対応する変数ファイルを分離して管理できます。

```
inventory/
├── hosts.ini
├── group_vars/
│   ├── all.yml           # 全ホスト共通
│   └── webservers.yml    # webserversグループ
└── host_vars/
    └── web1.yml          # web1ホスト固有
```

---

## ハンズオン

### STEP 1：03_variables.yml を実行する

[`playbooks/03_variables.yml`](../playbooks/03_variables.yml) を開いて内容を確認してから実行します。

```bash
ansible-playbook playbooks/03_variables.yml
```

**確認ポイント:**
- `vars:` で定義した変数が `{{ app_name }}` で展開されていること
- `ansible_facts['memtotal_mb']` などのファクト変数が表示されること
- `when: ansible_facts['distribution'] == "Debian"` の条件が評価されること

**変数をコマンドラインから上書きしてみる:**

```bash
ansible-playbook playbooks/03_variables.yml -e "app_port=9090"
```

`-e` オプションは最も優先度が高いため、Playbook内の `app_port: 8080` が上書きされます。

---

### STEP 2：ファクトを直接確認する

```bash
# 全ファクトを表示
ansible target1 -m ansible.builtin.setup

# メモリ関連のファクトだけ絞り込む
ansible target1 -m ansible.builtin.setup -a "filter=ansible_memory*"

# OS情報だけ絞り込む
ansible target1 -m ansible.builtin.setup -a "filter=ansible_distribution*"
```

---

### STEP 3：04_templates.yml を実行する

[`playbooks/04_templates.yml`](../playbooks/04_templates.yml) を開いて内容を確認してから実行します。

```bash
ansible-playbook playbooks/04_templates.yml
```

**確認ポイント:**
- `template` モジュールで `nginx.conf.j2` が展開され、変数が埋め込まれること
- 設定ファイルを変更したとき `notify: nginx を再起動` でハンドラーがトリガーされること
- **もう一度実行すると** テンプレートに変更がないため `ok` になり、ハンドラーも動かないこと

---

### STEP 4：ドライランで変更内容を事前確認する

実際に変更を加える前に `--check` で何が変わるかを確認できます。

```bash
ansible-playbook playbooks/04_templates.yml --check
```

本番環境への適用前に必ず実行する習慣をつけましょう。

---

### 公式ドキュメント

このフェーズで使うモジュールのリファレンスです。`state: present` などのパラメーター詳細はここで確認できます。

- [`ansible.builtin.apt`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)
- [`ansible.builtin.copy`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)
- [`ansible.builtin.template`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)
- [`ansible.builtin.file`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html)
- [`ansible.builtin.lineinfile`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/lineinfile_module.html)
- [`ansible.builtin.command`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html)
- [`ansible.builtin.shell`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/shell_module.html)
- [`ansible.builtin.service`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html)
- [`ansible.builtin.user`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html)
- [`ansible.builtin.debug`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html)
- [`ansible.builtin.setup`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html)

---

→ [フェーズ3：ロールで構成管理する](../README.md#フェーズ3ロールで構成管理する) へ
