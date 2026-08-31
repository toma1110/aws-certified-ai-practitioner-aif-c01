# Amazon Bedrock Knowledge Basesで根拠付き回答を確認する

## 目的

Amazon Bedrock Knowledge Basesを使い、次の流れを一つずつ観察します。

1. 質問（query）を送る
2. 関連する文章（retrieved chunk）が取得される
3. 取得元（source）を確認する
4. 質問と取得結果を使って回答が生成される
5. 回答をcitationと元文章に照らして検証する

この手順は講師環境で検証した参考結果を再現するためのものです。AWSが内部で使用する非公開promptを推測したり、表示されていない内容を「実際のprompt」として扱ったりしません。

## 前提

- Amazon Bedrockを利用できるAWSアカウント
- Knowledge BasesとAmazon S3を操作できる権限
- 利用する生成モデル、embedding model、Knowledge Basesに必要なmodel access
- Regionは一つに固定する（講師の検証環境は`us-east-1`）
- 実習専用の空のS3 bucketを用意できること
- 顧客情報、credential、secret、個人情報を入力しないこと

AWS Consoleの表示やmanaged modelの選択肢は更新される場合があります。画面上の名称が異なるときは、後述の公式documentationで現在の手順を確認してください。

## 実習resourceを識別して記録する

既存resourceとの取り違えや削除漏れを防ぐため、作成前に実習専用の名前prefixを決めます。例は`aif-c01-rag-demo-<短い識別子>`です。S3 bucket名は全AWSアカウントで一意である必要があるため、他者が推測しにくい短い識別子を使い、account IDや個人情報は含めません。

作成時に次を手元へ記録し、すべて同じRegion・同じ実習用prefixに対応していることを確認します。

| 記録項目 | 記録する値 |
| --- | --- |
| Region | Knowledge Baseと関連resourceを作成したRegion |
| Knowledge Base | Knowledge Base名とID |
| Data source | data source名とID |
| S3 | 実習専用bucket名とobject key |
| Vector store | 選択した方式、管理service、indexまたはstore名 |
| IAM | Console workflowで作成したBedrock service role名 |

この一覧はcleanupで削除対象を特定するために使います。既存resourceや別の実習resourceを一覧へ混ぜないでください。

## 固定入力を作る

UTF-8のtext fileを作り、次の内容を保存します。file名の例は`aif-c01-rag-demo.txt`です。

```text
Learner support policy

The return window for standard course materials is 45 calendar days from the delivery date.
A refund request must include the order ID and the delivery date.
Digital download codes are not eligible for return after activation.
```

この架空のpolicyは実習専用です。実在する会社やCourseの返品規約ではありません。

## 実行手順

### 1. S3へ固定入力を置く

1. 実習専用bucketを作成します。
   - 名前は記録した実習用prefixを基にし、作成後のexact bucket名を一覧へ記録します。
2. Block Public Accessを有効のままにします。
3. default encryptionを有効にします。
4. 固定入力fileをuploadします。
5. bucket、object、Regionが実習専用であることを確認します。

account ID、bucket名、presigned URLなどをscreen captureや共有資料へ含めないでください。

### 2. Managed Knowledge Baseを作る

1. Amazon Bedrock ConsoleでKnowledge Basesを開きます。
2. Knowledge Baseを新規作成します。
3. S3をdata sourceに選び、固定入力を置いた場所を指定します。
4. embedding modelとvector storeは利用可能なmanaged optionを選びます。
5. 必要なservice roleはConsole workflowで作成します。
6. 作成完了後、Knowledge Baseとdata sourceが利用可能な状態になるまで待ちます。
7. Knowledge Base名とID、data source名とID、vector storeの方式・管理service・indexまたはstore名、service role名を一覧へ記録します。

実習では一時resourceだけを作り、既存のproduction resourceを再利用しません。

### 3. Data sourceをsyncする

1. 作成したdata sourceを開きます。
2. syncを実行します。
3. statusが完了したことを確認します。
4. warningやfailed itemがないことを確認します。

### 4. Retrievalを確認する

Knowledge Baseのtest画面で、次の質問を送ります。

```text
How many calendar days after delivery are standard course materials eligible for return?
```

まずretrieval resultを確認します。

- `45 calendar days`を含むchunkが取得されている
- sourceがuploadした固定入力fileである
- 質問と関係のある文章が上位にある

scoreは検索方式やservice updateにより変わり得ます。特定の数値だけを合否条件にせず、取得内容とsourceの対応を確認してください。

### 5. 根拠付き回答を確認する

response generationを有効にして同じ質問を送ります。画面上で次を確認します。

- 回答が`45 calendar days`を示す
- delivery dateから数えることが明確である
- citationから固定入力の該当chunkへ辿れる
- 元文章にない条件を断定していない

質問とretrieved contextがmanaged generationへ渡り、回答とcitationが返る関係を観察します。provider内部のpromptが画面に表示されない場合、その文面を再現したとは主張しません。

