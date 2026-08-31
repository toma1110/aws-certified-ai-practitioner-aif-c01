# Agentのtool useを五つの観察点で確認する

## 目的

Amazon Bedrock RuntimeのConverse APIを使い、利用者の依頼から最終回答までを、画面に返る情報だけで追います。モデルの内部思考を推測する演習ではありません。

確認する順序は次のとおりです。

1. 利用者の依頼（request）
2. 観測可能な行動選択（observable action decision）
3. ツール呼び出し（tool call）
4. ツールの観測結果（observation）
5. 最終回答（response）

## 前提

- Amazon Bedrockを利用できるAWS環境
- Amazon Nova Microを利用できるRegionとmodel access
- 対象modelに対する`bedrock:InvokeModel`権限
- AWS CloudShellを起動できること
- 検証用の値だけを使用し、認証情報（credential）、個人情報、顧客データを入力文やログへ含めないこと

この例では、Amazon Bedrock Runtime、`us-east-1`、US geo inference profileの`us.amazon.nova-micro-v1:0`、実行program内の決定的な計算関数を使用します。Bedrock Agent、Lambda、IAM role、S3、databaseなどの永続AWS resourceは作成しません。

## 実行する依頼

```text
単価12,800円の商品を17%割引で3個購入します。合計金額を円単位で計算してください。必ず calculate_total_price tool を使ってください。
```

## 実行手順

### 1. CloudShellを準備する

1. AWS ConsoleでRegionを`us-east-1`にします。
2. AWS CloudShellを開きます。
3. PythonとBoto3を確認します。

```bash
python3 --version
python3 -c "import boto3; print(boto3.__version__)"
```

`ModuleNotFoundError`になる場合は、CloudShellのPython環境へBoto3を追加してから再確認します。access keyやsecret keyをcodeへ書かず、CloudShellに付与された現在のAWS sessionを使用します。

### 2. 完全なprogramを保存する

CloudShellのEditorを開き、次の全内容を`agent_tool_demo.py`として保存します。

```python
from decimal import Decimal, ROUND_HALF_UP
import json

import boto3


REGION = "us-east-1"
MODEL_ID = "us.amazon.nova-micro-v1:0"
USER_REQUEST = (
    "単価12,800円の商品を17%割引で3個購入します。"
    "合計金額を円単位で計算してください。"
    "必ず calculate_total_price tool を使ってください。"
)

TOOL_CONFIG = {
    "tools": [
        {
            "toolSpec": {
                "name": "calculate_total_price",
                "description": "数量、単価、割引率から円単位の合計金額を計算する",
                "inputSchema": {
                    "json": {
                        "type": "object",
                        "properties": {
                            "quantity": {"type": "integer", "minimum": 1},
                            "unit_price_yen": {"type": "integer", "minimum": 0},
                            "discount_percent": {
                                "type": "number",
                                "minimum": 0,
                                "maximum": 100,
                            },
                        },
                        "required": [
                            "quantity",
                            "unit_price_yen",
                            "discount_percent",
                        ],
                    }
                },
            }
        }
    ]
}


def calculate_total_price(tool_input):
    quantity = tool_input.get("quantity")
    unit_price_yen = tool_input.get("unit_price_yen")
    discount_percent = tool_input.get("discount_percent")

    if isinstance(quantity, bool) or not isinstance(quantity, int) or quantity < 1:
        raise ValueError("quantity must be an integer greater than or equal to 1")
    if (
        isinstance(unit_price_yen, bool)
        or not isinstance(unit_price_yen, int)
        or unit_price_yen < 0
    ):
        raise ValueError("unit_price_yen must be a non-negative integer")
    if (
        isinstance(discount_percent, bool)
        or not isinstance(discount_percent, (int, float))
        or not 0 <= discount_percent <= 100
    ):
        raise ValueError("discount_percent must be between 0 and 100")

    discount = Decimal(str(discount_percent))
    total = (
        Decimal(unit_price_yen)
        * Decimal(quantity)
        * (Decimal("1") - discount / Decimal("100"))
    ).quantize(Decimal("1"), rounding=ROUND_HALF_UP)

    return {
        "total_yen": int(total),
        "calculation": (
            f"{unit_price_yen} × {quantity} × "
            f"(1 - {discount_percent}/100) = {int(total)}"
        ),
    }


client = boto3.client("bedrock-runtime", region_name=REGION)
messages = [{"role": "user", "content": [{"text": USER_REQUEST}]}]

# 1回目: 利用者の依頼とtool schemaをmodelへ渡す
first_response = client.converse(
    modelId=MODEL_ID,
    messages=messages,
    toolConfig=TOOL_CONFIG,
)
print("first_stop_reason:", first_response["stopReason"])
if first_response["stopReason"] != "tool_use":
    raise RuntimeError("The model did not request the expected tool")

# modelが返したassistant messageを会話履歴へそのまま追加する
assistant_message = first_response["output"]["message"]
messages.append(assistant_message)
tool_use = next(
    block["toolUse"]
    for block in assistant_message["content"]
    if "toolUse" in block
)
if tool_use["name"] != "calculate_total_price":
    raise RuntimeError(f"Unexpected tool: {tool_use['name']}")

print("tool_name:", tool_use["name"])
print("tool_input:", json.dumps(tool_use["input"], ensure_ascii=False))

# modelの入力値を検証してから、program内の計算関数を実行する
tool_result = calculate_total_price(tool_use["input"])
print("observation:", json.dumps(tool_result, ensure_ascii=False))

# 1回目で返されたtoolUseIdを使い、tool resultをuser messageとして追加する
messages.append(
    {
        "role": "user",
        "content": [
            {
                "toolResult": {
                    "toolUseId": tool_use["toolUseId"],
                    "content": [{"json": tool_result}],
                }
            }
        ],
    }
)

# 2回目: assistant messageとtool resultを含む全履歴をmodelへ返す
second_response = client.converse(
    modelId=MODEL_ID,
    messages=messages,
    toolConfig=TOOL_CONFIG,
)
print("second_stop_reason:", second_response["stopReason"])
final_text = "\n".join(
    block["text"]
    for block in second_response["output"]["message"]["content"]
    if "text" in block
)
print("final_response:", final_text)
```

