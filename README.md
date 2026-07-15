------

# 👑 Nginx 障害対応演習

## 📖 演習概要

本演習は、EC2（Amazon Linux）上に構築した Nginx Web サーバーを対象に、実際の企業インフラ運用で発生しうる障害対応の一連の流れを体験することを目的とする。

**核心思想：** 正常確認 → 変更管理 → 障害発生 → 状態確認 → ログ解析 → 原因特定 → 修復 → 安全反映 → 復旧確認 → 履歴記録

- 演練ディレクトリ：`~/ec2-user/tf-pro/linux2/nginx-lab`

------

## 📂 リポジトリ構成（Directory Structure）

GitHub 上および実務において専門性を示すため、以下のディレクトリ構成を維持する。

```
nginx-lab/
├── README.md          # 演習目的、手順と説明
├── backup/            # 設定ファイルのバックアップ格納ディレクトリ
├── logs/               # 過去のトラブルシューティングログの控え
└── change.log          # 本演習の障害対応・変更記録（Change Record）
```

### 【準備作業】EC2 上でのディレクトリ初期化

```bash
cd ~/tf-pro/linux2
mkdir -p nginx-lab/{backup,logs} && cd nginx-lab
touch change.log
```

> `mkdir -p nginx-lab/{backup,logs}` は、最終的に `mkdir -p nginx-lab/backup nginx-lab/logs` と等価である。

------

## 🛠️ 障害対応演習 7つのフェーズ

### Phase 1: 構築（Setup）＆ 起動確認

Nginx をインストールし、OS レイヤーでのサービス稼働状況を直ちに確認する。

```bash
# 1. Nginx のインストール
sudo dnf install -y nginx

# 2. サービスの起動と自動起動の有効化
sudo systemctl start nginx
sudo systemctl enable nginx

# 3. サービス起動状態の確認（Active: active (running) を確認）
sudo systemctl status nginx
```

![01-nginx-normal](E:\GitHub\nginx-incident-response-lab\screenshots\01-nginx-normal.png)

**`01-nginx-normal.png`** 

（Nginx が正常に起動し、`active (running)` となっている状態のスクリーンショット）

------

### Phase 2: 正常確認（Verify）＆ AWS連携確認

自動化運用の慣習に沿ったコマンドラインツールを用いてサービスの健全性を検証し、ネットワークの listen 状態を確認する。

```bash
# 1. HTTP ヘッダーのみを取得し、ステータスコードを抽出
# 期待される出力: HTTP/1.1 200 OK
curl -I http://localhost/index.html | grep HTTP
```

- `curl -I`：HTTPレスポンスヘッダーのみを取得する（本文はダウンロードしない）。運用における迅速なサイト状態確認に利用。
- `|`（パイプ）：前段コマンドの出力を後段コマンドの入力として渡す。
- `grep HTTP`：出力の中から `HTTP` を含む行のみを抽出する。

```bash
# 2. ソケット統計を確認し、インフラ層への影響を把握
# 期待される出力: LISTEN 0 511 0.0.0.0:80
sudo ss -tulnp | grep :80
```

- `ss`：Socket Statistics。ネットワーク接続・ポート使用状況を確認するツール。
- `-tulnp`：`-t`（TCPのみ）、`-u`（UDPも含める）、`-l`（LISTEN状態のみ）、`-n`（数値表示）、`-p`（プロセス情報表示）。
- `grep :80`：80番ポート（HTTPデフォルトポート）の記録のみを抽出。

![05-nginx-stauts](E:\GitHub\nginx-incident-response-lab\screenshots\05-nginx-stauts.png)

**`05-nginx-stauts.png`** （`curl -I` および `ss -tulnp` の正常な実行結果）

------

### Phase 3: 変更前バックアップ（Backup）

設定変更前に、秒単位のタイムスタンプ付きバックアップを取得し、同日内の複数回変更による上書きを防止する。

```bash
# 1. 秒単位のタイムスタンプを付与してバックアップを実施
sudo cp /etc/nginx/nginx.conf ./backup/nginx.conf.$(date +%F-%H%M%S)
```

- `sudo cp`：管理者権限でコピー。`/etc/nginx/nginx.conf` はシステムファイルのため sudo が必要。

- ```
  $(date +%F-%H%M%S)
  ```

  ：コマンド置換によりタイムスタンプをファイル名に埋め込む。

  - `%F`：年月日（例：2026-07-14）
  - `%H%M%S`：時分秒（例：230045）
  - 生成例：`nginx.conf.2026-07-14-230045`

```bash
# 2. バックアップの存在確認
ls -l ./backup/
```

------

### Phase 4: 障害発生（Break）

本番環境で頻出する2種類の障害を意図的に再現する。

#### 手法A：権限異常（403 Forbidden 障害の再現）

