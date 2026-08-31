# Amazon Bedrockで基盤モデルを呼び出し、トークン数と料金を確認する

## このデモで確認すること

Amazon BedrockでFoundation Modelを1回だけ呼び出し、次の流れを実画面で確認します。

1. 目的に合うモデルを選ぶ
2. 入力メッセージを送る
3. モデルの出力を確認する
4. 入出力トークン数を費用と結び付ける

これは実装力を問うハンズオンではありません。AIF-C01で、モデル、入力、出力、トークン量、費用を同じ流れとして判断できるようにするための最小デモです。受講者自身での操作は任意です。

## 実行条件

- 実行環境: AWS CloudShell
- Region: `us-east-1`
- Service: Amazon Bedrock Runtime
- Model: Amazon Nova Micro
- Model ID: `amazon.nova-micro-v1:0`
- 課金方式: Standard on-demand
- 必要な操作: `bedrock:InvokeModel`

Amazon Nova Microはテキスト専用で、速度と低費用を重視するモデルです。このデモは短いテキスト応答だけが必要なため、この条件に合うモデルとして選びます。

実行前に最新のAmazon Bedrock料金を確認してください。2026-08-28の検証時点では、Nova Microは入力1,000トークンあたりUSD 0.000035、出力1,000トークンあたりUSD 0.00014でした。

## 準備してから実行する

1. AWSアカウントへサインインします。
2. AWS Management Console上部のCloudShellアイコンを選び、CloudShellを開きます。
3. 利用するリージョンが`us-east-1`であることを確認します。
4. `bedrock:InvokeModel`を実行できる権限があることを確認します。権限がない場合は自分で追加せず、管理者へ確認します。
5. 次のコマンドをCloudShellへ貼り付けて実行します。

## 入力して実行する

```bash
aws bedrock-runtime converse \
  --region us-east-1 \
  --model-id amazon.nova-micro-v1:0 \
  --messages '[{"role":"user","content":[{"text":"AWS Certified AI Practitionerの学習ポイントを日本語で1文だけ説明してください。"}]}]' \
  --inference-config '{"maxTokens":64,"temperature":0}' \
  --output json
```

`maxTokens`は出力の上限です。短い1文へ絞ることで、デモの費用と実行時間を小さくします。

## 検証済みの実行結果

同じリクエストを2026-08-28に1回実行した結果です。生成結果は再実行時に変わる可能性があるため、文言の完全一致を成功条件にはしません。

```json
{
  "output": {
    "message": {
      "role": "assistant",
      "content": [
        {
          "text": "AWS Certified AI Practitionerの学習ポイントは、AWSのAI/MLサービスの基礎知識とそれらを活用してデータ駆動型ソリューションを設計・実装する方法を理解することです。"
        }
      ]
    }
  },
  "stopReason": "end_turn",
  "usage": {
    "inputTokens": 17,
    "outputTokens": 47,
    "totalTokens": 64
  },
  "metrics": {
    "latencyMs": 491
  }
}
```

見るべき点は回答文だけではありません。

- `inputTokens`: モデルへ渡した入力トークン数
- `outputTokens`: モデルが生成した出力トークン数
- `totalTokens`: 入力と出力の合計
- `stopReason: end_turn`: モデルが応答を完了したこと

このリクエストの推定料金は次のとおりです。

```text
17 / 1,000 × $0.000035 + 47 / 1,000 × $0.00014
= $0.000007175
```

つまり、同じモデルでもプロンプトや出力が長くなればトークン数が増え、費用も増えます。試験では、モデルの品質やレイテンシーだけでなく、入出力トークン量と費用も選択条件になります。

## 想定と異なる場合

- `AccessDeniedException`: 自分で権限を追加せず、`bedrock:InvokeModel`が許可されているか管理者へ確認します。
- モデルやリージョンのエラー: `us-east-1`と`amazon.nova-micro-v1:0`を指定しているか確認します。
- 出力が途中で終わる: `stopReason`が`max_tokens`なら、学習目的と料金を確認したうえで`maxTokens`を調整します。
- 生成文やトークン数が検証結果と異なる: 生成結果は変わり得ます。`output.message`と`usage`が返り、意味の通る応答であることを確認します。

## 後片付け（Cleanup）

このデモはオンデマンド推論を1回実行するだけで、永続リソースを作成しません。Knowledge Base、Agent、Provisioned Throughput、カスタムモデル、IAMリソースを作成していないため、削除操作はありません。

デモ終了時は、作成系の操作を追加していないことと、CloudShellに認証情報やシークレットを保存していないことを確認します。

## 公式情報

- [Amazon Nova Micro model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-amazon-nova-micro.html)
- [Amazon Bedrock Converse API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_Converse.html)
- [Amazon Bedrock pricing](https://aws.amazon.com/bedrock/pricing/)
