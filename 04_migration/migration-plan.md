# Raspberry Pi 移行計画

## 1. 目的と範囲

- 開発環境で動いている 4 つのコンテナ（Samba、rclone、restic、Discord Bot）を、Raspberry Pi 上の**本番モード**（`NODE_ENV=production`）で動かす。
- 本番モードでは、rclone は FUSE マウントではなく **bisync** で、クラウド（Proton Drive）と SSD 上のローカルコピーを 30 分ごとに双方向同期する。restic は SSD 上のコピー（`/data`）を HDD にバックアップする。
- 対象外: 外部公開、新規ハードウェアの調達、Immich（`docker-compose.yml` でコメントアウト）。

## 2. 現状と本番の違い

| 項目 | 開発環境（現在） | 本番（移行後） |
| - | - | - |
| 機器 | x86_64、Arch Linux | Raspberry Pi 4（4GB）、Raspberry Pi OS |
| rclone | `rclone mount`（クラウドを直接参照。ローカルコピーなし） | `rclone bisync`（SSD にローカルコピー） |
| `/sync` | 何もしない | bisync を 1 回実行 |
| restic の対象 | `/config`（動作確認用の小さなディレクトリ） | `/data`（クラウドのローカルコピー全体） |
| 保管先（HDD） | ローカルのディレクトリ | 外付け HDD（exFAT 想定） |
| Discord | `DEV_CH_*` のチャンネル | `PROD_CH_*` のチャンネル |
| 開発用ボタン | あり | なし |
| 共有領域のファイルシステム | NTFS | ext4（SSD） |

**開発環境では、bisync を一度も実行していない。** 同期の動作は、設定と rclone の仕様からの説明にとどまる。

## 3. 事前に決めること

移行の前に、次を決める。あとから変えると、作り直しや再同期が必要になるものを先に挙げる。

| No. | 決めること | 選択肢 | 推奨 | 期限 |
| - | - | - | - | - |
| D1 | **SSD・HDD の暗号化** | 暗号化しない／LUKS で全体を暗号化／restic のみ（restic は常に暗号化済み） | 後から変えるとデータの移し替えが必要になるため、**SSD を初期化する前に決める**。盗難対策の必要性と、起動時のパスフレーズ入力の手間（無人起動ができなくなる）を比べる | Raspberry Pi の準備の前 |
| D2 | **Discord の Bot を、開発用と本番用で分けるか** | 同じ Bot（トークン）を共用／アプリを 2 つに分ける | **分ける**。同じトークンで 2 つのプロセスが動くと、同じ操作に 2 つが応答する。分けない場合は、開発環境の Bot を止めてから本番を動かす | 本番の Bot を起動する前 |
| D3 | **HDD の運用** | 常時マウント／バックアップ時だけマウント（憲章の方針） | 憲章は「普段はアンマウント」。現在のコンテナ構成は常時マウントを前提にしている。**まず常時マウントで検証し、アンマウント運用は別途実装する**（6 の No.5 を参照） | HDD での検証の前 |
| D4 | **HDD のファイルシステム** | exFAT（Windows・Mac でも読める）／ext4（権限・安定性） | restic は exFAT でも動くとされるが、**未検証**。実機で試して決める | HDD の検証の前 |
| D5 | **Raspberry Pi 4 か 5 か** | Pi 4（所有）／Pi 5（検討中） | まず Pi 4 で性能を測り、基準を満たさなければ Pi 5 を検討する（[verification-plan.md](verification-plan.md) の性能の基準） | 性能の検証後 |
| D6 | **クラウドの総容量** | — | 未測定。SSD（1TB）に収まるか、`rclone size` で確認する | 移行の最初 |
| D7 | **開発環境の扱い** | 移行後に停止／検証用に残す | 1 週間の連続稼働が済むまで残し、**同期は動かさない**（マウントのみ） | 切り替えの前 |

## 4. 移行の手順

### フェーズ 1: リポジトリ側の準備（開発機）

6 の「リポジトリ側の修正事項」を行う。特に次は、移行の前に必要。

