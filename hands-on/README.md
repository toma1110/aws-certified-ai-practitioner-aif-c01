# ハンズオン一覧

このページは、講座で扱う最小ハンズオンの共通入口です。各ハンズオンは、試験で必要な判断へつながる入力、出力、構成だけを確認します。

## 予定している確認

| テーマ | 確認すること | 現在の状態 |
| --- | --- | --- |
| Amazon BedrockでFoundation Modelを利用する | modelを選び、入力に対する出力とtoken使用量を確認する | 個別手順の公開前 |
| Knowledge Bases / RAGの構成を確認する | query、検索結果、source、回答の関係を確認する | 個別手順の公開前 |
| Agentのtool useを確認する | request、tool call、結果確認、responseの流れを確認する | 個別手順の公開前 |

個別手順へのlinkが表示されるまでは、AWS resourceを作成する操作はありません。

## 共通の事前確認

個別ハンズオンを開始する前に、そのREADMEで次を確認してください。

- 解決する課題と、終了時に確認できる結果
- 使用するAWS accountとRegion
- 作成、変更、削除するresource
- 必要なIAM permission
- 料金の発生条件と、実行時点の料金を確認する方法
- cleanupの削除順序と、削除後の残存確認

AWSの料金、service quota、利用可能なRegionは変わることがあります。金額や利用可否は個別READMEのlinkからAWS公式情報を確認し、想定と異なる場合は操作を止めてください。

## 共通の進め方

1. **課題を確認する**: 何を判断するためのハンズオンかを先に読みます。
2. **実行条件を確認する**: account、Region、permission、料金、resource名を確認します。
3. **手順を実行する**: 各手順の目的を確認しながら、指定された操作だけを実行します。
4. **期待結果と照合する**: 出力、状態、logなどをREADMEの期待結果と比べます。
5. **cleanupする**: 指定順にresourceを削除し、検索や一覧表示で残っていないことを確認します。

## Credentialとdata

- access key ID、secret access key、session token、passwordを貼り付けないでください。
- credential file、`.env`、consoleからdownloadした設定fileをrepositoryへ追加しないでください。
- sample dataが必要な場合は、個別READMEで指定する合成dataだけを使用してください。
- account ID、email address、社内情報を画面captureや共有fileへ残さないでください。

想定外の料金やresourceが見つかった場合は、新しい操作を続けず、対象service、Region、resource名を確認してから個別READMEのcleanupへ戻ってください。
