# Batch 2 強制確認バッチ

## 概要

毎月実行する処理
未確認項目を強制的に確認済に変更する

## シーケンス図

```mermaid
sequenceDiagram
    participant Scheduler as Cron
    participant Batch as BatchJob
    participant DB as Database

    Scheduler->>Batch: バッチ処理起動 (20日 18:00)

    Batch->>DB: アクティブなグループ一覧取得
    DB-->>Batch: アクティブなグループ一覧

    loop アクティブなグループ一覧
        Batch->>DB: 前月分の集計の実行有無
        DB-->>Batch: 実行済 / 未実行

        alt 実行済
            Batch-->>Scheduler: 終了 (実行済)
        else 未実行
            Batch->>DB: 処理対象について未確認項目の個数を取得
            DB-->>Batch: 未確認項目の個数

            loop 未確認項目の個数 > 0
                Batch-->>DB: 強制的に確認済に変更(comfirmed=true, forced=true)
                DB-->>Batch: 登録済
            end

            Batch-->>Scheduler: 終了 (完了)
        end
    end
```

## cron サンプル（例）

- 毎月16〜20日の20:00に実行（Linux cron形式）:
  `0 20 16-20 * * /path/to/run-batch.sh`
- 20日のみ実行（強制確認を伴う実行を別コマンドで分けたい場合）:
  `0 20 20 * * /path/to/run-batch-with-force-confirm.sh`

## PDF出力について

- 支出の合計金額, ユーザー1への請求額, ユーザー2への請求額を計算する
- 各列に表示するのは、支出額, 支出割合を計算した請求額
- [出力サンプル](../samples/invoice_2026-01.pdf)を参考に計算する。

## DB参照

- テーブル定義・関連情報は [db.md](db.md) を参照
