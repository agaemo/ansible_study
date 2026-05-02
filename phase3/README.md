# フェーズ3：ロールで構成管理する

Playbookをロールに分割し、再利用可能な構成管理コードを書く。

---

## 教材

### ロールとは

Playbook が大きくなってきたときに、機能単位で分割・再利用するための仕組みです。「nginxをセットアップする」「MySQLをセットアップする」といった単位でロールを作ります。

### ディレクトリ構成

```
roles/
└── webserver/              # ロール名
    ├── tasks/
    │   └── main.yml        # タスクのメイン（必須）
    ├── handlers/
    │   └── main.yml        # ハンドラー
    ├── templates/
    │   └── nginx.conf.j2   # Jinja2テンプレート
    ├── files/
    │   └── ssl.crt         # そのままコピーするファイル
    ├── vars/
    │   └── main.yml        # ロール内変数（優先度高い）
    ├── defaults/
    │   └── main.yml        # デフォルト変数（優先度低い・上書き前提）
    ├── meta/
    │   └── main.yml        # 依存ロールの宣言
    └── README.md
```

すべてのディレクトリが必要なわけではありません。`tasks/main.yml` だけでも動作します。

**ディレクトリ名と `main.yml` は Ansible の規約で固定です。**
Playbook で `- webserver` と書くと、Ansible は自動的に `roles/webserver/tasks/main.yml` をエントリーポイントとして読み込みます。`tasks/start.yml` のようにファイル名を変えても認識されません。`templates/` と `files/` はファイル名が自由ですが、それ以外のディレクトリは `main.yml` が起点になります。

### ロールの呼び出し

```yaml
# Playbook からロールを使う
- name: Webサーバーセットアップ
  hosts: webservers
  roles:
    - webserver                    # シンプルな指定

    - role: webserver              # 変数を渡す場合
      vars:
        app_port: 8080
```

### defaults vs vars の使い分け

```yaml
# defaults/main.yml（優先度低い）
# → 呼び出し元から上書きされることを想定したデフォルト値
app_port: 80
server_name: localhost

# vars/main.yml（優先度高い）
# → ロール内で固定したい値。基本的に外から上書きしない
_internal_config_dir: /etc/myapp
```

**原則:** 外から変えてほしい変数は `defaults`、ロール実装の詳細は `vars` に入れます。

### meta/main.yml による依存関係

```yaml
# roles/webserver/meta/main.yml
dependencies:
  - role: common       # webserverの前に common ロールを実行
  - role: ssl
    vars:
      ssl_port: 443
```

### ロールの入手：Ansible Galaxy

コミュニティが公開しているロールを使えます。

```bash
# ロールを検索
ansible-galaxy search nginx

# ロールをインストール（roles/ ディレクトリに展開）
ansible-galaxy install geerlingguy.nginx

# requirements.yml でまとめて管理
ansible-galaxy install -r requirements.yml
```

```yaml
# requirements.yml
roles:
  - name: geerlingguy.nginx
    version: "3.2.0"
  - name: geerlingguy.mysql
```

### コレクション (Collection)

ロールをさらに大きな単位でまとめたもの。モジュール・プラグイン・ロールをパッケージとして配布できます。

```bash
ansible-galaxy collection install community.general
ansible-galaxy collection install amazon.aws
```

FQCN（完全修飾コレクション名）で書くのが現代のベストプラクティスです。
例: `ansible.builtin.apt`（`apt` だけでも動くが推奨はFQCN）

---

## ハンズオン

### STEP 1：サンプルロールの構成を読む

まず本リポジトリの `roles/webserver/` を開いて構成を確認します。

```
roles/webserver/
├── tasks/main.yml       # nginxインストール・設定・起動
├── handlers/main.yml    # nginx再起動ハンドラー
├── templates/
│   ├── nginx.conf.j2    # 仮想ホスト設定テンプレート
│   └── index.html.j2    # インデックスページテンプレート
└── defaults/main.yml    # app_name・app_port などのデフォルト値
```

`defaults/main.yml` の変数がどこで使われているかを `tasks/main.yml` と `templates/` を見比べながら確認してみましょう。

---

### STEP 2：05_roles.yml を実行する

[`playbooks/05_roles.yml`](../playbooks/05_roles.yml) を開いて内容を確認してから実行します。

```bash
ansible-playbook playbooks/05_roles.yml
```

**確認ポイント:**
- `roles: - webserver` の1行だけでロール全体が適用されること
- `roles/webserver/tasks/main.yml` の各タスクが順に実行されること
- テンプレートから生成されたファイルの内容を確認する

```bash
# target1 に入ってnginxの設定ファイルを確認
docker compose exec target1 cat /etc/nginx/sites-available/myapp

# インデックスページを確認（ファクト変数が展開されているか）
docker compose exec target1 cat /var/www/myapp/index.html
```

---

### STEP 3：変数を上書きしてロールを呼ぶ

Playbookからロールに変数を渡して動作を変えてみます。

```bash
ansible-playbook playbooks/05_roles.yml -e "app_name=testapp app_port=8080"
```

`defaults/main.yml` のデフォルト値が上書きされ、別の設定が適用されることを確認します。

---

### STEP 4：自分でロールを追加してみる（演習）

以下のいずれかを自分で作ってみましょう。

**演習A：commonロール**

全ホストに適用する共通設定をロールにまとめます。

```bash
# ロールのスケルトンを生成
ansible-galaxy role init roles/common
```

実装アイデア:
- タイムゾーンを `Asia/Tokyo` に設定する
- `vim` `curl` `tree` など基本パッケージをインストールする
- `motd`（ログイン時のメッセージ）をテンプレートで設定する

**演習B：databaseロール**

`target2` に MySQL または PostgreSQL をセットアップするロールを作ります。

実装アイデア:
- パッケージをインストールする
- サービスを起動・自動起動設定する
- `defaults/main.yml` にDB名・ユーザー名をデフォルト変数として定義する

---

### 公式ドキュメント

- [ロール開発ガイド](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
- [Ansible Galaxy ユーザーガイド](https://docs.ansible.com/ansible/latest/galaxy/user_guide.html)
- [コレクション利用ガイド](https://docs.ansible.com/ansible/latest/collections_guide/index.html)

---

→ [ベストプラクティス](../README.md#ベストプラクティス) へ
