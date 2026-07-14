# タスク管理アプリ 仕様書

- 文書名: SPEC.md
- 対象プロダクト: タスク管理Webアプリ
- 作成日: 2026-07-14
- バージョン: v0.1 (初版)

## 1. 目的
個人または小規模チームが、日々のタスクを作成・整理・進捗管理できるWebアプリを提供する。

## 2. 想定ユーザー
- 一般ユーザー: 自分のタスクを登録・更新・完了管理したい利用者
- 管理ユーザー（将来拡張）: チームメンバーのタスク状況を確認したい利用者

## 3. ユースケース一覧（初版）
本仕様書は、まずユースケースの列挙から開始する。

### 3.1 認証・アカウント
- UC-001: ユーザー登録をする
- UC-002: ログインする
- UC-003: ログアウトする
- UC-004: パスワードを再設定する

### 3.2 タスク管理（コア機能）
- UC-101: タスクを作成する
- UC-102: タスク一覧を表示する
- UC-103: タスク詳細を表示する
- UC-104: タスクを編集する
- UC-105: タスクを削除する
- UC-106: タスクを完了にする
- UC-107: 完了済みタスクを未完了に戻す

### 3.3 検索・整理
- UC-201: キーワードでタスクを検索する
- UC-202: 状態（未着手/進行中/完了）で絞り込む
- UC-203: 優先度で絞り込む
- UC-204: 期限で並び替える
- UC-205: 作成日で並び替える

### 3.4 期限・通知
- UC-301: タスクに期限を設定する
- UC-302: 期限切れタスクを判別表示する
- UC-303: 期限が近いタスクを通知する

### 3.5 ラベル・カテゴリ
- UC-401: タスクにラベルを付与する
- UC-402: ラベルでタスクを絞り込む

### 3.6 表示体験
- UC-501: モバイルでも使いやすく表示する
- UC-502: ページ再読込後もタスク状態を保持する

## 4. DB設計

### 4.1 設計方針
- DB種別: SQLite（開発・初期運用）
- ORM: SQLAlchemy 2.x
- 文字コード: UTF-8
- 日時型: UTCで保存（`created_at`/`updated_at`/`completed_at`/`due_at`）
- 論理削除は行わず、初版は物理削除を採用

### 4.2 ER（概念）
- `users` 1 --- n `tasks`
- `tasks` n --- n `labels`（中間テーブル `task_labels`）
- `users` 1 --- n `password_reset_tokens`

### 4.3 テーブル定義

#### 4.3.1 users
- 用途: ログインユーザー情報を保持
- 主なユースケース: UC-001, UC-002, UC-003, UC-004

| カラム名 | 型 | NULL | 制約/備考 |
|---|---|---|---|
| id | INTEGER | NOT NULL | PK, AUTOINCREMENT |
| email | TEXT | NOT NULL | UNIQUE, 255文字以内 |
| password_hash | TEXT | NOT NULL | ハッシュ済みパスワード |
| display_name | TEXT | NOT NULL | 1-100文字 |
| is_active | INTEGER | NOT NULL | 0/1, default 1 |
| created_at | DATETIME | NOT NULL | default CURRENT_TIMESTAMP |
| updated_at | DATETIME | NOT NULL | default CURRENT_TIMESTAMP |

#### 4.3.2 tasks
- 用途: タスク本体
- 主なユースケース: UC-101〜UC-107, UC-201〜UC-205, UC-301〜UC-303

| カラム名 | 型 | NULL | 制約/備考 |
|---|---|---|---|
| id | INTEGER | NOT NULL | PK, AUTOINCREMENT |
| user_id | INTEGER | NOT NULL | FK -> users.id ON DELETE CASCADE |
| title | TEXT | NOT NULL | 1-200文字 |
| description | TEXT | NULL | 詳細説明 |
| status | TEXT | NOT NULL | `todo` / `in_progress` / `done` |
| priority | TEXT | NOT NULL | `low` / `medium` / `high` |
| due_at | DATETIME | NULL | 期限 |
| completed_at | DATETIME | NULL | 完了日時 |
| created_at | DATETIME | NOT NULL | default CURRENT_TIMESTAMP |
| updated_at | DATETIME | NOT NULL | default CURRENT_TIMESTAMP |