```bash
# ※このリンクは作業用ショートカットであり、Nginxの実際のDocumentRootではない
ln -s /usr/share/nginx/html ./html_dir
```

- `ln -s`：シンボリックリンク（ショートカット）を作成。
- `/usr/share/nginx/html`：Nginx の DocumentRoot（Webページ格納先）。
- 目的：長いパスを毎回入力せずに、カレントディレクトリから直接操作できるようにする。

```bash
# インデックスファイルの権限を 000（読込不可）に変更
sudo chmod 000 /usr/share/nginx/html/index.html
```

- 通常は 644（所有者は読み書き可、それ以外は読み取りのみ）。Nginx の Worker Process（`nginx` や `nobody` ユーザー）が読み取れるよう「その他」の読み取り権限が必要。
- `chmod 000` により、あらゆるユーザー・プロセスからのアクセスを完全に遮断する。

#### 手法B：設定ミス（404 Not Found 障害の再現）

```bash
sudo vim /etc/nginx/nginx.conf
```

`server { }` ブロック内の `root` ディレクティブを、存在しないパスに書き換える。

```
root /usr/share/nginx/no_such_dir;
sudo systemctl reload nginx
```

> **重要：** 通常は設定変更後に直接 reload してはならず、必ず `sudo nginx -t`（構文チェック）を先に実行する。ただし本演習では、`nginx -t` を実行しても `test is successful`（構文は完全に正しい）と表示される点に注意。
>
> - **構文適合性チェック**：セミコロン `;` の欠落や中括弧 `{}` の対応をチェック。
> - **ディレクティブ適合性チェック**：`root` のようなキーワードのスペルミスをチェック。
> - **物理ファイルシステムは検査しない**：指定パスが実在するかどうかは確認しない。
> - **ファイル権限も検査しない**：手法Aのような権限問題も検出できない。
>
> ⚠️ **「構文が通ること」と「サービスが利用可能であること」はイコールではない。**

![02-404-error](E:\GitHub\nginx-incident-response-lab\screenshots\02-404-error.png)

![04-404-log](E:\GitHub\nginx-incident-response-lab\screenshots\04-404-log.png)

**`02-404-error.png`**  **`04-404-log`** （404障害発生時の curl 結果） 

![03-403-error](E:\GitHub\nginx-incident-response-lab\screenshots\03-403-error.png)

![09-trigger-403](E:\GitHub\nginx-incident-response-lab\screenshots\09-trigger-403.png)

**`03-403-error.png`**    **`09-trigger-403`** （403障害発生時の curl 結果）

------

### Phase 5: 障害調査（Debug）

`curl` でステータスコードの異常を検知した際に実施する、トラブルシューティング三段階。

```bash
# 異常の検知（403 または 404 になることを確認）
curl -I http://localhost/index.html | grep HTTP
# STEP 1: サービス状態の確認（プロセスが生存しているか）
sudo systemctl status nginx
# STEP 2: アプリケーションログの確認（Nginxエラーログの監視）
sudo tail -n 20 /var/log/nginx/error.log
```

- **手法A（403権限エラー）の場合**： `... [error] ... open() "/usr/share/nginx/html/index.html" failed (13: Permission denied) ...` → Linuxファイル権限の問題。`chmod` による修復が必要。
- **手法B（404パスエラー）の場合**： `... [error] ... "no such file or directory" ...` → 設定ファイルの `root` パスまたはリクエストファイル名の誤り。

![04-404-log](E:\GitHub\nginx-incident-response-lab\screenshots\04-404-log.png)

**`04-404-log.png`**（404原因を示す error.log の抜粋） 

![10-permission-error](E:\GitHub\nginx-incident-response-lab\screenshots\10-permission-error.png)

**`10-permission-error.png`**（403原因を示す Permission denied ログ） 

![06-error-log](E:\GitHub\nginx-incident-response-lab\screenshots\06-error-log.png)

**`06-error-log.png`**（error.log 確認全体像）

```bash
# STEP 3: OS/システムログの確認（Systemdジャーナルからの多角的解析）
sudo journalctl -u nginx --since "10 minutes ago" -n 50
```

- `journalctl`：systemd のシステムログ管理ツール。
- `-u nginx`：nginx サービスユニットに関するログのみ表示。
- `--since "10 minutes ago"`：直近10分間のログに限定する運用上の必須パラメータ。
- `-n 50`：直近50件に制限。

![07-system-log](E:\GitHub\nginx-incident-response-lab\screenshots\07-system-log.png)

**`07-system-log.png`**（journalctl の実行結果）

------

### Phase 6: 復旧（Recover）

#### 🧅 第一段階：主要因の除去（404パス問題の修復）

この時点では error.log に `no_such_dir` のみが確認され、「設定ミスだ、バックアップから即座にロールバックしよう」と判断する。

