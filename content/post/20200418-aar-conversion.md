---
title: "Azure Remote Rendering用のアセットを作る"
date: 2020-04-25T12:19:17+09:00
slug: "20200418_aar-conversion"
author: "d_yama"
draft: false
categories: ["Azure"]
tags: ["Azure Remote Rendering", "Azure"]
---

# mixpaceと連携させてみた

先日public previewで公開された[Azure Remote Rendering](https://azure.microsoft.com/ja-jp/services/remote-rendering/)(以下、AAR)について、mixpaceでもAAR用のアセットを作るコンバータのプロトタイプを作ってみました。

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">mixpace にモデル(CAD/BIM/CG)を突っ込むと勝手にAzure Remote Renderingに入るようになったｗ <a href="https://twitter.com/hashtag/HoloLens?src=hash&amp;ref_src=twsrc%5Etfw">#HoloLens</a> <a href="https://twitter.com/hashtag/%E3%81%95%E3%81%99%E3%82%84%E3%81%BE?src=hash&amp;ref_src=twsrc%5Etfw">#さすやま</a> <a href="https://t.co/EsppFHlxp1">pic.twitter.com/EsppFHlxp1</a></p>&mdash; 中村 薫(Kaoru Nakamura)@HoloLab Inc. (@kaorun55) <a href="https://twitter.com/kaorun55/status/1248956512942211072?ref_src=twsrc%5Etfw">April 11, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

mixpaceはCADやBIMといった3DデータをコンバートしてHoloLens 2やiPad上のクライアントアプリで閲覧できるサービスなのですが、コンバート後のファイルフォーマットとして[GLB](https://www.khronos.org/gltf/)を採用しています。AARではアセットを作るためのデータソースとしてFBX、glTF、GLBをサポートしているのでmixpaceにとっては都合が良いです。

AAR用のアセットを作るための方法として、[サンプルプロジェクトのリポジトリ](https://github.com/Azure/azure-remote-rendering)にてPowerShellスクリプトが公開されています。中身をみると分かるのですが、AAR用のアセットへのコンバートはREST APIを叩くだけで実行できます。

今回はAAR用アセットの作り方を少し紹介したいと思います。

# AAR用アセットへのコンバート

コンバートの手順はざっくり以下の通りです。

1. [Blob Storage](https://azure.microsoft.com/ja-jp/services/storage/blobs/)を用意する
2. Blob Storageにソースファイルをアップロードする
3. コンバートジョブをトリガーするためREST APIをたたく
4. コンバートジョブの進捗を確認するためREST APIをたたく

## Blob Storageを用意する

`Azure Storage Account`を準備すればOKです。StorageV2でもBlob Storage onlyでもどちらでも大丈夫です。

## Blob Storageにソースファイルをアップロードする

方法は問いません。[Storage Explorer](https://azure.microsoft.com/ja-jp/features/storage-explorer/)からアップロードしてもいいし、REST APIからアップロードしてもOKです。

ドキュメントをみると、入力となるソースファイルを格納するコンテナと出力となるAAR用アセットを格納するコンテナを用意するように書いてありますが、同じコンテナにしても問題ありません。

## コンバートジョブをトリガーするため、REST APIを実行する

### アクセストークンの取得

model conversion REST APIにアクセスするためにはアクセストークンが必要となります。アクセストークンは専用のエンドポイントが用意されているのでそちらから取得できます。アクセストークン取得のためにはAARのAccountIDとAccess Keyが必要となります。（[確認方法はこちらのキャプチャを参考に](https://docs.microsoft.com/en-us/azure/remote-rendering/how-tos/create-an-account#create-an-account)）

---

* ドメイン
  * sts.mixedreality.azure.com
* エンドポイント
  * accounts/{AARのAccountID}/token
* メソッド
  * GET
* 必要なヘッダ
  * Authorization
    * Bearer {AARのAccountID}:{AARのAccessKey}


[リファレンス](https://docs.microsoft.com/en-us/azure/remote-rendering/how-tos/tokens#token-service-rest-api)

---

### コンバート用APIの選択

コンバート用のAPIは2つ用意されています。出力されるAAR用アセットに違いはなく、AARからBlob Storageへのアクセス方法の選択によって変わります。アクセス方法としてはIAMを使う方法とSASを使う方法があります。

--- 

### IAMを使ってアクセスする

Storage Accountに対して、`Owner`、`Storage Account Contributor`、`Storage Blob Data Contributor`のロールをAARのManaged Identityに割り当てます。[設定方法はこちら](https://docs.microsoft.com/en-us/azure/remote-rendering/how-tos/create-an-account#link-storage-accounts)。

設定完了後、APIを叩くとAzure上でコンバートが開始されます。

---

* ドメイン
  * 以下の4つから選ぶ
    * remoterendering.southeastasia.mixedreality.azure.com
    * remoterendering.eastus.mixedreality.azure.com
    * remoterendering.westeurope.mixedreality.azure.com
    * remoterendering.westus2.mixedreality.azure.com
* エンドポイント
  * /v1/accounts/{AARのAccountID}/conversions/create
* メソッド
  * POST
* 必要なヘッダ
  * Authorization
    * Bearer {アクセストークン}

リクエストボディのフォーマットは以下の通り

```json
{
    "input":
    {
        "storageAccountname": "ソースファイルが置いてあるストレージアカウントの名前",
        "blobContainerName": "ソースファイルが置いてあるコンテナの名前",
        "folderPath": "ソースファイルが置いてあるコンテナの中のフォルダの名前（オプション）",
        "inputAssetPath" : "ソースファイル名"
    },
    "output":
    {
        "storageAccountname": "AAR用アセットの出力先のストレージアカウントの名前",
        "blobContainerName": "AAR用アセットの出力先のコンテナの名前",
        "folderPath": "AAR用アセットの出力先のフォルダ名（オプション）",
        "outputAssetFileName": "AAR用アセットのファイル名。拡張子を`.aarAsset`にする必要がある"
    }
}
```

レスポンスとしてコンバートジョブのIDが返ってきます。コンバートの進捗を確認するときに必要となります。

```json
{
    "conversionId": "GUID"
}
```

[リファレンス](https://docs.microsoft.com/en-us/azure/remote-rendering/how-tos/conversion/conversion-rest-api)

---

### SASを使ってアクセスする

[SAS](https://docs.microsoft.com/ja-jp/azure/storage/common/storage-sas-overview)とはStorage Account内のデータへアクセスするための時限式トークンみたいなものです。こちらもStorage ExplorerやREST APIから取得することが可能です。

ソースファイルと出力先のSASをそれぞれ取得したあと、それを使ってAPIを叩きます。

---

* ドメイン
  * 以下の4つから選ぶ
    * remoterendering.southeastasia.mixedreality.azure.com
    * remoterendering.eastus.mixedreality.azure.com
    * remoterendering.westeurope.mixedreality.azure.com
    * remoterendering.westus2.mixedreality.azure.com
* エンドポイント
  * /v1/accounts/{AARのccountID}/conversions/createWithSharedAccessSignature
* メソッド
  * POST
* 必要なヘッダ
  * Authorization
    * Bearer {アクセストークン}

リクエストボディのフォーマットは以下の通り

```json
{
    "input":
    {
        "storageAccountname": "ソースファイルが置いてあるストレージアカウントの名前",
        "blobContainerName": "ソースファイルが置いてあるコンテナの名前",
        "folderPath": "ソースファイルが置いてあるコンテナの中のフォルダの名前（オプション）",
        "inputAssetPath" : "ソースファイル名",
        "containerReadListSas" : "ソースファイルのSAS"
    },
    "output":
    {
        "storageAccountname": "AAR用アセットの出力先のストレージアカウントの名前",
        "blobContainerName": "AAR用アセットの出力先のコンテナの名前",
        "folderPath": "AAR用アセットの出力先のフォルダ名（オプション）",
        "outputAssetFileName": "AAR用アセットのファイル名。拡張子を`.aarAsset`にする必要がある",
        "containerWriteSas" : "出力先のSAS"
    }
}
```

レスポンスとしてコンバートジョブのIDが返ってきます。コンバートの進捗を確認するときに必要となります。

```json
{
    "conversionId": "GUID"
}
```

[リファレンス](https://docs.microsoft.com/en-us/azure/remote-rendering/how-tos/conversion/conversion-rest-api)

---

## コンバートジョブの進捗を確認するためREST APIを実行する

コンバートの進捗を確認するためのREST APIも用意されています。

---

* ドメイン
  * 以下の4つから、コンバートジョブを実行したときと同じものを選ぶ
    * remoterendering.southeastasia.mixedreality.azure.com
    * remoterendering.eastus.mixedreality.azure.com
    * remoterendering.westeurope.mixedreality.azure.com
    * remoterendering.westus2.mixedreality.azure.com
* エンドポイント
  * /v1/accounts/{AARのAccountID}/conversions/{conversionId}
    * `conversionId`はジョブ実行時のレスポンスから取得したもの
* メソッド
  * GET
* 必要なヘッダ
  * Authorization
    * Bearer {アクセストークン}

レスポンスとしてコンバートの進捗が文字列で返ってきます。

```json
{
    "status": "'Created' | 'Running' | 'Success' | 'Failure' "
    "error": "string or null"
}
```

コンバートが完了すると、`Success`か`Failure`が返ってきます。

[リファレンス](https://docs.microsoft.com/en-us/azure/remote-rendering/how-tos/conversion/conversion-rest-api)

---

# まとめ

AAR用のアセットを作る手順をまとめました。まだ各言語用のSDKは用意されていませんが、コンバートに必要なREST APIの種類も少なくシンプルなので実装に困ることはそれほどないかなと思います。今回作ったプロトタイプもNode.js + TypeScriptで[axios](https://github.com/axios/axios)を使って実装しました。

# 参考

[Convert models - MS公式ドキュメント](https://docs.microsoft.com/en-us/azure/remote-rendering/how-tos/conversion/model-conversion)  
[Azure Remote Rendering - Hiromuブログ ](https://www.hiromukato.com/entry/2020/04/17/000504)