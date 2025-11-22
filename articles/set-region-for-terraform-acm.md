---
title: "CloudFrontのACMをTerraformで作成する際のRegion指定"
emoji: "⚒️" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["AWS", "Terraform" ] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

## 課題内容

AWSのCloudFrontはグローバルサービスとして、us-east-1で作成されます。
CloudFrontでカスタムドメインを設定する場合に、AWS ACMを利用する際にはそのACMも同じus-east-1のregionで作成する必要があります。

ここまではよく言及される内容であり、Terraformでのコードサンプルも多くあるのですが、
Terraformで作成した際に、ACMの検証をTerraformで定義したRoute53のドメイン検証を利用する場合には、
この**ドメイン検証部分** にもreigionの指定が必要でした。

## 設定例

サンプルコードは以下になります。
他に参考となる記事も多いため、Route53のコード、providerなどの各種他の設定は割愛しています。

```tf
resource "aws_acm_certificate" "example_app" {
  domain_name = var.example_app_root_domain
  # ワイルドカード証明書のため次を指定
  subject_alternative_names = ["*.${var.example_app_root_domain}"]
  validation_method         = "DNS"
  provider                  = aws.virginia
  lifecycle {
    create_before_destroy = true
  }
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-acm-cert"
  })
}

# 作成したACMの検証
resource "aws_acm_certificate_validation" "example_app" {
  certificate_arn = aws_acm_certificate.example_app.arn
  # apply時SSL証明書の検証が完了するまで待機する設定
  validation_record_fqdns = flatten(values(aws_route53_record.example_app_cert_validation)[*].fqdn)
  # 証明書と同じリージョン(us-east-1)で検証を実行する必要がある
  provider = aws.virginia
}
```


参考になりますと幸いです！

