# B-1. user_quota 整合性チェック・修復機構 — 引継ぎ資料

- 作成日: 2026-10-12
- 対象: `Zaoral-backend-springboot`(実装はすべてこのリポジトリで行う)
- 元の記録: [将来やること.md](./将来やること.md) B-1、[05_DB切り分け.md](./05_DB切り分け.md)の「受容したリスク」「必要となるフォロー」

このファイルは、本項目に着手する(別の)エージェント向けの引継ぎ資料。移行計画(ステップ1〜9)はすでに完了しており、本項目は移行計画のスコープ外として先送りされていたものに着手する。

## 1. 問題の背景

`"User"`(NextAuth/Prisma所有)と`user_quota`(Flyway/Spring Boot所有)は、本来1対1で対応する前提のテーブル。`user_quota`はサインアップ時に次の経路で作られる。

```
初回ログイン
  └─ NextAuthが "User" 行を作成
       └─ events.createUser 発火
            └─ Next.js → Spring Boot: POST /api/internal/user-quota
                 └─ Spring Bootが user_quota 行を作成(ON CONFLICT DO NOTHING で冪等)
```

`events.createUser`は**イベント**であり、ここで例外を投げても`"User"`行はロールバックされない。Spring Boot停止やネットワークエラーでこの呼び出しが失敗すると、**`"User"`はあるが`user_quota`が無いユーザー**が生まれる。現在の実装は「行が無ければ明示的にエラー」という設計(下記3参照)なので、この状態になったユーザーは**残機・プラン関連の全機能が500エラーになり、自動回復しない**。

この挙動は[05](./05_DB切り分け.md)で**意図的に受容したリスク**であり、「遅延生成(無ければその場で作る)」は採用しない方針が明記されている。理由は、遅延生成を許すと「`user_quota`の行数がユーザー数と一致しない」「読み取るすべての経路に生成の規律が必要になる」という、即時生成を選んだ理由そのものが崩れるため。

**本タスクは、この不整合を検知・修復する仕組みを、遅延生成以外の方法で作ること。**

## 2. やること(未決定、最初に決める)

[05](./05_DB切り分け.md)では「定期的な整合性チェック、起動時の一括チェック、修復コマンド等が想定されるが、いずれも未決定」とされている。着手前にどれにするか決めること。参考までに特徴を挙げる。

| 案 | 概要 | 特徴 |
|---|---|---|
| 起動時の一括チェック | Spring Boot起動時に`"User"`と`user_quota`の差分を検知し、ログ出力または自動修復 | 実装は簡単。ただし常時起動しているアプリでないと効かない |
| 定期チェック(`@Scheduled`) | 一定間隔で差分検知・修復を実行 | ローカル検証に向く。本番運用するなら実行間隔や多重起動の考慮が要る |
| 修復コマンド/内部API | 手動で叩いて検知・修復する | 一番低リスク。ただし「気づいて叩く」運用が前提になる |

どれか1つを選んでもよいし、複数を組み合わせてもよい(例: 内部APIで検知・修復ロジックを作り、起動時チェックや定期チェックはそれを呼ぶだけにする)。**この会話の冒頭(着手前)で、ユーザーに方針を確認すること。**

## 3. 現在のコード(前提として必ず読むこと)

### user_quotaが無いときの挙動(現状)

`src/main/java/com/zaoral/backend/service/AccountService.java`:

```java
/**
 * quota行が無いのは "User" と user_quota の1対1が崩れた不整合であり、デフォルト値で補完せず明示的にエラーにする。
 * 補完すると不整合が隠れ、検知できなくなるため(05_DB切り分け.md 参照)。
 */
public UserQuota findQuota(String userId) {
    return userQuotaRepository.findById(userId).orElseThrow(() -> {
        log.error("user_quota が存在しません: userId={}", userId);
        return new ApiException(HttpStatus.INTERNAL_SERVER_ERROR, "ユーザーのクォータ情報が見つかりません");
    });
}
```

**この「明示的にエラーにする」設計は変えない**こと。検知・修復の仕組みは、この挙動を隠すのではなく、この状態が起きたことを別の経路で把握・解消するためのもの。

### quota初期化(冪等)

`src/main/java/com/zaoral/backend/repository/UserQuotaRepository.java`:

```java
@Modifying
@Query(value = "INSERT INTO user_quota (user_id) VALUES (:userId) ON CONFLICT (user_id) DO NOTHING",
        nativeQuery = true)
int insertIfAbsent(@Param("userId") String userId);
```

修復処理でも、この`insertIfAbsent`(または同等のON CONFLICT DO NOTHING)をそのまま使ってよい。新しいロジックが必要なのは「どの`userId`が不整合を起こしているかを見つける」部分。

### 不整合を検知するクエリの材料

