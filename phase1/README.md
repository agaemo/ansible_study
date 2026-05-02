# フェーズ1：インベントリと疎通確認

「どのサーバーを管理するか」を定義し、Ansibleが実際に動く感覚をつかむ。

---

## 教材

### インベントリとは

管理対象サーバーの一覧ファイルです。「どのサーバーを管理するか」をAnsibleに伝えます。

### INI形式とYAML形式の使い分け

インベントリファイルはINI形式とYAML形式のどちらでも書けます。機能面での差はほぼなく、好みやチームの方針で選んで構いません。

| | INI形式 | YAML形式 |
|---|---|---|
| 記述量 | 少なく簡潔 | やや多い |
| 複雑なネスト | 書きにくい | 書きやすい |
| 普及度 | **公式ドキュメント・チュートリアルの大半がこちら** | 大規模・複雑な構成で採用されることが多い |

**INI形式がデファクトスタンダードです。** 公式ドキュメントや世の中のチュートリアルのほとんどがINI形式で書かれているため、他の資料を参照する際に読めると便利です。本リポジトリもINI形式を採用しています。

### INI形式（本リポジトリで使用）

```ini
# グループなし（ungrouped）
192.168.1.10

# [グループ名] でグループを作る
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com

# グループ変数（そのグループ全体に適用）
[webservers:vars]
ansible_user=deploy
ansible_port=2222

# 子グループ（グループのグループ）
[production:children]
webservers
dbservers
```

### YAML形式

```yaml
all:
  children:
    webservers:
      hosts:
        web1.example.com:
        web2.example.com:
    dbservers:
      hosts:
        db1.example.com:
          ansible_host: 10.0.0.5   # ホスト固有変数
```

### 接続変数

| 変数 | 説明 | 例 |
|---|---|---|
| `ansible_host` | 実際の接続先IP/ホスト | `192.168.1.10` |
| `ansible_port` | SSHポート | `22`（デフォルト） |
| `ansible_user` | SSHユーザー | `ubuntu` |
| `ansible_password` | SSHパスワード（非推奨） | `secret` |
| `ansible_ssh_private_key_file` | 秘密鍵のパス | `~/.ssh/id_rsa` |
| `ansible_python_interpreter` | Python パス | `/usr/bin/python3` |

> **本番環境ではパスワードではなくSSH鍵認証を使いましょう。**

### 特殊グループ

| グループ | 対象 |
|---|---|
| `all` | インベントリ内の全ホスト |
| `ungrouped` | どのグループにも属さないホスト |

### 動的インベントリ

AWSやGCPなどクラウド環境では、サーバーが増減するためファイルベースのインベントリは現実的ではありません。スクリプトやプラグインでサーバー一覧を動的に取得できます。

```bash
# AWS EC2の例（ansible-community/amazon.aws コレクション）
ansible-inventory -i aws_ec2.yml --list
```

---

## ハンズオン

### 事前準備：環境を起動する

```bash
# リポジトリのルートで実行
docker compose up -d --build

# コントロールノードに入る
docker compose exec control bash
```

以降のコマンドはすべてコントロールノード（コンテナ内）で実行します。

---

### STEP 1：インベントリの確認

まず管理対象サーバーの一覧を確認します。

> **target1 / target2 という名前について**
> `inventory/hosts.ini` で使っている `target1` `target2` は、`docker-compose.yml` の `hostname` 設定と一致しています。
> Dockerの内部ネットワーク（`ansible-net`）がこのホスト名を名前解決するため、IPアドレスを書かずに接続できます。

```bash
# ツリー形式で確認
ansible-inventory --graph
```

**期待する出力:**
```
@all:
  |--@ungrouped:
  |--@targets:
  |  |--target1
  |  |--target2
  |--@webservers:
  |  |--target1
  |--@dbservers:
  |  |--target2
```

`inventory/hosts.ini` の内容がどのように解釈されているかを確認できます。

```bash
# 特定ホストの変数を確認
ansible-inventory --host target1
```

---

### STEP 2：ad-hocコマンドで疎通確認

Playbookを書かずに単発でコマンドを実行できます。まず全ホストへの疎通を確認します。

```bash
ansible all -m ping
```

**期待する出力:**
```
target2 | SUCCESS => {
    "ping": "pong"
}
target1 | SUCCESS => {
    "ping": "pong"
}
```

`SUCCESS` が返れば SSH接続 + Python実行が正常に動いています。

---

### STEP 3：01_ping.yml を実行する

**このステップの目的:** STEP 2のad-hocコマンドと違い、Playbookとして手順をファイルに記述することを体験します。
また `gather_facts: yes` でAnsibleが自動収集するシステム情報（ファクト）を確認します。

[`playbooks/01_ping.yml`](../playbooks/01_ping.yml) を開いて内容を確認してから実行します。

```bash
ansible-playbook playbooks/01_ping.yml
```

**期待する出力（抜粋）:**
```
TASK [ターゲットのOSとIPアドレスを表示] ***
ok: [target1] => {
    "msg": "target1 / OS: Debian 12 / IP: 172.x.x.x"
}
```

**確認ポイント:**
- `gather_facts: yes` によってOSやIPが自動収集されていること
- `{{ ansible_facts['distribution'] }}` などのファクト変数が展開されていること
- ファクト変数は `ansible_facts["キー名"]` 形式が推奨（`ansible_distribution` のようなトップレベル変数は将来廃止予定）

---

### STEP 4：02_packages.yml を実行する

[`playbooks/02_packages.yml`](../playbooks/02_packages.yml) を開いて内容を確認してから実行します。

```bash
ansible-playbook playbooks/02_packages.yml
```

**1回目の実行後（抜粋）:**
```
TASK [複数パッケージをインストール] ***
changed: [target1]
changed: [target2]
```

**もう一度実行してみる:**
```bash
ansible-playbook playbooks/02_packages.yml
```

```
TASK [複数パッケージをインストール] ***
ok: [target1]
ok: [target2]
```

**確認ポイント:**
- 1回目は `changed`（実際にインストールした）、2回目は `ok`（すでにインストール済み）になること
- これが**冪等性**です。何度実行しても結果が同じになります

---

### インベントリ確認コマンド（参考）

```bash
ansible-inventory --list           # JSON形式で全情報
ansible-inventory --graph          # ツリー形式
ansible-inventory --host target1   # 特定ホストの変数
```

---

### 終了：環境を片付ける

```bash
exit                  # コントロールノードから抜ける
docker compose down   # コンテナとネットワークを削除
```

各コマンドの違いは[トップの README](../README.md#終了片付け) を参照してください。

---

### 公式ドキュメント

このフェーズで使うモジュールのリファレンスです。`state: present` などのパラメーター詳細はここで確認できます。

- [`ansible.builtin.ping`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ping_module.html)
- [`ansible.builtin.debug`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html)
- [`ansible.builtin.apt`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)

---

→ [フェーズ2：Playbookを書く](../README.md#フェーズ2playbookを書く) へ