- `setup.sh` の修正（現状は `docker.io`・`docker-compose` v1 を入れる。`docker compose` が使えない）
- ログのローテーションの設定
- 本番用の `.env` の雛形

### フェーズ 2: Raspberry Pi の準備

1. SSD に **Raspberry Pi OS（64bit、Lite）** を書き込む（D1 の暗号化の決定に従う）。SD カードではなく SSD から起動する。
2. 初期設定: ホスト名、固定の IP（またはルーターの予約）、SSH、タイムゾーン（`Asia/Tokyo`）。
3. **時刻同期（NTP）を確認する。** bisync の競合の判定は「更新時刻が新しいほう」のため、時計のずれは、古いファイルが勝つ原因になる。
4. ユーザーの UID が **1000** であることを確認する（rclone コンテナと Samba の `force user` が UID 1000 を前提にしている）。
5. Docker と Compose プラグインをインストールする。
6. ディレクトリを作る。所有者は UID 1000 にする。
   ```
   /mnt/data/cloud    # クラウドのローカルコピー（SSD）
   /mnt/data/local    # ローカル専用の置き場（SSD）
   /mnt/data/hdd      # バックアップ先（HDD をマウント）
   ```
7. HDD を接続する。`/etc/fstab` には `nofail` を付け、未接続でも起動できるようにする。
8. リポジトリを取得し、`.env` を作る。次の値を設定する。
   - `NODE_ENV=production`
   - `PROD_CH_*`（CLOUD、BACKUP、LOG、STATUS、COCKPIT）
   - `DATA_ROOT`、`DATA_CLOUD`、`DATA_LOCAL`、`DATA_HDD`
   - `SAMBA_USER`、`SAMBA_PASSWORD`、`RESTIC_PASSWORD`
   - `SYNC_INTERVAL`、`BACKUP_TIME`
   - D2 の決定に従い、`DISCORD_TOKEN`、`DISCORD_APPLICATION_ID`
9. `.env` と `rclone/rclone.conf` の権限を `600` にする。

### フェーズ 3: Samba と Bot の起動

1. rclone と restic は、まだ起動しない。Samba と Bot だけを起動する。
2. Bot の起動、チャンネルの警告の有無、スラッシュコマンドの登録（`docs/discord-bot.md`）を確認する。本番では、開発用のボタンが出ない。
3. PC から SMB で接続し、`/mnt/data/local` に書き込めることを確認する。

### フェーズ 4: rclone の認証と、検証用サブフォルダでの検証

1. **Raspberry Pi 上で、`rclone config` により Proton Drive を認証し直す**（2FA コードを使う）。開発機の `rclone.conf` はコピーしない。トークンを共用すると、片方の更新でもう片方が無効になる可能性がある。
2. `RCLONE_REMOTE=protondrive:naspi-test`（検証用のサブフォルダ）に設定し、検証する（[verification-plan.md](verification-plan.md) の A 群・B 群）。
3. 検証が済んだら、`RCLONE_REMOTE=protondrive:` に戻す。

### フェーズ 5: 全体への切り替え

1. ローカルコピーが空の状態から、初回の同期（`--resync --resync-mode newer`）で、クラウドの全体をダウンロードする（`sync.sh` が、基準のファイルがないときに自動で行う）。初回は削除を行わない。
2. 完了後に、クラウドとローカルの内容を `rclone check` で突き合わせる。
3. 以降は、30 分ごとの自動同期に任せる。

### フェーズ 6: バックアップと周辺

1. HDD に restic のリポジトリを `restic init` で初期化する（自動では作らない）。
2. 初回のバックアップを実行する（`/backup`）。
3. 復元テストを実行する（`docs/restic-restore-test.md`）。網羅性の確認も含める。
4. iPad から SMB で接続し、PDF の読み書きと、クラウドへの反映を確認する。

### フェーズ 7: 連続稼働と切り替え完了

1. 1 週間、連続稼働する。例外・遅延・失敗回数を記録する。
2. 問題がなければ、開発環境を停止し、移行を完了とする。

## 5. リスクと対策

