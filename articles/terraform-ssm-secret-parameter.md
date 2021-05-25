---
title: "AWSで使うパスワードなどの機密情報を暗号化してGit管理する"
emoji: "🔑"
type: "tech"
topics: ["AWS", "Terraform"]
published: true
---
# AWSで使うパスワードなどの機密情報を暗号化してGit管理する

## はじめに
AWSでパスワードやクレデンシャル情報などの機密情報の管理はどうしていますか？  
AWS Systems Managerパラメータストアに値を登録し、EC2などから参照していることが多いかと思います。  
今回はTerraformを使い、パラメータストアに機密情報を安全に保存する方法をご紹介します。

## パラメータストアに登録
Terraformでは以下のコードでパラメータストアに登録することができます。  
今回の場合は、 `password` という文字列が機密情報とします。
```
resource "aws_ssm_parameter" "parameter" {
  name  = "secret-value"
  value = "password"
  type  = "SecureString"
}
```
機密情報を登録するため `type` は `SecureString` を選択し、暗号化して保存されるようにします。

登録されたパラメータはコンソールからこのように確認できます。
![](https://storage.googleapis.com/zenn-user-upload/mtr76ekv6ea5r4yj3ocjggu7pu3k)

しかし、Terraformコード上では暗号化されていないため、パスワードがわかってしまう状態になってしまいます。  
このままではGit管理することが出来ません。  
そのため、この `password` という文字列を暗号化した状態で記述する必要があります。

## aws_kms_secretsを使う
そこで、Terraformの `aws_kms_secrets` というData Sourceを使います。  
[aws_kms_secrets | Data Sources | hashicorp/aws | Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/kms_secrets)
```
data "aws_kms_secrets" "secret" {
  secret {
    name = "encrypted_password"
    payload = "[暗号化した文字列]"
  }
}

resource "aws_ssm_parameter" "parameter" {
  name  = "secret-value"
  value = data.aws_kms_secrets.secret.plaintext["encrypted_password"]
  type  = "SecureString"
}
```
payloadに暗号化した文字列を入れると、plaintextで復号した文字列が取得できます。  
このようにすれば、`password` という文字列は暗号化されているため、安全に管理できます。

## 機密情報を暗号化する
payloadに入れるための暗号化された文字列を生成します。

文字列を暗号化するためKMSでキーを作成します。  
aliasも作っているため、`alias/my-key`という名前でキーを使えます。

```
resource "aws_kms_key" "key" {}

resource "aws_kms_alias" "key" {
  name = "alias/my-key"
  target_key_id = aws_kms_key.key.key_id
}
```

次に、キーを使って `password` という文字列を暗号化します。  
（AWSのクレデンシャルを予めセットしておく必要があります）
```bash
aws kms encrypt --key-id alias/my-key --plaintext fileb://<(echo -n 'password') --query CiphertextBlob --output text
```
このコマンドを実行すると暗号化した文字列が出力されるため、この値を `aws_kms_secrets` のpayloadにセットすれば完成です。

コマンドから復号したい場合には以下のコマンドを実行します。
```bash
aws kms decrypt --key-id alias/my-key --ciphertext-blob fileb://<(echo -n '[暗号化した文字列]' | base64 --decode) --output text --query Plaintext | base64 --decode
```

## おわりに
平文で記述しなければならないために、機密情報をTerraformで管理出来ていなかった方は試してみてください。  
ただし、この方法でもStateには平文の文字列が表示されるため、管理には注意をしてください。

## 参考

[aws_ssm_parameter | Resources | hashicorp/aws | Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ssm_parameter)  
[aws_kms_secrets | Data Sources | hashicorp/aws | Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/kms_secrets)  
[AWS Systems Manager パラメータストア - AWS Systems Manager](https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/systems-manager-parameter-store.html)