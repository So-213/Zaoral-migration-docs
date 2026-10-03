# 06. AWS関連の切り分け

- 決定日: 2026-10-03
- 前提: [移行計画書.md](./移行計画書.md)のゴール定義(ローカルで動作することをもって完了)

## 現状

### 使用しているAWSサービスはS3のみ

`terraform/`の定義は画像保存用バケット`project_pictures`とその付帯設定に限られる。Cognito/Lambda/SES等は使用していない。

- バケット本体、バージョニング、パブリック読み取りポリシー、CORS、31日ライフサイクル削除
- アップロード用の`aws_iam_user`(`runtime_user`)、IAMポリシー、**長期有効なアクセスキー**

### コードからのアクセスは`src/lib/s3.ts`のみ

| 関数 | 利用箇所 | 状態 |
|---|---|---|
| `uploadImageToS3` | `src/app/api/projects/upload/route.ts` | 早期returnで無効化済み。実際には呼ばれない |
| `getS3PublicUrl` | `src/lib/dashboard-config.ts` | picture型プロジェクトの表示URL生成。到達不能(下記) |
| `deleteImageFromS3` | なし | project-service.tsのpictureハンドラがコメントアウト済み |

環境変数は`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_REGION` / `S3_BUCKET_NAME`。

### picture型は完全に停止している

`src/lib/config.ts`の`VALID_PROJECT_TYPES`が`['message']`のみであるため、**picture型プロジェクトは作成できない**。アップロードエンドポイントも400を返す。したがってS3は現在まったく使われていない。

## 決定事項

### 1. 今回はS3を移植しない

picture型が停止中であり、ゴールがローカル動作であるため、S3関連の移植は行わない。`src/lib/s3.ts`は`Zaoral-frontend`に残置する(picture型再開時にSpring Bootへ移植する際の参照元とする)。

AWS資格情報もローカルでは不要(`getS3Client()`が呼ばれないため未設定でも動作する)。

### 2. 再開時の所有はSpring Bootとする

picture型を再開する場合、S3アクセスは**Spring Bootが持つ**。

理由:
- `projects_picture`テーブルはFlyway/Spring Boot所有([05](./05_DB切り分け.md))。画像の保存・削除はそのテーブルのライフサイクルと一体の業務処理である
- Next.jsにAWS資格情報を持たせない。認証境界とクラウド資格情報を同じ場所に置かない

### 3. 公開URLはAPIレスポンスに含めて返す

`getS3PublicUrl`相当の処理もSpring Boot側に置き、**組み立て済みのURLをレスポンスに含める**。フロントエンドがバケット名やS3のURL構造を知る必要をなくす。

現在は`dashboard-config.ts`がフロント側でURLを組み立てているが、この方式は踏襲しない。

### 4. terraformは対象外

デプロイがスコープ外であるため、terraformの見直しは行わない。

## 申し送り

- **picture型を再開する場合に必要な作業**: Spring BootへのS3アクセス移植、MinIO/LocalStackのローカル構築、`VALID_PROJECT_TYPES`への`picture`追加、`project-service.ts`のpictureハンドラ復活(Spring Boot側で再実装)。
- 現在のIAM構成は**長期有効なアクセスキーを環境変数で保持する方式**。将来デプロイする際はインスタンスロール等の一時credentialへ切り替えることが望ましい(スコープ外)。
- `src/lib/s3.ts`には調査時に不審なデバッグコード(AWS環境変数をローカルエンドポイントへ送信)が含まれていたが、削除済み([01_調査.md](./01_調査.md)参照)。
