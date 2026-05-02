# はじめに：Ansibleとは何か

## Ansibleがない世界の問題

サーバーが1台であれば、手動でSSHログインして設定することも現実的です。しかしサーバーが10台・100台になると、次のような問題が起きます。

- **作業ミス・漏れ**: 同じコマンドを手作業で繰り返すと、設定のtypoや手順のスキップが発生する
- **環境の差異**: 「このサーバーだけ設定が違う」という状態が生まれ、本番障害の原因になる
- **再現性がない**: 誰がいつどの手順でセットアップしたかが記録に残らず、同じ環境を再構築できない
- **スケールできない**: 担当者が増えるほど手順の属人化が進み、サーバーの増減に対応できなくなる

## Ansibleで解決できること

Ansibleはこれらの問題を「**構成をコードで管理する（Infrastructure as Code）**」という考え方で解決します。

```yaml
# Playbookの例：どのサーバーにも同じ状態を適用できる
- name: nginxをインストールして起動する
  hosts: webservers        # 対象サーバーのグループ
  tasks:
    - apt:
        name: nginx
        state: present     # "インストール済み"という状態を宣言
    - service:
        name: nginx
        state: started
        enabled: yes
```

このPlaybookを実行するだけで、対象の全サーバーに同じ状態を適用できます。手作業の代わりにコードが手順書になるため、**誰が実行しても同じ結果になり、Gitで変更履歴も管理できます**。

## 主な用途

| 用途 | 具体例 |
|---|---|
| サーバーのセットアップ自動化 | パッケージインストール・ユーザー作成・設定ファイル配置 |
| アプリケーションのデプロイ | コードの配置・サービス再起動・ヘルスチェック |
| 設定の一括変更 | 100台のサーバーのsshd設定を一度に変更する |
| 環境の再現 | 開発・ステージング・本番を同じPlaybookで構築する |
| 定期的なメンテナンス | セキュリティパッチの適用・ログローテーション |

---

## エージェントレスとは

構成管理ツールには「エージェント型」と「エージェントレス型」の2種類があります。

**エージェント型**（Chef, Puppetなど）では、管理対象サーバーに専用デーモン（エージェント）を事前インストールします。エージェントが定期的にマスターサーバーへ問い合わせ、自分自身の設定を適用します。

```
エージェント型
  masterサーバー ◄─── エージェント（常駐プロセス）がポーリング
                       ↑ 各サーバーにインストール必要
```

**エージェントレス型**（Ansible）では、管理対象サーバーに何もインストールしません。実行のたびにコントロールノードからSSHで接続し、必要な処理だけを送り込んで実行します。

```
エージェントレス型
  control node ──SSH──► サーバー（エージェント不要）
                         Python3 が入っていれば動く
```

| | メリット | デメリット |
|---|---|---|
| エージェントレス | 対象サーバーに何もいれない・セットアップが簡単 | 実行のたびにSSH接続が発生する（大規模では遅い） |
| エージェント型 | 継続的な状態管理・大規模でも高速 | エージェントの管理・バージョン管理が必要 |

---

## 他のツールとの比較

| ツール | 方式 | 特徴 |
|---|---|---|
| Ansible | エージェントレス (SSH) | 学習コスト低・対象サーバーに何も入れない |
| Chef / Puppet | エージェント型 | 大規模向け・学習コスト高 |
| Terraform | 宣言型IaC | インフラ構築向け（サーバー設定は苦手） |

Ansibleは「すでにあるサーバーを設定する」用途に強く、Terraformは「インフラ自体を作る」用途に強いため、両者は補完関係にあります。

---

## 冪等性（べきとうせい）

Ansibleの核心概念。**何度実行しても同じ結果になる** ことを指します。

```yaml
- name: nginxをインストール
  apt:
    name: nginx
    state: present   # "インストール済み" という状態を宣言
```

このタスクは、nginxが未インストールなら install し、すでにインストール済みなら何もしません（`changed: false`）。「何をするか」ではなく「**どういう状態にするか**」を書くのがAnsibleのスタイルです。