## 講師環境での検証済み結果

2026年8月の講師環境では、次の結果を確認しました。

| 観察項目 | 検証結果 |
| --- | --- |
| Query | 標準教材をdelivery後何日まで返品できるか |
| Retrieval | 1件のsource chunk |
| Top score | 1.0 |
| Source fact | delivery dateから45 calendar days |
| Generated answer | 45 calendar days以内と回答 |
| Citation | marker 2個、cited source 1件 |
| Fact match | sourceとanswerが一致 |
| Error / throttling | 0.0% / 0.0%（講師環境での確認時） |

この結果は固定された講師環境の参考値です。受講者環境ではmodel、retrieval設定、Console更新などにより表示やscoreが異なる場合があります。到達点は、query、retrieval、source、answer、citationの意味的な対応を説明できることです。

## 結果の見方

- 正しいretrievalとは、質問に関連する根拠がsource付きで取得されることです。
- 正しいgenerationとは、回答が取得根拠に支えられ、sourceにない内容を断定しないことです。
- citationが表示されるだけでは十分ではありません。citation先の文章が回答を実際に支えているか確認します。
- 回答が不正確な場合、生成モデルだけでなく、入力document、chunking、retrieval result、質問の具体性を順に調べます。

## Troubleshooting

### Retrieval resultが0件になる

- data sourceのsync statusを確認する
- 正しいbucket、prefix、Regionを選んだか確認する
- uploadしたfileが対応形式で、空でないことを確認する
- 質問に固定入力の主要語を含めて再確認する

### 関係のないchunkが取得される

- data sourceへ不要なdocumentを混在させていないか確認する
- 質問を具体化する
- chunkingとretrieval設定を確認する
- scoreだけでなくchunk本文を読む

### 回答にcitationが付かない、または根拠と合わない

- retrieval-onlyで正しいchunkが返るか先に確認する
- response generationとcitation表示を有効にしているか確認する
- sourceに質問への明示的な答えがあるか確認する
- 根拠にない内容を求める質問へ変えていないか確認する

### 作成やsyncが失敗する

- Region、model access、IAM permissionを確認する
- S3 objectをservice roleが読めるか確認する
- Consoleのerror detailとCloudTrailを確認する
- resourceを重複作成せず、失敗原因を解消してから再試行する

## 料金と安全

講師の一時検証では、実行前の見積もりを`USD 0.05未満`としました。これは将来の料金を保証する値ではありません。Knowledge Bases、embedding、vector store、model invocation、S3などに料金が発生する可能性があります。

- 実行前に[Amazon Bedrock pricing](https://aws.amazon.com/bedrock/pricing/)と利用するvector storeの料金を確認する
- 固定入力と少数のqueryだけで検証する
- 同じ操作を不用意に繰り返さない
- 実習終了後、次の順序でresourceを削除する

## Cleanup

削除前に、作成時に記録した一覧と、現在表示されているRegion・resource名・ID・管理serviceを照合し、実習用resourceだけが対象であることを確認します。既存resourceや他の利用者のresourceを削除しないでください。

1. Knowledge Baseに紐づくdata sourceとmanaged vector index/storeの関係、およびvector storeを管理するserviceを確認します。
2. Knowledge Baseとdata sourceを削除します。この操作だけではvector store自体は削除されないため、ここではcleanup完了と判断しません。
3. vector storeを管理するserviceのConsoleまたはSDKで、実習用vector index/storeを明示的に削除し、同じRegionの一覧または取得APIで対象が0件になったことを確認します。
4. 実習用に自動作成されたBedrock service roleをIAMから削除します。
5. 実習用S3 objectを削除します。
6. 実習専用S3 bucketが空であることを確認してbucketを削除します。
7. 同じRegionでKnowledge Bases一覧を再読込し、対象が0件であることを確認します。
8. IAMで実習用service role、S3で実習用bucketがそれぞれ検索結果0件であることを確認します。
9. 最後にvector storeを管理するserviceのConsoleまたはSDKを再確認し、実習用vector storeが0件であることを記録します。CloudFormation、CloudWatch log deliveryなど、作成時に追加されたresourceについても残存がないか確認します。

作成時に記録した一覧の各resourceが、該当Regionと管理serviceで0件またはNot foundになったことを確認できた時点でcleanup完了です。

## 公式AWS documentation

- [Amazon Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
- [Delete a Knowledge Base](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-delete.html)
- [Retrieve data and generate AI responses](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-how-retrieval.html)
- [Test a Knowledge Base](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-retrieve.html)
- [Amazon Bedrock pricing](https://aws.amazon.com/bedrock/pricing/)
- [AWS Certified AI Practitioner AIF-C01 Domain 3](https://docs.aws.amazon.com/aws-certification/latest/ai-practitioner-01/ai-practitioner-01-domain3.html)
