---
title: "Athena VACUUMクエリによるS3オブジェクト削除時のレスポンス確認"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "Athena", "iceberg"]
published: true
---

https://docs.aws.amazon.com/ja_jp/athena/latest/ug/vacuum-statement.html

AthenaでVACUUMクエリを実行する際の案内として

> VACUUM がデータファイルを削除できるようにするには、クエリ実行ロールに、Iceberg テーブル、メタデータ、スナップショット、およびデータファイルが配置されているバケットに対する s3:DeleteObject 許可が必要です。許可が存在しない場合、VACUUM クエリは成功しますが、ファイルは削除されません。

という項目があります。

VACUUMクエリ自体はオブジェクトを削除する権限がなくても成功するようです。
ホンマか！？と思ったので実際に試してみました。

## 結果


- VACUUMクエリ実行時に削除対象のオブジェクトがあって`s3:DeleteObject`が付与されていない場合はちゃんとエラーが返る
- 削除対象のオブジェクトが無い場合は成功する

という形でした。ちゃんと削除対象のオブジェクトがある場合はエラーになりました。助かりますね。


### VACUUMによるオブジェクト削除の確認

まずは適切にVACUUMクエリでオブジェクトが削除できる環境を作ります。


テーブルを作ります。
すぐに期限切れとなるように保持するスナップショットの数や経過時間をできるだけ小さくします。

https://docs.aws.amazon.com/ja_jp/athena/latest/ug/create-table-as.html#ctas-table-properties

```sql:
CREATE TABLE default.test_iceberg
WITH (
  table_type = 'ICEBERG',
  is_external = false,
  location = 's3://quark-sandbox/test_iceberg/',
  format = 'parquet',
  vacuum_min_snapshots_to_keep = 1,
  vacuum_max_snapshot_age_seconds = 60
) AS
SELECT *
FROM (
  VALUES
    (1, 'alice'),
    (2, 'bob')
) AS t (id, name);

```

![](/images/athena_vacuum_object_delete_test/2025-08-23_061059.png)
*テーブルはこんな感じ*

dataフォルダ下のオブジェクト数とサイズを確認します。

```bash:
$ aws s3 ls s3://quark-sandbox/test_iceberg/data/ --human --recursive --sum
2025-08-23 06:10:33  345 Bytes test_iceberg/data/20250822_211032_00223_uzhbd-bcdccbb0-a332-47a1-a767-c82f2eba88e7.parquet

Total Objects: 1
   Total Size: 345 Bytes
```



以下データをMERGE INTOして一部行はUPDATE,一部行はINSERTされるクエリを発行します。

```sql:
MERGE INTO default.test_iceberg AS t
USING (
    VALUES
        (1, 'carol'),
        (2, 'dave'),
        (3, 'mark')
    ) AS s(id, name)
ON t.id = s.id
WHEN MATCHED THEN UPDATE SET name = s.name
WHEN NOT MATCHED THEN INSERT (id, name) VALUES (s.id, s.name);

```

![](/images/athena_vacuum_object_delete_test/2025-08-23_061608.png)
*データが更新&挿入された*

再度S3のdataオブジェクトを確認します。
新しいオブジェクトが2つ入っています。

```
$ aws s3 ls s3://quark-sandbox/test_iceberg/data/ --human --recursive --sum
2025-08-23 06:10:33  345 Bytes test_iceberg/data/20250822_211032_00223_uzhbd-bcdccbb0-a332-47a1-a767-c82f2eba88e7.parquet
2025-08-23 06:15:38  934 Bytes test_iceberg/data/4TIMHA/20250822_211536_00063_xxjk2-2b92bf7b-4cbc-4eab-b95c-88ae8e20a557.parquet
2025-08-23 06:15:38  359 Bytes test_iceberg/data/sd21wQ/20250822_211536_00063_xxjk2-c165da75-433d-4b0b-adec-0a27b399dff1.parquet

Total Objects: 3
   Total Size: 1.6 KiB
```


OPTIMIZEを実行してMERGE INTOで参照先が増えてしまったオブジェクトを統合させます。

```sql:
OPTIMIZE default.test_iceberg REWRITE DATA USING BIN_PACK;
```

結果の上から3行目 06:19:01がOPTIMIZEで新たに作成されたオブジェクトです。

```
$ aws s3 ls s3://quark-sandbox/test_iceberg/data/ --human --recursive --sum
2025-08-23 06:10:33  345 Bytes test_iceberg/data/20250822_211032_00223_uzhbd-bcdccbb0-a332-47a1-a767-c82f2eba88e7.parquet
2025-08-23 06:15:38  934 Bytes test_iceberg/data/4TIMHA/20250822_211536_00063_xxjk2-2b92bf7b-4cbc-4eab-b95c-88ae8e20a557.parquet
2025-08-23 06:19:01  359 Bytes test_iceberg/data/o7K3Kg/20250822_211859_00095_22kit-49ae08ea-6d76-4968-ac56-36f3ff2777a5.parquet
2025-08-23 06:15:38  359 Bytes test_iceberg/data/sd21wQ/20250822_211536_00063_xxjk2-c165da75-433d-4b0b-adec-0a27b399dff1.parquet

Total Objects: 4
   Total Size: 2.0 KiB
```


