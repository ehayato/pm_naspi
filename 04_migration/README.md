# Raspberry Pi への移行と本番モードの検証

開発環境（x86_64、FUSE マウント）から、Raspberry Pi 上の本番運用（rclone bisync）へ移すための計画をまとめる。
計画書（[plan.md](../02_plan/plan.md)）の「rclone移植」「バックアップ実装」「テスト・安定化」の残りの作業にあたる。

| 文書 | 内容 |
| - | - |
| [migration-plan.md](migration-plan.md) | 移行計画（前提、決めること、手順、切り戻し、リポジトリ側の修正事項） |
| [verification-plan.md](verification-plan.md) | 本番モード（bisync）と Raspberry Pi の検証計画（シナリオ、合格基準、記録） |

## 全体の流れ

```
0. 決めること  →  1. リポジトリ側の準備  →  2. Raspberry Pi の準備  →  3. 検証用サブフォルダで検証
   →  4. 全体へ切り替え  →  5. 周辺（バックアップ・Samba・iPad）  →  6. 1週間の連続稼働  →  7. 切り替え完了
```

- 検証は、**クラウドの全体ではなく、検証用のサブフォルダで始める**。本番データを巻き込まずに、削除の上限や競合を試せる。
- 開発環境は、移行が終わるまで切り戻し先として残す。
- 結果は、[03_progress/test-log.md](../03_progress/test-log.md) と [incident-log.md](../03_progress/incident-log.md) に記録する。