---

## 主要コンポーネント

### インベントリ (Inventory)
管理対象サーバーの一覧。IPアドレスやホスト名、グループ分けを定義します。
通常は `inventory/` ディレクトリを作り、`hosts.ini` というファイル名で保存します。

```
inventory/
└── hosts.ini
```

```ini
# inventory/hosts.ini
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com
```

### モジュール (Module)
Ansibleが持つ機能の単位。数千種類ありますが、よく使うものは限られています。

`ansible.builtin.apt` はその代表例で、Ubuntu/Debian の `apt` コマンドをAnsibleから操作するモジュールです。`state: present` と書くと「インストールされている状態にする」、`state: absent` なら「アンインストールされている状態にする」という意味になります。

**よく使うモジュール一覧:**

| モジュール | 対応するLinuxコマンドのイメージ | 主な用途 |
|---|---|---|
| [`ansible.builtin.apt`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html) | `apt install / apt remove` | Ubuntu/Debian のパッケージ管理 |
| [`ansible.builtin.yum`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_module.html) | `yum install / yum remove` | RHEL/CentOS のパッケージ管理 |
| [`ansible.builtin.copy`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html) | `cp` | ファイルをコントロールノードからコピー |
| [`ansible.builtin.template`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html) | `cp`（変数展開あり） | Jinja2テンプレートを展開してコピー |
| [`ansible.builtin.file`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html) | `mkdir / chmod / chown / rm` | ファイル・ディレクトリの作成・削除・権限変更 |
| [`ansible.builtin.lineinfile`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/lineinfile_module.html) | `sed` | ファイルの特定行を追加・変更・削除 |
| [`ansible.builtin.service`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html) | `systemctl start/stop/enable` | サービスの起動・停止・自動起動設定 |
| [`ansible.builtin.command`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html) | そのままコマンドを実行 | 単純なコマンド実行（パイプ不可） |
| [`ansible.builtin.shell`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/shell_module.html) | `bash -c "..."` | シェル経由のコマンド実行（パイプや `&&` が使える） |
| [`ansible.builtin.user`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html) | `useradd / usermod` | ユーザーの作成・変更 |
| [`ansible.builtin.ping`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ping_module.html) | `ssh` で接続確認 | 疎通確認（ICMPのpingとは別物） |
| [`ansible.builtin.debug`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html) | `echo` | 変数の中身や任意のメッセージを出力 |

> `ansible.builtin.` というプレフィックスはAnsible本体に同梱されていることを示します。コミュニティ製モジュールは `community.general.xxx` や `amazon.aws.xxx` のような別のプレフィックスになります。

### タスク (Task)
モジュールを呼び出す1つの操作単位。

```yaml
- name: わかりやすいタスク名（ログに出る）
  ansible.builtin.apt:      # モジュール名
    name: nginx             # インストールするパッケージ名
    state: present          # present=インストール / absent=アンインストール / latest=最新版に更新
```

### Playbook
複数のタスクをまとめた1つのYAMLファイル。`ansible-playbook` コマンドに直接指定して実行する**エントリーポイント**です。「どのホストに」「何をするか」を定義します。

### ロール (Role)
タスク・ハンドラー・テンプレート・変数などをまとめた**再利用可能な部品**。ディレクトリとして存在しますが、`ansible-playbook` コマンドで直接実行することはできません。必ずPlaybookの中から名前で呼び出します。

```
ansible-playbook playbooks/site.yml   # ← 常にPlaybookファイルを指定する
        ↓
  site.yml の中に roles: - webserver と書く
        ↓
  Ansible が roles/webserver/ を探して実行する
```

```yaml
# playbooks/05_roles.yml の中身
- hosts: webservers
  roles:
    - webserver    # ディレクトリパスではなく名前で呼ぶ
```

`roles/` はAnsibleが**デフォルトで探しに行く規約のディレクトリ名**です。`- webserver` と書かれると、Ansibleは次の順で `webserver` という名前のロールを探します。

