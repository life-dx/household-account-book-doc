# データベース定義

## 概要

割り勘アプリ V2用のデータベース定義(参考)

## テーブル一覧

### ユーザー系

#### ユーザーマスタ

- ユーザーのログイン情報を保存する。

| 列名        | データ型     | 概要                                      |
|-------------|--------------|-------------------------------------------|
| user_id     | uuid         | Primary Key, システム内部で扱うユーザーID |
| google_sub  | varchar(255) | Unique, Googleアカウントのユーザー識別子  |
| email       | varchar(255) | メールアドレス                            |
| name        | varchar(255) | ユーザー名                                |
| picture_url | TEXT         | アイコン画像URL                           |
| created_at  | timestamp    | 作成日                                    |
| updated_at  | timestamp    | 更新日                                    |

#### グループマスタ

- 割り勘グループの一覧を保存する

| 列名        | データ型     | 概要                                            |
|-------------|--------------|-------------------------------------------------|
| group_id    | uuid         | Primary Key, システム内部で使うグループの識別子 |
| group_name  | varchar(255) | グループ名称                                    |
| description | text         | グループ説明                                    |
| user_id_1   | uuid         | foreign key, ユーザーマスタのユーザーID         |
| user_id_2   | uuid         | foreign key, ユーザーマスタのユーザーID         |
| active      | bool         | true: グループが有効 = 集計などを実行する       |
| created_at  | timestamp    | 作成日                                          |

#### ユーザー通知先マスタ

- 通知先一覧。ラインしかないのでユーザーマスタと統合してもいいかもしれない。

| 列名         | データ型    | 概要                                       |
|--------------|-------------|--------------------------------------------|
| user_id      | uuid        | foreign key, ユーザーマスタのユーザーID    |
| line_user_id | varchar(50) | ライン通知用ユーザーID(ラインのIDではない) |
| updated_at   | timestamp   | 更新日                                     |

### 入力系

#### 支出一覧

- 支出情報を保存する

| 列名          | データ型     | 概要                                                                  |
|---------------|--------------|-----------------------------------------------------------------------|
| group_id      | uuid         | Foreign Key, グループの識別子                                         |
| expense_id    | uuid         | Primary Key, システム内部で使う支出の識別子                           |
| expense_date  | date         | 支払日                                                                |
| expense_title | varchar(255) | 件名                                                                  |
| invoice_rate  | int          | 相手に請求する割合 (%)                                                |
| expensed_by   | uuid         | foreign key, 支払った人, ユーザーマスタのユーザーID                   |
| created_by    | uuid         | foreign key, 作成者, 通常支払った人と同じ。ユーザーマスタのユーザーID |
| created_at    | timestamp    | 作成日, システム内部管理                                              |
| updated_at    | timestamp    | 更新日                                                                |
| fixed         | bool         | 確定フラグ, trueの時は編集不可。(アプリ側制御)                        |
| file_path     | varchar(255) | 保存先ファイルパス                                                    |
| file_hash     | varchar(255) | ファイルハッシュ(sha256)                                              |

#### カテゴリマスタ

- カテゴリ情報を保存する

| 列名          | データ型    | 概要                                    |
|---------------|-------------|-----------------------------------------|
| category_id   | varchar(50) | Primary Key, カテゴリの識別子。FOODとか |
| display_name  | varchar(50) | 表示名                                  |
| display_color | varchar(10) | カラーコード                            |

#### 品目マスタ

- 品目情報を保存する。入力補助用。

| 列名        | データ型     | 概要                                        |
|-------------|--------------|---------------------------------------------|
| item_id     | uuid         | Primary Key, システム内部で使う品目の識別子 |
| item_name   | varchar(255) | 品目名 (not null)                           |
| price       | int          | 単価 (nullable)                             |
| category_id | varchar(50)  | Foreign Key, カテゴリマスタのカテゴリ識別子 |

#### 品目一覧

- 支出情報に紐づく品目一覧を保存する。

| 列名        | データ型     | 概要                                           |
|-------------|--------------|------------------------------------------------|
| expense_id  | uuid         | Foreign Key, 支出一覧にある支出の識別子        |
| item_id     | uuid         | Primary Key, システム内部で使う支出の識別子    |
| item_name   | varchar(255) | 品目名 (not null)                              |
| price       | int          | 単価                                           |
| category_id | varchar(50)  | Foreign Key, カテゴリマスタのカテゴリ識別子    |
| quantity    | int          | 数量                                           |
| fixed       | bool         | 確定フラグ, trueの時は編集不可。(アプリ側制御) |
| rule_id     | uuid         | Foreign Key, ルールを識別するID, NULL許容      |
| created_at  | timestamp    | 作成日, システム内部管理                       |
| updated_at  | timestamp    | 更新日                                         |

#### 確認履歴

- 支出の確認履歴を保持する。expense_idのtimestamp最新をとると状況がわかるので、それをもとに判断する。
- 更新禁止

| 列名       | データ型  | 概要                                    |
|------------|-----------|-----------------------------------------|
| expense_id | uuid      | Foreign Key, 支出一覧にある支出の識別子 |
| confirmed  | bool      | 確認したかどうか                        |
| created_at | timestamp | 確認日時                                |
| updated_by | uuid      | Foregin Key, ユーザーID                 |
| forced     | bool      | 強制的に確認されたかどうか              |

#### 割り勘集計履歴

- 割り勘の集計履歴を保持する。その時のsnapshotである。
- 更新禁止

| 列名         | データ型     | 概要                          |
|--------------|--------------|-------------------------------|
| aggregate_id | uuid         | Primary Key, 集計を特定するID |
| created_at   | timestamp    | 集計日時                      |
| file_path    | varchar(255) | 集計後ファイルのパス          |
| file_hash    | varchar(255) | ファイルハッシュ(sha256)      |

#### 入力ルール

- 必須入力, 推奨入力の設定を保持する。システム内部で管理する。
- 品目を入力するときにそのルールのデータであることを明示する。

| 列名          | データ型     | 概要                                                         |
|---------------|--------------|--------------------------------------------------------------|
| group_id      | uuid         | Foreign Key, グループの識別子。ルールを適用するグループ      |
| rule_id       | uuid         | Primary Key, ルールの識別子                                  |
| rule_name     | varchar(255) | ルールを識別するための表示名                                 |
| behavior_code | smallint     | 月次処理における未入力時の扱い。1: 集計を禁ず, 2: 警告を発す |
| notify_to     | uuid         | Foreign Key,  通知先ユーザー                                 |
| created_at    | timestamp    | 作成日時                                                     |
| updated_at    | timestamp    | 更新日時                                                     |
| updated_by    | uuid         | 更新者                                                       |
