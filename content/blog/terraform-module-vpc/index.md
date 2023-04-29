---
title: TERRAFORMのMODULEを作成する(VPC)
description: TerraformでAWSのVPCを作成してみます
slug: terraform-module-vpc
date: 2019-12-14 00:00:00+0000
image: cover.jpg
categories:
    - IaC
tags:
    - Terraform
---

みなさんこんにちは！  
インフラエンジニアのねずみさん家です。  

今回はTerraformのModuleを作成してAWSのVPCを作成します。  
初めての投稿なので初回セットアップの部分も書いてみたいと思います。  
Terraformは非常に便利なツールなので是非利用してみてくださいー。  

## Terraformのインストール

Macを利用していてbrewが利用できる方は以下でインストール可能です。  

    brew install terraform

TerraformはGolangで開発されているため  
[**こちらから**](https://www.terraform.io/downloads.html)バイナリファイルをダウンロードすることも可能です。  

インストールできたらバージョンを確認してみましょう。

    terraform version

今回はバージョン0.12.18を利用します。  
古いバージョンの0.11ではサポートされていない記述も含まれているためご注意ください。  

## クレデンシャルの用意

Terraformの処理でAWSのAPIを実行することになるためクレデンシャルを用意します。  
用意したらTerraformから読み込むために環境変数に登録します。  
変数名の頭にTF_VARとつけることでTerraformから読み込むことができるようになります。

    export TF_VAR_access_key=[Access Key]
    export TF_VAR_secret_key=[Secret Key]
    export TF_VAR_role_arn=[Role ARN]

terraform.tfvarsに記載しても良いですが  
Gitに誤って上げてしまって不正アクセスを受けたという記事が散見されるため  
基本的にはファイルに直接認証情報を記載しないのが良いと思います。  

また今回はTerraformからAssumeRoleを行うため  
適切な権限を付与したIAMロールも用意しておきます。

## HCLの記述

ここからHCLを記載していきます。  
HCLはHashicorp Configuration Languageの略で  
Terraformのコードを書いていくときに利用する独自言語を指します。  

今回記載したVPCのモジュールの階層構造は以下のようになっています。

    .
    ├── network
    │   └── vpc
    │       ├── main.tf
    │       ├── outputs.tf
    │       └── variables.tf
    ├── network.tf
    ├── outputs.tf
    ├── provider.tf
    ├── terraform.tfstate
    ├── terraform.tfstate.backup
    └── variables.tf

tfstateをローカルで管理していますが  
実際に使う場合はbackendでS3などで管理するようにするのが定石です。  

まずはprovider.tfを書いていきましょう。

    provider "aws" {
      access_key = var.access_key
      secret_key = var.secret_key
      region     = var.region

      assume_role {
        role_arn = var.role_arn
      }
    }

providerではAWSにアクセスするために必要な認証情報を指定します。  
Terraformではvariableで変数を定義してvar.変数名で変数の値を取得することが可能です。  
variables.tfの中身を見てみましょう。

    variable "access_key" {
      description = "AWS Access Key"
    }

    variable "secret_key" {
      description = "AWS Secret Key"
    }

    variable "role_arn" {
      description = "AWS Role ARN for Assume Role"
    }

    variable "region" {
      description = "AWS Region"
      default     = "ap-northeast-1"
    }

variableで変数を定義しています。  
この変数には先ほど環境変数に指定した値が自動で代入されます。  

ここまで書けたらいよいよVPCの作成部分を書いていきましょう。  
network.tfの中身を見てみます。

```
module "vpc" {
  source = "./network/vpc"
  
  vpc_config = {
    name                 = "example"
    cidr_block           = "10.0.0.0/16"
    enable_dns_support   = true
    enable_dns_hostnames = true
  }
}
```

sourceに読み込むModuleの処理を記載した.tfが存在するディレクトリの相対パスを記載します。  
また、Module側で必要な値を変数化するため  
vpc_configという名前で変数を作成してModule側に渡しています。

Module側を見ていきます、まずはvariables.tfから見ていきましょう。  
ここでnetwork.tfから渡された変数を受け取っています。

```
variable "vpc_config" {
  type = object({
    name                 = string
    cidr_block           = string
    enable_dns_support   = bool
    enable_dns_hostnames = bool
  })
}
```

Terraform0.12からobjectという便利なタイプが追加されています。  
0.11まではmap型でも同じ型の値しか格納することができませんでしたが  
objectを利用するとstringとboolといった異なる型の値を同時に扱うことができます。  

次にmain.tfを見ていきましょう。  
ここでVPCを作成するためにaws_vpcリソースを記述しています。  

```
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_config.cidr_block
  enable_dns_support   = var.vpc_config.enable_dns_support
  enable_dns_hostnames = var.vpc_config.enable_dns_hostnames

  tags = {
    Name = var.vpc_config.name
  }
}
```

ここでも値が変数化されていますが  
vpc_configという名前で呼び出し側から値を受け取っているため  
var.変数名.オブジェクトキーというように指定すると値を参照することができます。

ここまでくればもう一息です。  
最後にリソースが作成された後の情報を参照できるようにoutputを書いておきましょう。  
outputs.tfを見ていきます。  

```
output "vpc" {
  value = aws_vpc.this
}
```

ここも0.12から追加された記述を利用しています。  
0.11までは1つのoutputに対して1つの値しか扱うことができませんでしたが  
0.12からは複数の値を扱うことができるようになります。  
aws_vpc.リソース名とするとvpc_idやcidr_blockなどの複数の値がmap型でまとめて出力されます。  

ここまで書いたらVPCを作成することが可能です。
まずはドライランを実行してみましょう。

```
terraform plan
```

成功すると以下のように結果が返ってきます。  
エラーが出た場合はHCLに記述ミスがある可能性があるので書き直しましょう。  

```
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # module.vpc.aws_vpc.this will be created
  + resource "aws_vpc" "this" {
      + arn                              = (known after apply)
      + assign_generated_ipv6_cidr_block = false
      + cidr_block                       = "10.0.0.0/16"
      + default_network_acl_id           = (known after apply)
      + default_route_table_id           = (known after apply)
      + default_security_group_id        = (known after apply)
      + dhcp_options_id                  = (known after apply)
      + enable_classiclink               = (known after apply)
      + enable_classiclink_dns_support   = (known after apply)
      + enable_dns_hostnames             = true
      + enable_dns_support               = true
      + id                               = (known after apply)
      + instance_tenancy                 = "default"
      + ipv6_association_id              = (known after apply)
      + ipv6_cidr_block                  = (known after apply)
```

ドライランは問題ないようですので実際に反映してみましょう。

```
terraform apply
```

反映されたら作成されたVPCの情報を参照したいため  
Module側で指定したoutputを呼び出し側で受け取り内容を確認します。  
呼び出し元のoutputs.tfを確認します。

```
output "vpc_id" {
  value = module.vpc.vpc.id
}
```

ここで記述するのはmodule.モジュール名.参照する値となり  
今回はVPCのIDを確認したいためmodule.vpc.vpc.idとなります。  
記述したら再度terraform applyを実行します。

```
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

vpc_id = vpc-XXXXXXXXXXXXXXXXX
```

正常に実行されるとこのようにVPCのIDを確認することができます。  
これでVPCのモジュールは完成です。  
作成したリソースが不要になったら以下のコマンドで消してくださいね。

```
terraform destroy
```

## 次回

VPCを作成したので次はサブネットの作成を行います。  
また後日記事にまとめるのでお楽しみにー！！