1. Playbookファイルと同じ階層の `roles/webserver/`
2. コマンドを実行したディレクトリの `roles/webserver/`
3. `ansible.cfg` の `roles_path` に設定したパス（このリポジトリでは `roles/`）

---

## ファイル構成とコンポーネントの対応

```
ansible_study/
├── ansible.cfg           # Ansible全体の設定（インベントリパス・接続方式など）
├── inventory/
│   └── hosts.ini         # インベントリ：管理対象サーバーの一覧とグループ定義
├── playbooks/
│   └── site.yml          # Playbook：タスクをまとめたYAMLファイル
└── roles/
    └── webserver/        # ロール名（Playbookから "webserver" という名前で呼ぶ）
        ├── tasks/
        │   └── main.yml
        ├── templates/
        │   └── nginx.conf.j2
        └── handlers/
            └── main.yml
```

### ansible.cfg がインベントリを指定する仕組み

`ansible-playbook` を実行すると、Ansibleはまず同じディレクトリの `ansible.cfg` を読み込みます。このファイルに `inventory = inventory/hosts.ini` と書かれているため、コマンドにインベントリのパスを指定しなくても自動的に読み込まれます。

```ini
# ansible.cfg の抜粋
[defaults]
inventory = inventory/hosts.ini   # ← インベントリの場所
roles_path = roles                # ← ロールの検索パス（未設定だとPlaybookと同じ階層を探す）
```

`roles_path` を設定しないと、Ansibleはプレイブックファイルの隣にある `roles/` を探しに行きます。プレイブックが `playbooks/` 配下にある場合は `playbooks/roles/` が対象になり、プロジェクトルートの `roles/` が見つからずエラーになります。

`ansible.cfg` を置かずに毎回指定する場合は `-i` オプションで渡すこともできます。

```bash
ansible-playbook -i inventory/hosts.ini playbooks/site.yml
```

---

## 実行の流れ

Ansibleを使うときの基本コマンドは `ansible-playbook` です。引数に実行したいPlaybookのファイルを指定します。

```bash
ansible-playbook playbooks/site.yml
#               └─ 実行するPlaybookファイルのパス
```

| ステップ | 内容 |
|---|---|
| 1. インベントリを読み込む | `inventory/hosts.ini` を読んで「どのサーバーに対して実行するか」を決める |
| 2. 対象サーバーにSSH接続する | コントロールノードから管理対象サーバーへSSH接続する。ネットワーク疎通と認証情報が必要 |
| 3. ファクトを収集する | サーバーのOS・IP・メモリなどを自動収集する（`gather_facts: yes` のとき）。Playbook内で変数として使える |
| 4. タスクを上から順に実行する | Playbookに書いたタスクを1つずつ実行する。各タスクは下表のいずれかの状態を返す |
| 5. PLAY RECAP を表示する | 全タスクが終わると実行結果のサマリーが出る |

タスクの実行結果（ステップ4）:

| 状態 | 意味 |
|---|---|
| `ok` | 成功。すでに望ましい状態だったので何も変更しなかった |
| `changed` | 成功。実際にサーバーに変更を加えた |
| `failed` | タスクがエラーで失敗した（以降のタスクはスキップ） |
| `skipped` | `when` 条件が満たされなかったのでスキップした |

PLAY RECAP のサマリー例:

```
PLAY RECAP ************************************
target1 : ok=3  changed=1  unreachable=0  failed=0  skipped=0
target2 : ok=3  changed=0  unreachable=0  failed=0  skipped=0
```

`target1` は3タスク成功・うち1つで実際に変更あり、`target2` は3タスク成功・変更なし（すでに望ましい状態だった）という意味です。`failed=0` と `unreachable=0` であれば正常終了です。

---

## チェックモード（ドライラン）

`--check` オプションで実際には変更せずに「何が変わるか」を確認できます。

```bash
ansible-playbook playbook.yml --check
```

本番環境に適用する前に必ず確認する習慣をつけましょう。

---

→ [フェーズ1：インベントリと疎通確認](../README.md#フェーズ1インベントリと疎通確認) へ