| リスク | 影響 | 対策 |
| - | - | - |
| 開発環境と Raspberry Pi の両方が、同じクラウドを書き換える | 意図しない上書き・競合 | 開発環境は同期（bisync）を動かさない。Raspberry Pi の bisync の検証中は、開発環境のマウントも止める |
| 同じ Discord トークンのプロセスが 2 つ動く | 同じ操作に 2 回応答する | D2 のとおり、アプリを分けるか、開発環境の Bot を止める |
| Pi 4 の性能不足（CPU、RAM） | SMB の速度低下、同期やバックアップの遅れ | 性能を測る。基準に満たなければ、Pi 5 を検討する |
| 時計のずれ | 古いファイルが勝つ（bisync の競合） | NTP を有効にし、起動時に確認する |
| 初回同期の容量・時間 | SSD の不足、長時間の通信 | `rclone size` で事前に確認する。Proton のレート制限に注意する |
| 認証切れ・レート制限 | 同期の停止 | 認証エラー検知（実装済み）。Raspberry Pi 上で認証し直す手順を、運用手順に書く |
| `rclone.conf` の平文トークン | 流出すると、クラウドにアクセスされる | 権限 `600`。D1 の暗号化 |
| HDD（exFAT）で restic が失敗する | バックアップできない | 実機で初期化・バックアップ・復元を検証する（D4） |
| 電源断・SSD の障害 | データの破損 | クラウドが正本。ext4 のジャーナルに任せ、定期的に復元テストを行う |
| `dperson/samba` が古い（2020 年のイメージ） | セキュリティ更新が来ない | LAN 内のみで使う。後継のイメージへの置き換えを検討する（6 の No.7） |
| Bot が `docker.sock` を使う | Bot が侵害されると、ホストに影響する | 外部公開しない。Bot のトークンを管理する |

## 6. リポジトリ側の修正事項

| No. | 内容 | 理由 | 優先 |
| - | - | - | - |
| 1 | `setup.sh` を、Docker 公式の手順（`docker-ce` と `docker-compose-plugin`）に変更する | 現状は `docker.io` と `docker-compose`（v1）を入れる。`docker compose` が使えない。`echo "\n..."` も改行されない | 高 |
| 2 | Docker のログのローテーション（`max-size`、`max-file`）を設定する | 30 分ごとの同期ログが溜まり、SSD を圧迫する | 高 |
| 3 | 本番用 `.env` の雛形と、`.env` 項目の説明を整える（`DATA_*` の関係） | 移行時の設定ミスを防ぐ | 中 |
| 4 | bisync の運用手順書を書く（初回、競合、`--force`、致命的エラー後の `--resync`） | 計画書の成果物。現状は未作成 | 高 |
| 5 | HDD のマウント・アンマウント運用の実装（D3） | 憲章は「普段はアンマウント」。現状は常時マウント前提 | 中（D3 次第） |
| 6 | 認証し直す手順を、運用手順書に書く | 認証切れは、実際に発生している | 高 |
| 7 | `dperson/samba` の代替の検討 | 古いイメージ | 低 |
| 8 | Raspberry Pi 向けの性能の記録方法（CPU 温度、メモリ、転送速度） | 検証の合格基準に必要。`/status` は CPU 温度を表示できる | 中 |

## 7. 切り戻し

- クラウド（Proton Drive）が正本のため、Raspberry Pi を作り直しても、データは失われない。ただし、**同期前のローカルの変更は失われうる**ため、切り戻しの前に同期を 1 回実行する。
- 切り戻し先は、開発環境のマウント。`NODE_ENV=development` に戻し、Raspberry Pi の rclone コンテナを止める。
- bisync が致命的エラーで止まった場合は、自動では復旧しない。内容を確認して、手動で `--resync` を実行する。

## 8. 完了の条件

- [verification-plan.md](verification-plan.md) の必須の項目がすべて合格している。
- 1 週間の連続稼働で、重大な障害がない。
- 運用手順書（bisync、認証のやり直し、バックアップ・復元）が揃っている。
- 計画書の進捗（19 章）と、リスク表を更新している。