### 3. Programを実行する

```bash
python3 agent_tool_demo.py
```

同じ操作を不用意に繰り返さず、最初の実行結果を確認してください。

### 4. 出力を五つの観察点へ対応付ける

出力の文面やJSON内のkey順は変わる場合があります。少なくとも次を確認します。

```text
first_stop_reason: tool_use
tool_name: calculate_total_price
tool_input: ... quantity=3、unit_price_yen=12800、discount_percent=17 ...
observation: ... total_yen=31872 ...
second_stop_reason: end_turn
final_response: ... 31,872円 ...
```

| 観察点 | Programで確認する場所 | 意味 |
| --- | --- | --- |
| 利用者の依頼（request） | `USER_REQUEST` | 何を計算し、どのtoolを使うか |
| 観測可能な行動選択 | `first_stop_reason: tool_use` | modelが次の行動としてtool利用を選んだ |
| ツール呼び出し | `tool_name`と`tool_input` | 選択したtoolと渡した値 |
| 観測結果（observation） | `observation` | 実行programが決定的に計算した結果 |
| 最終回答（response） | `second_stop_reason`と`final_response` | 観測結果を反映した利用者向け回答 |

`stopReason=tool_use`は外部から確認できる行動選択です。hidden chain-of-thoughtではなく、modelの非公開な内部思考を表示したものでもありません。

## Tool schemaの読み方

```json
{
  "toolSpec": {
    "name": "calculate_total_price",
    "description": "数量、単価、割引率から円単位の合計金額を計算する",
    "inputSchema": {
      "json": {
        "type": "object",
        "properties": {
          "quantity": { "type": "integer", "minimum": 1 },
          "unit_price_yen": { "type": "integer", "minimum": 0 },
          "discount_percent": { "type": "number", "minimum": 0, "maximum": 100 }
        },
        "required": ["quantity", "unit_price_yen", "discount_percent"]
      }
    }
  }
}
```

実行programの計算関数は、モデルから受け取った三つの値を検証してから、次の式を決定的に計算します。

```text
total_yen = unit_price_yen × quantity × (1 - discount_percent / 100)
```

## 確認済みの実行例

一回目のConverse呼び出しへ利用者の依頼とtool schemaを渡します。モデルの応答にある`stopReason`が`tool_use`なら、応答内のtool名と入力値を確認します。これは次の行動としてtoolが選ばれたことを示す観測値であり、非公開の内部思考ではありません。

確認済みのtool callは次のとおりです。

```text
calculate_total_price(quantity=3, unit_price_yen=12800, discount_percent=17)
```

実行programで計算した結果をtool resultとして二回目のConverse呼び出しへ返します。

```json
{
  "total_yen": 31872,
  "calculation": "12800 × 3 × (1 - 17/100) = 31872"
}
```

## 期待結果

- 一回目の`stopReason`が`tool_use`
- tool名が`calculate_total_price`
- inputが数量3、単価12,800円、割引率17%
- observationの`total_yen`が`31872`
- 最終回答が31,872円を反映
- 二回目の`stopReason`が`end_turn`

確認済みの最終回答は次のとおりです。

```text
単価12,800円の商品を17%割引で3個購入した場合の合計金額は31,872円です。
```

## Troubleshooting

- `AccessDeniedException`: sessionやpermission setを変更せず、対象modelへの`bedrock:InvokeModel`権限を管理者へ確認します。
- モデルを利用できない: 対象Region、model access、inference profile IDを確認します。
- `stopReason`が`tool_use`にならない: tool schema、tool名、必須項目、利用者の依頼でtool利用を明示しているかを確認します。
- 入力値がschemaに合わない: 実行programで型、範囲、必須項目を検証し、toolを実行せずerrorとして扱います。
- 計算結果が一致しない: modelの文章ではなく、計算関数とtool resultを正本にして照合します。
- 二回目の応答が得られない: 一回目のassistant messageが履歴にあること、`toolUseId`がtool useとtool resultで一致すること、tool resultが`user` roleで追加されていることを確認します。

## 料金

Bedrock Runtimeのmodel入出力tokenに料金が発生します。料金はmodel、Region、利用方式、token数で変わるため、実行前に最新の公式pricingを確認し、短い検証へ上限を設けてください。本教材の確認実行は入力1,253 tokens、出力150 tokensで、実行時の公開例示単価による概算は0.01米ドル未満でした。この値を現在または別のAWS accountの保証額として扱わないでください。

## Cleanup

この最小例は実行program内の一時的な計算関数だけを使い、永続AWS resourceを作りません。実行終了後、CloudShell上の教材fileを削除します。

```bash
rm agent_tool_demo.py
```

Bedrock Agent、Lambda、IAM role、S3、databaseなどの一覧に、この実習の名前を持つ永続resourceが作られていないことを確認します。構成を拡張して永続resourceを作成した場合は、この例の「削除対象なし」を流用せず、作成したserviceごとに削除し、対象が0件になったことを確認してください。

## 公式ドキュメント

- [Amazon Bedrock Converse API](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference-call.html)
- [Tool use with Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use-inference-call.html)
- [Amazon Nova Micro model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-amazon-nova-micro.html)
- [Amazon Bedrock pricing](https://aws.amazon.com/bedrock/pricing/)
