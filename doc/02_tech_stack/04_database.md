# データベース・ストレージ選定書

本ドキュメントは、お米管理Webアプリ（`rice-management`）におけるデータベース選定、スキーマ管理手法（Flyway）、Docker Compose による永続化、およびバックアップ方針を定義したものです。

---

## 1. データベース選定サマリー

| 項目 | 採用技術 | 選定理由 |
| :--- | :--- | :--- |
| **DBMS** | **MySQL 8.x** | 世界標準のRDBMS。トランザクション（ACID特性）による確実なデータ整合性、jOOQおよびFlywayとの成熟した連携エコシステム。 |
| **マイグレーション** | **Flyway** | SQLスクリプト（`V1__init.sql` 等）による明瞭なバージョン管理。CI/CDやローカル起動時に完全自動適用可能。 |
| **実行環境** | **Docker Compose (MySQLコンテナ)** | ホスト環境を汚さず、誰の開発機でも同一のDB環境を即座に再現。ボリュームマウントによる確実なデータ永続化。 |
| **文字コード** | `utf8mb4` | 絵文字や特殊な人名漢字（親戚の名前等）に対応。 |
| **照合順序** | `utf8mb4_0900_ai_ci` | 日本語の濁点・半濁点、大文字小文字を適切に扱う標準Collation。 |

---

## 2. Docker Compose によるコンテナ & 永続化設計

DBコンテナ停止時や再ビルド時にデータが消失しないよう、Docker Named Volume を使用して永続化します。

```yaml
# docker-compose.yml 抜粋イメージ
services:
  rice-db:
    image: mysql:8.4
    container_name: rice-management-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: rice_management
      MYSQL_USER: rice_user
      MYSQL_PASSWORD: rice_password
      TZ: Asia/Tokyo
    ports:
      - "3306:3306"
    volumes:
      - rice-db-data:/var/lib/mysql
      - ./docker/mysql/init:/docker-entrypoint-initdb.d
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_0900_ai_ci
      - --default-time-zone=+09:00

volumes:
  rice-db-data:
    driver: local
```

---

## 3. Flyway によるスキーマバージョン管理 & jOOQ連携

### 3.1 ディレクトリ構成
マイグレーションSQLファイルは Spring Boot の標準ディレクトリに配置します。

```text
src/main/resources/db/migration/
├── V1__create_initial_schema.sql     # 在庫、履歴、マスタテーブル定義
├── V2__insert_initial_masters.sql    # 家族メンバー、贈答先初期データ投入
└── V3__add_...sql                    # 今後のスキーマ変更
```

### 3.2 テーブル設計の基本方針
* **論理削除フラグ (`is_deleted`)**:
  * 誤操作による削除を取り消せるよう、マスタテーブルおよび出納履歴テーブルには `is_deleted TINYINT(1) DEFAULT 0` を付与。
* **監査カラム**:
  * すべてのテーブルに `created_at`（登録日時）、`updated_at`（更新日時）を共通定義。
* **外部キー制約**:
  * 履歴テーブルから家族マスタ（`member_id`）および贈答先マスタ（`recipient_id`）へ外部キー制約を張り、データの整合性を担保。

---

## 4. バックアップ & リカバリ方針

家庭内の貴重な記録を守るため、軽量かつ確実なバックアップ体制を構築します。

1. **データ規模の特性**:
   * 年間 50〜100 レコード程度、10年でも1,000レコード未満（データ容量は数十MB未満）。
2. **自動バックアップスクリプト**:
   * 定期的に `docker exec` 経由で `mysqldump` を実行し、日付付きSQLファイル（`.sql.gz`）を出力するシェルスクリプトを用意。
3. **リストア確認**:
   * バックアップファイルからワンコマンドでローカルDBに復元できるスクリプトを `docker/backup/` 配下に整備。
