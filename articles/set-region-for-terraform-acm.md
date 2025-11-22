---
title: "CloudFrontのACMをTerraformで作成する際のRegion指定"
emoji: "⚒️" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["AWS", "Terraform" ] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

## 課題内容

AWSのCloudFrontはグローバルサービスとして、us-east-1で作成されます。
CloudFrontでカスタムドメインを設定する場合に、AWS ACMを利用する際にはそのACMも同じus-east-1のregionで作成する必要がります。

Terraformで作成する際には、domain検証のterraform定義部分でも、virginiaを指定する
検証が失敗してエラーになる