#### 4.3.3 labels
- 用途: タスク分類ラベル
- 主なユースケース: UC-401, UC-402

| カラム名 | 型 | NULL | 制約/備考 |
|---|---|---|---|
| id | INTEGER | NOT NULL | PK, AUTOINCREMENT |
| user_id | INTEGER | NOT NULL | FK -> users.id ON DELETE CASCADE |
| name | TEXT | NOT NULL | 1-50文字 |
| color | TEXT | NULL | HEXカラー（例: #1F6FEB） |
| created_at | DATETIME | NOT NULL | default CURRENT_TIMESTAMP |
| updated_at | DATETIME | NOT NULL | default CURRENT_TIMESTAMP |

補足制約:
- `UNIQUE(user_id, name)`（同一ユーザー内でラベル名重複禁止）

#### 4.3.4 task_labels
- 用途: タスクとラベルの多対多関連

| カラム名 | 型 | NULL | 制約/備考 |
|---|---|---|---|
| task_id | INTEGER | NOT NULL | FK -> tasks.id ON DELETE CASCADE |
| label_id | INTEGER | NOT NULL | FK -> labels.id ON DELETE CASCADE |
| created_at | DATETIME | NOT NULL | default CURRENT_TIMESTAMP |

主キー/一意制約:
- `PRIMARY KEY(task_id, label_id)`

#### 4.3.5 password_reset_tokens
- 用途: パスワード再設定トークン管理
- 主なユースケース: UC-004

| カラム名 | 型 | NULL | 制約/備考 |
|---|---|---|---|
| id | INTEGER | NOT NULL | PK, AUTOINCREMENT |
| user_id | INTEGER | NOT NULL | FK -> users.id ON DELETE CASCADE |
| token_hash | TEXT | NOT NULL | UNIQUE |
| expires_at | DATETIME | NOT NULL | 失効日時 |
| used_at | DATETIME | NULL | 使用済み日時 |
| created_at | DATETIME | NOT NULL | default CURRENT_TIMESTAMP |

### 4.4 インデックス設計
- `idx_users_email` on `users(email)`（UNIQUE）
- `idx_tasks_user_status` on `tasks(user_id, status)`
- `idx_tasks_user_due_at` on `tasks(user_id, due_at)`
- `idx_tasks_user_priority` on `tasks(user_id, priority)`
- `idx_tasks_user_created_at` on `tasks(user_id, created_at DESC)`
- `idx_labels_user_name` on `labels(user_id, name)`（UNIQUE）
- `idx_prt_user_expires` on `password_reset_tokens(user_id, expires_at)`

### 4.5 データ整合性ルール
- `status = done` のとき `completed_at IS NOT NULL` を推奨（アプリ層で保証）
- `status != done` のとき `completed_at IS NULL` を推奨（アプリ層で保証）
- `due_at` は過去日も許可（期限超過タスク表現のため）
- `title` 空文字禁止（バリデーション）

### 4.6 SQLAlchemy実装メモ
- SQLiteでは厳密なENUM型を使わず、`String + CHECK制約`で状態値を制御する
- `updated_at` は更新時に自動更新（ORMイベントまたは`onupdate`）
- N+1回避のため、タスク一覧取得時はラベルを`selectinload`で事前取得する

## 5. UI画面とコンポーネント構成

### 5.1 画面一覧
- SCR-001: ログイン画面
- SCR-002: ユーザー登録画面
- SCR-003: パスワード再設定画面
- SCR-101: タスク一覧画面（メイン）
- SCR-102: タスク作成/編集モーダル
- SCR-103: タスク詳細パネル
- SCR-104: ラベル管理画面

### 5.2 画面遷移
- 未認証時: `SCR-001` / `SCR-002` / `SCR-003` のみ遷移可能
- 認証成功後: `SCR-101` をホーム表示
- `SCR-101` から `SCR-102`（新規/編集）と `SCR-103`（詳細）を開く
- `SCR-101` から `SCR-104` に遷移し、完了後に `SCR-101` に戻る

