# 🌰 どんぐり郵便（Donguri Postal Service）

> 大切な気持ちをポストカードに込めて、希望する日時に届ける予約メール型ポストカードサービス

---

## 紹介

**どんぐり郵便**は、日本語の「どんぐり」をコンセプトにした、温かみのあるメール型ポストカードサービスです。  
ユーザーはポストカードのテンプレートを選択し、内容を作成した後、送信日時を予約します。  
予約した日時になると、受信者へメールが自動送信され、受信者はリンクからBGM付きのポストカードを確認できます。

---

## 主な機能

| 機能 | 説明 |
|---|---|
| **ポストカード予約送信** | 日付・時刻を指定してメール型ポストカードを予約し、Quartzスケジューラが自動送信 |
| **ポストカード保管箱** | 自分が送信したポストカードの一覧照会および内容確認 |
| **テンプレート選択** | 管理者が登録したBASEテンプレート + QRスキャンで解放されるADDEDテンプレート |
| **QRコード解放** | QR画像をアップロードすると、サーバー側でデコード後、テンプレートを解放 |
| **幸運のどんぐり（おみくじ）** | 1日1回、運勢を引く機能（BAD ～ BIG_GOOD の5段階） |
| **メール認証付き会員登録** | 6桁の認証コードをメールで送信し、会員登録を実施 |
| **BGM再生** | ポストカード作成時および受信時に、背景音楽の選択・再生が可能 |
| **管理者パネル** | テンプレート登録・修正・削除、QRテンプレート生成、問い合わせ管理 |

---

## 技術スタック

| 分類 | 技術 |
|---|---|
| **Backend** | Java 8, Servlet 4.0, JSP, JSTL |
| **Build** | Gradle（WAR）, Apache Tomcat |
| **Database** | Oracle DB, JDBC（ojdbc8）, P6Spy（クエリログ）, Apache Commons DBCP2 |
| **Storage** | AWS S3（ap-northeast-2） |
| **Scheduler** | Quartz 2.3.2 |
| **Mail** | JavaMail（Gmail SMTP） |
| **QR** | ZXing（Google） |
| **その他** | Lombok, Logback, GSON |

---

## はじめに

### 事前要件

- JDK 8+
- Apache Tomcat 9+
- Oracle Database
- AWS S3 バケット
- Gmailアプリパスワード（2段階認証の有効化が必要）

### 1. リポジトリのクローン

```bash
git clone <repository-url>
cd donguri
```

### 2. 環境変数ファイルの作成

`src/main/resources/.env` ファイルを作成し、以下の内容を入力します。

```env
# Oracle DB接続情報
DB_URL=jdbc:oracle:thin:@//ホスト:1521/サービス名
DB_USER=アカウント名
DB_PASSWORD=パスワード

# Gmail SMTP設定
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_EMAIL=送信元メールアドレス@gmail.com
SMTP_PASSWORD=Gmailアプリパスワード

# サービスベースURL（メール内リンク生成に使用）
BASE_URL=http://localhost:8080/donguri

# AWS S3
AWS_S3_ACCESS_KEY=アクセスキー
AWS_S3_SECRET_KEY=シークレットキー
AWS_S3_BUCKET_NAME=バケット名

# サーバー再起動時に未送信メールを即時送信するかどうか（開発環境では true 推奨）
SKIP_EMAIL_SEND_ON_BOOT=true
```

### 3. データベース初期化

`src/main/java/com/c1/donguri/sql/` ディレクトリ内のSQLファイルを順番に実行します。

```text
1. create.sql  → テーブルおよびトリガー作成
2. insert.sql  → 初期データ挿入（おみくじ運勢など）
```

### 4. ビルドおよびデプロイ

```bash
# WARファイルのビルド
./gradlew war

# build/libs/donguri-1.0-SNAPSHOT.war を Tomcat webapps にデプロイ
```

**Tomcat JVMオプション設定必須**（文字化け防止）：

```text
-Dfile.encoding=UTF-8
```

---

## プロジェクト構成

```text
src/main/java/com/c1/donguri/
├── main/           # メインページ、サーバー起動・終了リスナー
├── user/           # 会員登録、ログイン、マイページ、メール認証
├── template/       # ポストカードテンプレート照会・管理、QR生成・解放
├── reservation/    # ポストカード予約作成・照会・修正・削除
├── post/           # 受信ポストカードビュー、送信済み一覧照会
├── omikuji/        # おみくじ機能
├── scheduler/      # Quartzメール予約ジョブ管理
├── inquiry/        # 問い合わせ（ユーザー・管理者）
└── util/           # DBManager, S3Uploader, EmailSend, QRGenerator, EnvLoader

src/main/webapp/
├── main.jsp        # 共通レイアウト（ヘッダー + content領域）
├── home.jsp        # メインホーム画面
├── jsp/            # 各機能別JSPビュー
├── css/            # 機能別スタイルシート
├── js/             # クライアントスクリプト
├── image/          # UIアセット
└── bgm/            # 背景音楽ファイル
```

---

## 管理者アカウント

DBの `users` テーブルで `roll = 'ADMIN'` に設定すると、管理者権限が付与されます。  
JSPでは `nickname = '관리자'` であるかどうかにより、管理者メニューの表示を制御します。

管理者専用メニュー：
- `template-list-admin` — 全テンプレート管理
- `template-create-admin` — 新規テンプレート登録（BASE / QR解放型）
- `inquiry-admin` — 問い合わせ一覧照会

---

## データベースERD（概要）

```text
users ──────────────────────── reservation
  │                                │
  │ (user_template)          email_content
  │                                │
template ←──── user_template   template

omikuji    inquiry    send_log ── reservation
```

| テーブル | 説明 |
|---|---|
| `users` | 会員情報、`is_deleted='Y'` による論理削除 |
| `template` | ポストカードテンプレート（`BASE` / `ADDED`） |
| `email_content` | ポストカード本文（タイトル、内容、カバー画像、BGM） |
| `reservation` | 送信予約（`is_done='N'` → `Y` で完了処理） |
| `send_log` | 送信成功・失敗履歴 |
| `user_template` | QRで解放されたテンプレートとユーザーの関係 |
| `omikuji` | 運勢メッセージプール |
| `inquiry` | 問い合わせ受付履歴 |

---

## サーバー再起動時の動作

サーバーが起動すると、`AppStartupListener` が未完了（`is_done='N'`）の予約を全件照会します。

- **すでに過ぎた予約** → 即時メール送信（`.env` の `SKIP_EMAIL_SEND_ON_BOOT=true` の場合は省略）
- **まだ到来していない予約** → Quartzジョブに登録し、予約時刻に自動送信
