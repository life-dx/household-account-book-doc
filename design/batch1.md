# Batch 1 月次割り勘集計バッチ

## 概要

毎月実行する処理
前月の収支状況を集計しPDFを作成, 作成完了したらLINE通知を送信する
未確認項目があって集計できない場合もLINE通知を送信する

## シーケンス図

```mermaid
sequenceDiagram
    participant Scheduler as Cron
    participant Batch as BatchJob
    participant DB as Database
    participant PDF as PDFGenerator
    participant LINE as LINEService

    Scheduler->>Batch: バッチ処理起動 (16日-20日)

%% グループ一覧取得, ループで以下処理を実行。←そういう風にかけ

    Batch->>DB: 前月分の集計の実行有無
    DB-->>Batch: 実行済 / 未実行

    alt 実行済
        Batch-->>Scheduler: 終了 (実行済)
    else 未実行
        Batch->>DB: 処理対象について未確認項目の個数を取得
        DB-->>Batch: 未確認項目の個数

        alt 未確認項目の個数 > 0
            Batch-->>Batch: 処理を中止 (未確認項目が存在する)
            Batch->>LINE: 未確認項目の確認ユーザーに確認を促すメッセージを送る
            Batch-->>Scheduler: 終了 (中止)
        end

        Batch->>DB: 前月1か月のデータを取得
        DB-->>Batch: 前月1か月のデータ

        Batch-->>Batch: 集計実行

        Batch->>DB: 入力必須項目の登録有無を取得
        DB-->>Batch: 入力必須項目の登録有無

        Batch->>PDF: PDF生成(集計結果, 入力必須項目登録状況)
        alt 入力必須項目が未登録
            PDF->>PDF: PDF内に必須項目がない旨の警告を表示する
        end
        PDF-->>Batch: file_path, file_hash

        Batch->>DB: 集計履歴保存(aggregate_histories, file_path, file_hash)
        DB-->>Batch: 保存OK

        Batch->>LINE: PDFへのリンクを含むメッセージを送信(グループのユーザーへ)
        LINE-->>Batch: 送信結果

        Batch-->>Scheduler: 終了 (完了)
    end
```

## 未確認項目がある場合のメッセージサンプル

```txt
次の項目が未確認です。
確認お願いします。
- xxxx (https://example.com/confirm/xxxx)
- yyyy (https://example.com/confirm/yyyy)
- zzzz (https://example.com/confirm/zzzz)

確認: https://example.com/confirm
閲覧: https://example.com/list
```

## 正常に生成できた場合のメッセージサンプル

```txt
〇月の集計が完了しました。
合計支出額: 〇円
あなたの支払額: 〇円

集計結果詳細: https://example.com/aggregate/files/path-to/file.pdf
```

## 入力必須項目が未登録だったが、集計が完了した場合のメッセージサンプル

```txt
〇月の集計が完了しました。
合計支出額: 〇円
あなたの支払額: 〇円

* 未入力項目がありました。修正する場合はこちら (https://example.com/register)

集計結果詳細: https://example.com/aggregate/files/path-to/file.pdf
```

## cron サンプル（例）

- 毎月16〜20日の20:00に実行（Linux cron形式）:
  `0 20 16-20 * * /path/to/run-batch.sh`
- 20日のみ実行（強制確認を伴う実行を別コマンドで分けたい場合）:
  `0 20 20 * * /path/to/run-batch-with-force-confirm.sh`

## PDF出力について

- 支出の合計金額, ユーザー1への請求額, ユーザー2への請求額を計算する
- 各列に表示するのは、支出額, 支出割合を計算した請求額
- 強制的に確認済に変更されていた場合はその旨も表示する
- [出力サンプル](../samples/invoice_2026-01.pdf)を参考に計算する。

## DB参照

- テーブル定義・関連情報は [db.md](db.md) を参照
