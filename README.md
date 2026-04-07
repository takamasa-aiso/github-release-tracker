# github-release-tracker

GitHub Actions を使った自動化ワークフロー。  
外部リポジトリのリリースを監視し、ローカル(WSL2)へ自動反映 + Discord通知を行います。

---

## ワークフロー一覧

```
GitHub サーバ
  └─ リポジトリの .github/workflows/github-release-tracker.yml を読む
       └─ 時間(cron)になったら ローカル(WSL2)の self-hosted runner にジョブを送る

WSL2上の self-hosted runner サービス（常駐）
  └─ GitHubのジョブキューをポーリングしてジョブを受信
       └─ [Step 1] Check latest release
            ├─ GitHub APIで OWNER/REPO の最新リリースを取得
            ├─ .upstream-version-REPO名 の現在バージョンと比較
            ├─ 差分なし → updated=false を GITHUB_OUTPUT に書き込み → Step 2をスキップ → 終了
            └─ 差分あり → cd /path/tp/repo
                          git fetch upstream
                          git checkout $LATEST
                          .upstream-version-REPO名 を最新バージョンで上書き
                          updated=true / latest=$LATEST を GITHUB_OUTPUT に書き込み
                               └─ [Step 2] Notify update（条件: updated==true）
                                    └─ Discord Webhook に curl でPOST
                                         └─ Discordに通知が届く
```

---

## github-release-tracker.yml

### 概要

```
GitHub Actions (スケジュール: 30分ごと)
  └─ GitHub API で監視対象リポジトリの最新リリースを確認
       ├─ 更新なし → 何もしない
       └─ 更新あり → self-hosted runner (WSL2) で git checkout
                      └─ Discord に通知
```

### 監視対象リポジトリ

| リポジトリ | ローカルパス |
|------------|-------------|
| `OWNER/REPO` | `/path/to/repo` |

リポジトリの追加は `github-release-tracker.yml` の `matrix.include` に追記します。

---

## セットアップ手順

### 1. リポジトリをfork
https://github.com/takamasa-aiso/github-release-tracker にアクセスして、自身のGithubにforkします。
(Github上にコピーできればfork以外の方法でもOKです)

### 2. リポジトリをクローン (WSL2)

```bash
git clone https://github.com/takamasa-aiso/github-release-tracker.git
cd github-release-tracker
```

### 3. 必要パッケージの確認 (WSL2)

```bash
# 確認
curl --version && jq --version && git --version && tar --version

# 未インストールの場合
sudo apt update && sudo apt install -y curl jq git tar
```

### 4. GitHub Actions self-hosted runner のインストール (WSL2)

GitHubのリポジトリページで Runner を登録・インストールします。

```
このリポジトリ → Settings → Actions → Runners → New self-hosted runner
→ OS: Linux を選択
→ 表示されたコマンドを WSL2 上で実行
```

表示されるコマンドは以下のような形です。

```bash
mkdir ~/actions-runner && cd ~/actions-runner
curl -o actions-runner-linux-x64-*.tar.gz -L https://github.com/actions/runner/releases/download/...
tar xzf ./actions-runner-linux-x64-*.tar.gz
./config.sh --url https://github.com/自分のID/リポジトリ名 --token XXXXX
```

インストール後、サービスとして常駐させます。

```bash
cd ~/actions-runner   # PATHを変えた場合は適宜変更
sudo ./svc.sh install
sudo ./svc.sh start

# 状態確認
sudo ./svc.sh status
```

### 5. Discord Webhook URL の取得

1. Discord で通知を受け取りたいチャンネルを右クリック →「チャンネルの編集」
2. 「連携サービス」→「ウェブフック」→「新しいウェブフック」
3. 名前をつけて「ウェブフックURLをコピー」

### 6. GitHub Secrets への登録

```
このリポジトリ → Settings → Secrets and variables → Actions
→ New repository secret

Name:  DISCORD_WEBHOOK_URL
Value: https://discord.com/api/webhooks/xxxx/yyyy (コピーしたURL)
```

### 7. ワークフローをプッシュ (WSL2)

```bash
git init
vi .github/workflows/github-release-tracker.yml
git add .github/workflows/github-release-tracker.yml
git commit -m "Add watch-upstream workflow"
git remote add origin https://github.com/takamasa-aiso/github-release-tracker.git
git push origin main
```

プッシュ後、リポジトリの「Actions」タブにワークフローが表示されれば完了。

ワークフローに表示されない場合は、yml構文が正しいこと、もしくは設定を確認してみてください。

```
リポジトリ → Settings → Actions → General
→ Actions permissions が「Allow all actions」になっているか
```

---

## 動作確認

手動実行でテストできます。

```
Actions タブ → Watch Upstream Release → Run workflow
```

---

## 監視対象リポジトリの追加方法

`github-release-tracker.yml` の `matrix.include` にブロックを追記。

```yaml
matrix:
  include:
    - repo: OWNER/REPO
      local_path: /path/to/repo
    # ↓ 追加例
    - repo: OWNER/REPO
      local_path: /path/to/repo
```

---

## 注意事項

- self-hosted runner はWSL2上で常駐している必要があります
- Windowsのスリープ中はrunnerが停止します (6時間以内に復帰すればジョブは実行されます)
- GitHub Actionsのスケジュール実行は混雑時に数分〜数十分遅延する場合があります
