---
title: PACKERがHCL2をサポートしたのでDOCKERイメージを作成してみる
description: いつの間にかサポートされてました
slug: packer-hcl2-sample
date: 2020-03-29 00:00:00+0000
image: cover.jpg
categories:
    - IaC
tags:
    - Packer
---

## 記述したコード
- https://github.com/nezumisannn/packer-hcl2-sample.git

## Packerとは
- HashiCorpが開発しているCLIツール
- マルチクラウドに対応したゴールデンイメージの作成ができる
- 設定ファイルの書式はJson or HCL2
- バージョン1.5からHCL2に対応した

## 参照
- https://packer.io/guides/hcl/
- https://packer.io/docs/builders/docker.html
- https://packer.io/docs/provisioners/shell.html

## HCL2で記述するメリット
- TerraformもHCL2で記述するため共通の書式を利用できる
- 設定ファイルを用途に合わせて分割できる
- コードの途中でコメントを追加できる

## Packerのバージョン
```
$ packer version
Packer v1.5.5
```

## 設定ファイル一覧
```
$ tree
.
├── README.md
├── build.pkr.hcl
├── sources.pkr.hcl
└── variable.pkr.hcl
```

### variable.pkr.hcl
- variableを利用してコード内で利用する変数を定義する

### sources.pkr.hcl
- ビルド時に利用するイメージを利用する
- Json形式で記述する際のBuildersに相当する

### build.pkr.hcl
- ビルドを行う際の処理を記述する
- sourcesでsources.pkr.hclに書いた設定を複数読み込むことが可能
- その他Jsonで記述する際と同様にprovisionerでShellの他、Ansibleなどと連携可能

## ファイル内容

### variable.pkr.hcl
```
variable "image_nginx" {
  type    = string
  default = "nginx:latest"
}

variable "image_php-fpm" {
  type    = string
  default = "php:7.3-fpm"
}
```

### sources.pkr.hcl
```
/*
{
  "type": "docker",
  "image": "nginx:latest",
  "commit": true
}
*/
source "docker" "nginx" {
  image  = var.image_nginx
  commit = true
}

/*
{
  "type": "docker",
  "image": "php:7.3-fpm",
  "commit": true
}
*/
source "docker" "php-fpm" {
  image  = var.image_php-fpm
  commit = true
}
```

### build.pkr.hcl
```
build {
  sources = [
    "source.docker.nginx",
    "source.docker.php-fpm"
  ]

  provisioner "shell" {
    inline = [
      "hostname && cat /etc/os-release"
    ]
  }
}
```

## ビルド実行
```
$ packer build ./
```

## 現時点での問題
- packer validateが使えない
    - https://github.com/hashicorp/packer/issues/8538
    - 今後のアップデートに期待