```bash
# 1. バックアップファイルの確認
ls -l ./backup/

# 2. 正常な設定ファイルへロールバック（手法Bの解決）
# ※タイムスタンプは実際の値に置き換えること
sudo cp ./backup/nginx.conf.2026-07-14-140259 /etc/nginx/nginx.conf

# 3. 構文チェック（無条件で実施）
sudo nginx -t

# 4. チェック通過後、Nginx を平滑リロード
sudo systemctl reload nginx
```

![08-fix-404](E:\GitHub\nginx-incident-response-lab\screenshots\08-fix-404.png)

**`08-fix-404.png`**（ロールバックおよびリロード実行の様子）

#### 🚨 第二段階：隠れた障害の顕在化（403 Forbidden 再発）

```bash
# 5. 復旧確認（1回目）
curl -I http://localhost/index.html | grep HTTP
# → HTTP/1.1 403 Forbidden
```

404は解消されたはずが、今度は403エラーが発生。再度調査フローを実施する。

```bash
# 6. エラーログを再確認し、隠れていた「第二の原因」を特定
sudo tail -n 20 /var/log/nginx/error.log
# → failed (13: Permission denied)
```

![09-trigger-403](E:\GitHub\nginx-incident-response-lab\screenshots\09-trigger-403.png)

**`09-trigger-403.png`**（404修復後に403が顕在化する様子）

#### 🍃 第三段階：根本原因の完全排除（403権限問題の修復）

```bash
# 7. index.html の権限を標準の 644 に復元（手法Aの解決）
sudo chmod 644 /usr/share/nginx/html/index.html
```

> 💡 ファイル権限のみの変更であり Nginx 設定は変更していないため、`nginx -t` も `reload` も不要。Nginx はファイルシステムを動的に参照する。

```bash
# 8. 最終復旧確認
curl -I http://localhost/index.html | grep HTTP
# → HTTP/1.1 200 OK
```

![11-fix-403](E:\GitHub\nginx-incident-response-lab\screenshots\11-fix-403.png)

**`11-fix-403.png`**（chmod 644 実行と最終復旧確認の様子）

------

### Phase 7: 変更履歴記録（Change Record）

障害解決後は必ず変更内容を記録する。これは優秀な運用エンジニアと一般的なプログラマを分ける重要な工程である。

`change.log` に本対応記録を追記する。

```bash
cat << EOF >> change.log
==================================================
対応日時: $(date "+%Y-%m-%d %H:%M:%S")
対応者: 曹志祥（So Shisho）
対象サービス: Nginx Web Server
障害内容: 設定変更後の HTTP 404 障害、および連鎖的な 403 Forbidden 障害の発生

根本原因（Root Cause）:
  - 要因B（404エラー）: nginx.conf 内の root ディレクトリ設定に、存在しないパスが指定されていたため。
  - 要因A（403エラー）: 物理ファイルシステムの index.html のパーミッションが意図せず 000（アクセス不可）に変更されていたため。

対応内容（Action Taken）:
  1. nginx.conf を既存の正常なバックアップファイルからロールバック（復元）。
  2. 適用前に 'nginx -t' による構文チェックを実施し、正常終了を確認。
  3. 'systemctl reload nginx' を実行し、設定を無停止で平滑反映。
  4. 再度アクセスを試みた際、非表示だった要因A（403エラー）が浮上したため、index.html のパーミッションを 644 へ修復。

対応結果（Result）:
  curl による最終復旧確認を実施し、HTTP 200 OK の返却を確認。サービス完全復旧。
==================================================
EOF
```

![12-final-record](E:\GitHub\nginx-incident-response-lab\screenshots\12-final-record.png)

**`12-final-record.png`**（change.log の最終記録内容）

------

## 📌 演習まとめ

| フェーズ | 内容                | 主な使用コマンド                                        |
| -------- | ------------------- | ------------------------------------------------------- |
| Phase 1  | 構築・起動確認      | `dnf install`, `systemctl start/enable/status`          |
| Phase 2  | 正常確認            | `curl -I`, `ss -tulnp`                                  |
| Phase 3  | バックアップ        | `cp`, `date`                                            |
| Phase 4  | 障害発生（403/404） | `chmod 000`, `vim nginx.conf`                           |
| Phase 5  | 障害調査            | `systemctl status`, `tail error.log`, `journalctl`      |
| Phase 6  | 復旧                | `cp`（ロールバック）, `nginx -t`, `reload`, `chmod 644` |
| Phase 7  | 変更履歴記録        | `cat >> change.log`                                     |

**本演習で得られる学び：**

- 「構文チェック通過」＝「サービス正常稼働」ではないという運用上の重要な教訓
- 複数の障害要因が連鎖的に存在する場合の段階的トラブルシューティング手法
- 変更管理・履歴記録の重要性（実務運用における必須プロセス）