VACUUMを実行してそれ以外が期限切れ非参照で消えることを確認します。

```sql:
VACUUM default.test_iceberg;
```

消えました！

```bash:
$ aws s3 ls s3://quark-sandbox/test_iceberg/data/ --human --recursive --sum
2025-08-23 06:19:01  359 Bytes test_iceberg/data/o7K3Kg/20250822_211859_00095_22kit-49ae08ea-6d76-4968-ac56-36f3ff2777a5.parquet

Total Objects: 1
   Total Size: 359 Bytes
```


### s3:DeleteObjectを与えずに実行

ということで、[VACUUMによるオブジェクト削除の確認](#vacuumによるオブジェクト削除の確認)について`s3:DeleteObject`を拒否するポリシーを付けた状態で行ってみます。


まずは`s3:DeleteObject`を拒否するポリシーを作ってアタッチします。

```json:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::quark-sandbox/*"
    }
  ]
}
```

適当なファイルでポリシーが効いていることを確認します。

```bash:
$ touch test.txt
$ aws s3 cp test.txt s3://quark-sandbox/test.txt
upload: ./test.txt to s3://quark-sandbox/test.txt
$ aws s3 rm s3://quark-sandbox/test.txt
delete failed: s3://quark-sandbox/test.txt An error occurred (AccessDenied) when calling the DeleteObject operation: User: arn:aws:iam::AWS_ACCOUNT_ID:user/Quark is not authorized to perform: s3:DeleteObject on resource: "arn:aws:s3:::quark-sandbox/test.txt" with an explicit deny in an identity-based policy
```

VACUUMまで再実行します。

CREATE TABLE→MERGE INTO→OPTIMIZEまでの結果です。
同様に上から3番目にOPTIMIZEされたオブジェクトが表示されています。

```bash:
$ aws s3 ls s3://quark-sandbox/test_iceberg/data/ --human --recursive --sum
2025-08-23 06:26:14  345 Bytes test_iceberg/data/20250822_212612_00071_udmih-51c1825f-970d-447c-9532-e980c92fa655.parquet
2025-08-23 06:27:17  935 Bytes test_iceberg/data/CYADSQ/20250822_212715_00135_ckh9u-35383cee-a64b-481a-8f0f-1fb24641f397.parquet
2025-08-23 06:29:07  359 Bytes test_iceberg/data/_CC_2Q/20250822_212904_00055_ycwtz-84f6f294-d534-481b-964e-fa4880550b0c.parquet
2025-08-23 06:27:17  359 Bytes test_iceberg/data/iTrhOQ/20250822_212715_00135_ckh9u-9434680a-e309-446d-99c5-27b6e32b8b0c.parquet

Total Objects: 4
   Total Size: 2.0 KiB
```

VACUUMを実行してみます。本来は上から3番目以外が消えますが、`s3:DeleteObject`がないので消せません。

```sql:
VACUUM default.test_iceberg;
```

![](/images/athena_vacuum_object_delete_test/2025-08-23_063123.png)

> ICEBERG_FILESYSTEM_ERROR: Failed to delete file from S3 for table: default.test_iceberg. Please check your IAM permissions.

**IAM パーミッションをチェックしてくれとエラーが返ってきました！**


### VACUUMで削除対象のS3オブジェクトが無い場合

削除対象のS3オブジェクトが無い場合どうなるのかをCREATE TABLE→VACUUMで試してみます。

```sql:
CREATE TABLE default.test_iceberg2
WITH (
  table_type = 'ICEBERG',
  is_external = false,
  location = 's3://quark-sandbox/test_iceberg2/',
  format = 'parquet',
  vacuum_min_snapshots_to_keep = 1,
  vacuum_max_snapshot_age_seconds = 60
) AS
SELECT *
FROM (
  VALUES
    (1, 'alice'),
    (2, 'bob')
) AS t (id, name);

```

S3を確認します。

```
$ aws s3 ls s3://quark-sandbox/test_iceberg2/data/ --human --recursive --sum
2025-08-23 06:35:38  345 Bytes test_iceberg2/data/20250822_213536_00167_h3s65-47a36fc5-17e3-456d-9a01-4a5963f4ef49.parquet

Total Objects: 1
   Total Size: 345 Bytes
```


次にVACUUMします。削除対象のオブジェクトはないはずです。


```sql:
VACUUM default.test_iceberg2;
```

![](/images/athena_vacuum_object_delete_test/2025-08-23_063718.png)


成功しました！削除対象が無い場合はそのまま走るということですね。