### 5.3 画面仕様（確定）

#### 5.3.1 SCR-001 ログイン画面
- 目的: 既存ユーザーの認証
- 主なUI要素:
	- メールアドレス入力
	- パスワード入力
	- ログインボタン
	- ユーザー登録リンク
	- パスワード再設定リンク
- バリデーション:
	- メール形式チェック
	- 必須項目未入力時は送信不可

#### 5.3.2 SCR-101 タスク一覧画面
- 目的: タスクの閲覧、検索、状態更新、編集導線を集約
- 主なUI要素:
	- ヘッダー（アプリ名、ログアウト）
	- フィルタバー（キーワード、状態、優先度、ラベル）
	- ソート（期限/作成日）
	- 「タスク追加」ボタン
	- タスクリスト（カードまたは行表示）
	- 空状態表示（タスク0件時）
- タスクリスト1件あたりの表示:
	- タイトル
	- ステータス
	- 優先度
	- 期限
	- ラベル
	- 操作（詳細、編集、完了切替、削除）

#### 5.3.3 SCR-102 タスク作成/編集モーダル
- 目的: タスクの新規作成および更新
- 入力項目:
	- タイトル（必須、1-200文字）
	- 説明（任意）
	- ステータス（`todo` / `in_progress` / `done`）
	- 優先度（`low` / `medium` / `high`）
	- 期限（任意）
	- ラベル（複数選択可）
- 操作:
	- 保存
	- キャンセル

#### 5.3.4 SCR-103 タスク詳細パネル
- 目的: タスクの詳細確認とクイック操作
- 表示項目:
	- タイトル/説明
	- ステータス/優先度
	- 期限/完了日時
	- ラベル
	- 作成日時/更新日時
- 操作:
	- 編集
	- 完了/未完了切替
	- 削除

#### 5.3.5 SCR-104 ラベル管理画面
- 目的: ラベルの作成・一覧・削除
- 主なUI要素:
	- ラベル新規作成フォーム（名称、色）
	- ラベル一覧
	- ラベル削除操作
- 制約:
	- 同一ユーザー内でラベル名重複禁止

### 5.4 フロントエンドコンポーネント構成

#### 5.4.1 コンポーネントツリー（MVP）
- `App`
- `AuthLayout`
- `LoginForm`
- `SignupForm`
- `ResetPasswordForm`
- `TaskPage`
- `TaskHeader`
- `TaskFilterBar`
- `TaskSortControl`
- `TaskList`
- `TaskListItem`
- `TaskStatusBadge`
- `PriorityBadge`
- `LabelChips`
- `TaskEditorModal`
- `TaskDetailDrawer`
- `LabelManagementPage`
- `LabelForm`
- `LabelList`
- `ConfirmDialog`
- `ToastProvider`

#### 5.4.2 コンポーネント責務
- `TaskPage`: 一覧表示と検索条件・ソート状態を保持
- `TaskFilterBar`: キーワード/状態/優先度/ラベルの入力を管理
- `TaskList`: フィルタ済みデータの表示、空状態表示
- `TaskListItem`: 単一タスクの描画と行内アクション
- `TaskEditorModal`: 作成/編集フォームと入力バリデーション
- `TaskDetailDrawer`: 詳細表示とクイック操作
- `LabelManagementPage`: ラベルCRUDの画面制御

### 5.5 状態管理方針（確定）
- サーバー状態: API取得データ（タスク/ラベル/認証情報）
- クライアント状態: モーダル開閉、現在のフィルタ条件、ソート条件
- 永続化対象: フィルタ条件とソート条件（ローカル保存）

### 5.6 共通UIルール
- エラーメッセージは項目単位で表示し、送信失敗時はトースト通知
- 削除操作は確認ダイアログを必須化
- モバイル時は詳細を全画面モーダルとして表示
- キーボード操作: `Esc` でモーダル/ドロワーを閉じる

## 6. 次の作業（仕様詳細化）
次フェーズで、上記ユースケースごとに以下を定義する。
- 事前条件
- トリガー
- 基本フロー
- 代替フロー/例外フロー
- 事後条件
- 受け入れ条件
