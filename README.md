# AWS Certified AI Practitioner (AIF-C01) Hands-on

AWS Certified AI Practitioner（AIF-C01）講座で使用する、受講者向けハンズオン資料の配布リポジトリです。

この講座ではAWSサービスを順番に操作して覚えるのではなく、入力、出力、構成、料金、安全性を確認し、問題の条件から適切な選択肢を判断できることを目指します。

## ハンズオンの入口

[ハンズオン一覧と共通の進め方](hands-on/README.md)を最初に確認してください。

各ハンズオンが公開されたら、講義と同じ名前の案内から次の順序で進めます。

1. 学習目的と確認する結果を読む
2. 対象Region、必要権限、作成するresource、料金の発生条件を確認する
3. 手順を実行し、期待結果と照合する
4. 想定と異なる場合の確認項目を順に調べる
5. cleanupを実行し、resourceが残っていないことを確認する

## 安全に利用するために

- 実際のaccess key、secret、session token、password、個人情報、production dataをfileやcommandへ書き込まないでください。
- credentialをrepositoryへcommitまたはpushしないでください。
- 特別な指定がない限り、AWS Management Consoleから起動したAWS CloudShellを使用します。
- 講義で指定されたaccountとRegionを使用し、作成対象が明記されていない操作は実行しないでください。
- 実行前に料金の発生条件を確認し、終了後は各ハンズオンのcleanupと残存確認を完了してください。
- 組織や会社のAWS accountを利用する場合は、その組織のpolicyと承認手順を優先してください。

この初期構成には実行scriptやAWS resource定義は含まれず、リポジトリを閲覧するだけではAWS料金は発生しません。

## License

このリポジトリの内容は[MIT License](LICENSE)で提供します。
