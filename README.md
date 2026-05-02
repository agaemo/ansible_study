# Ansible 学習リポジトリ

Infrastructure as Code（IaC）の観点から Ansible を体系的に学ぶためのリポジトリです。
理論ドキュメントとDockerを使ったハンズオンをセットで提供しています。

> このリポジトリは [Claude Code](https://claude.ai/code) を使って作成しました。
> 内容の正確性には注意を払っていますが、必ず公式ドキュメントと合わせて確認してください。

---

## 必要なもの

| ツール | 用途 |
|---|---|
| [Docker Desktop](https://www.docker.com/products/docker-desktop/) | コンテナ環境の実行 |
| Docker Compose v2 | 複数コンテナの管理（Docker Desktop に同梱） |
| テキストエディタ | Playbook・設定ファイルの編集 |

ローカルに Ansible をインストールする必要はありません。コントロールノード（コンテナ）内で実行します。

---

## セットアップ

```bash
git clone https://github.com/agaemo/ansible_study.git
cd ansible_study
docker compose up -d --build
docker compose exec control bash
```

以降のコマンドはすべてコントロールノード（コンテナ内）で実行します。

---

## 終了・片付け

作業が終わったらコントロールノードから抜け、コンテナを停止・削除します。

```bash
# コントロールノードから抜ける
exit

# コンテナを停止して削除する（ネットワークも削除される）
docker compose down
```

| コマンド | 動作 |
|---|---|
| `docker compose stop` | コンテナを停止するだけ（次回 `up` で再開できる） |
| `docker compose down` | コンテナとネットワークを削除する（イメージは残る） |
| `docker compose down --rmi all --volumes` | イメージとボリュームも含めて完全削除 |

---

## 学習フェーズ

### はじめに：Ansibleとは何か

構成管理ツールの概念と、Ansibleが選ばれる理由を理解する。

- エージェントレスの仕組み（SSHとPythonだけで動く理由）
- 冪等性：「何をするか」ではなく「どういう状態にするか」を書く
- Inventory / Module / Task / Playbook / Role の関係
- 実行フローと PLAY RECAP の読み方

→ [教材を読む](intro/README.md)

---

### フェーズ1：インベントリと疎通確認

「どのサーバーを管理するか」を定義し、Ansibleが実際に動く感覚をつかむ。

- インベントリのグループ・ホスト変数・接続設定
- INI形式とYAML形式の使い分け
- ad-hocコマンドで ping・ファクト収集

→ [教材とハンズオン](phase1/README.md)

---

### フェーズ2：Playbookを書く

タスクを束ねたPlaybookを書き、冪等なコードを書く感覚を身につける。

- apt / copy / template / service など頻出モジュールの使い方
- `register` でコマンド結果を受け取り・`when` で条件分岐
- 変数・ファクト・Jinja2テンプレートで設定ファイルを動的に生成
- `notify` と `handlers` でサービス再起動をトリガーする

→ [教材とハンズオン](phase2/README.md)

---

### フェーズ3：ロールで構成管理する

Playbookをロールに分割し、再利用可能な構成管理コードを書く。

- ロールのディレクトリ構成（tasks / handlers / templates / defaults / vars）
- `defaults` と `vars` の使い分け・変数の優先順位
- Ansible Galaxy でコミュニティ製ロールを活用する
- 自分でロールを追加する演習（commonロール・databaseロールなど）

→ [教材とハンズオン](phase3/README.md)

---

### ベストプラクティス

本番環境で安全に運用するために知っておくべきことを整理する。

- Ansible Vault でパスワード・APIキーを暗号化する
- SSH鍵認証・最小権限・`--check` でのドライラン
- `ansible-lint` による静的解析
- 本番適用前のチェックリスト

→ [教材を読む](best_practices/README.md)

---

## 学習を終えて

Ansibleの核心は「**あるべき状態を宣言する**」という思想にあります。「nginxをインストールする」ではなく「nginxがインストールされている状態にする」と書くことで、冪等性が保たれ、何度実行しても安全なコードになります。

ハンズオンでPlaybookを繰り返し実行しているうちに、`ok` と `changed` の違いが直感的にわかるようになります。その感覚を身につけることが、本リポジトリの一番の目標です。

---

## ディレクトリ構成

```
.
├── ansible.cfg                   # Ansible設定ファイル
├── docker-compose.yml            # コントロール + ターゲット2台
├── docker/
│   ├── control/Dockerfile        # Ansible実行環境（Ubuntu + Ansible）
│   └── target/Dockerfile         # 管理対象サーバー（Ubuntu + SSH）
├── inventory/
│   └── hosts.ini                 # ホスト・グループ・変数の定義
├── playbooks/                    # 学習用Playbook（ステップ順）
│   ├── 01_ping.yml
│   ├── 02_packages.yml
│   ├── 03_variables.yml
│   ├── 04_templates.yml
│   └── 05_roles.yml
├── roles/
│   └── webserver/                # サンプルロール
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       ├── templates/
│       │   ├── nginx.conf.j2
│       │   └── index.html.j2
│       └── defaults/main.yml
├── intro/README.md               # はじめに（教材）
├── phase1/README.md              # フェーズ1（教材 + ハンズオン）
├── phase2/README.md              # フェーズ2（教材 + ハンズオン）
├── phase3/README.md              # フェーズ3（教材 + ハンズオン）
└── best_practices/README.md      # ベストプラクティス
```

---

## 参考資料

| 資料 | 説明 |
|---|---|
| [Ansible公式ドキュメント](https://docs.ansible.com/) | モジュールリファレンス・ガイド |
| [Ansible Galaxy](https://galaxy.ansible.com/) | コミュニティ製ロール・コレクション |
| [Ansible Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/index.html) | 公式のベストプラクティス集 |