`"User"`はSpring Boot側で`UserRefRepository`(`JpaRepository<UserRef, String>`)から`findAll()`できる。`user_quota`も`UserQuotaRepository`から同様。差分は「`"User"`に存在するが`user_quota`に存在しないID」で、SQLなら以下に相当する。

```sql
SELECT u.id FROM "User" u
LEFT JOIN user_quota q ON q.user_id = u.id
WHERE q.user_id IS NULL;
```

ネイティブクエリで書くのが素直(JPQLは2テーブルのLEFT JOIN+IS NULLとの相性が悪いため)。`UserQuotaRepository`に`@Query(nativeQuery = true)`で追加する形が、既存の`insertIfAbsent`と書き方が揃う。

### 関連するファイル一覧

| ファイル | 役割 |
|---|---|
| `entity/UserRef.java` | `"User"`の読み取り専用エンティティ(`@Immutable`) |
| `entity/UserQuota.java` | `user_quota`のエンティティ |
| `repository/UserRefRepository.java` | `"User"`読み取り |
| `repository/UserQuotaRepository.java` | `user_quota`読み書き。`insertIfAbsent`あり |
| `service/AccountService.java` | quota関連の業務ロジック。`initializeQuota`・`findQuota`がある |
| `controller/InternalController.java` | `POST /api/internal/user-quota`。Next.jsサーバーサイドからのみ呼ばれる内部用 |
| `exception/ApiException.java` / `GlobalExceptionHandler.java` | `{"error": "..."}`形式でエラーを返す共通処理。`@FailureMessage`でコントローラごとに500時のメッセージを指定できる |
| `src/main/resources/db/migration/V1__init.sql` | 現在の唯一のマイグレーション。新しいテーブル・カラムが要るなら`V2__xxx.sql`を追加する |

## 4. 開発環境(このセッションで確立した前提)

- **バックエンドはPleiades All in One Eclipseで開発・実行する。CLIの`mvn`は使わない**(m2eが内蔵Mavenを使う)。詳細は[07](./07_バックエンド実装.md)の「必要な環境」参照
- プロジェクトの取り込みは「既存Mavenプロジェクトのインポート」。普通のインポートでは`pom.xml`が読まれない
- 実行は右クリック→「実行」→「Spring Boot アプリケーション」(Spring Toolsの機能が同梱されている。詳細や注意点は[07](./07_バックエンド実装.md)参照)
- ローカルPostgresは`Zaoral-frontend`の`docker-compose.dev.yml`が起動するコンテナ(`zaoral-frontend-db-1`)。`application.yml`の既定値で接続できる(環境変数の設定は不要)
- `pom.xml`に依存を追加した場合のみ、Eclipseで「Maven→プロジェクトの更新」が必要。コード変更だけなら再起動で反映される

## 5. 動作確認の材料

- **Postmanコレクション**: `Zaoral-backend-springboot/postman/zaoral-backend.postman_collection.json`。quota初期化・取得、異常系(401/404)のリクエストが揃っている
- **テストユーザー作成**: `Zaoral-backend-springboot/db/create_test_user.sql`(冪等)。`"User"`に`test-user-1`を作る。`db/README.md`に実行方法あり
- **不整合を意図的に作る方法**: テストユーザー作成後、`user_quota`を手動で消せば再現できる

  ```sql
  DELETE FROM user_quota WHERE user_id = 'test-user-1';
  ```

  この状態で`GET /api/account/quota`を叩くと、現状は500(「ユーザーのクォータ情報が見つかりません」)になる。これが直す対象の状態。

## 6. 完了後にやること

- 実装したら、`AccountService.findQuota`の500挙動は変えていないことを確認する(検知・修復は別経路であるべき)
- ドキュメントの更新(このセッションでの更新漏れが多いので、作業後に必ず反映する)
  - [05_DB切り分け.md](./05_DB切り分け.md)の「必要となるフォロー(未設計)」を実施済みの内容に更新
  - [将来やること.md](./将来やること.md) B-1を完了または進捗状況に更新
  - 新しいFlywayマイグレーション(`V2`等)を追加した場合は、[07_バックエンド実装.md](./07_バックエンド実装.md)のテーブル一覧やエンティティ表も実態に合わせる
- 本ファイル(`B1_quota整合性チェック_引継ぎ.md`)は、作業完了後は削除してよい(内容は更新後の05・将来やること.mdに引き継がれる想定)

## 7. やらないこと(念のため)

- **遅延生成は採用しない**。「`findQuota`で見つからなければその場で作る」という実装は、[05](./05_DB切り分け.md)の受容したリスクの前提を崩すため不可
- `"User"`テーブルへの書き込み・マイグレーションは行わない(Spring Bootは読み取り専用で参照する方針。[05](./05_DB切り分け.md)参照)
