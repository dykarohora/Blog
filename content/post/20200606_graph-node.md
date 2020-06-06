---
title: "Node.jsとTypeScriptでMicrosoft Graph APIを使う"
date: 2020-06-06T10:15:10+09:00
slug: "20200606-graph-node"
author: "d_yama"
draft: false
categories: ["Azure"]
tags: ["TypeScript", "Azure", "Microsoft Graph"]
---

# サーバサイドでもGraph APIを使いたい

TypeScript/JavaScriptでGraph APIを使うときは[@microsoft/microsoft-graph-client](https://www.npmjs.com/package/@microsoft/microsoft-graph-client)を使うのがスタンダードかと思います。こちらのnpmパッケージはクライアントサイドでもサーバサイドでも利用できるものですが、世の中にあるチュートリアルはクライアントサイドでの利用を想定したものが多い気がします。[README](https://github.com/microsoftgraph/msgraph-sdk-javascript#readme)のチュートリアルを見ても、[「サーバサイドで使うときはAuthentication Providerを自分で作ってね」](https://github.com/microsoftgraph/msgraph-sdk-javascript/blob/dev/docs/CustomAuthenticationProvider.md)と書いてあるだけで、ちょっと寂しい状態です。

とはいえサーバサイド(Node.js)でGraph APIを使いたいケースもあると思うので、今回はNode.js(v12.18.0) + TypeScriptでの使い方を紹介します。

# 事前準備

## サービスプリンシパルの作成
Graph APIにアクセスするためにはOAutn2.0におけるアクセストークンが必要です。今回はサーバサイドアプリケーションからアクセスするケースとなりますので、サービスプリンシパルを使ってアクセストークンを取得します。

[サービスプリンシパルの作成方法はこちら](https://docs.microsoft.com/ja-jp/azure/active-directory/develop/quickstart-register-app)

サービスプリンシパルがGraph APIにアクセスするためには明示的な許可が必要です。サービスプリンシパルやAPIの許可についてはこちらの記事が大変参考になります。

[Azure AD のサービスプリンシパルを 3 つのユースケースから眺めてみた](https://qiita.com/watahani/items/1f3f533097b7a15d6698)

## npmパッケージのインストール

必要なパッケージはアクセストークンを取得するための[@azure/ms-rest-nodeauth](https://www.npmjs.com/package/@azure/ms-rest-nodeauth)と、Graph APIにアクセスするための[@microsoft/microsoft-graph-client](https://www.npmjs.com/package/@microsoft/microsoft-graph-client)です。

```
yarn add @microsoft/microsoft-graph-client @azure/ms-rest-nodeauth
```

# コード

Azure AD上の指定したグループに所属するユーザの情報を取得する、という利用シーンのサンプルです。

```typescript
import { Client } from '@microsoft/microsoft-graph-client'
import { loginWithServicePrincipalSecret } from '@azure/ms-rest-nodeauth'

async function getGroupMembers() {
    // サービスプリンシパルのアプリケーションID
    const clientId = process.env.AZURE_CLIENT_ID
    // サービスプリンシパルのシークレット
    const clientSecret = process.env.AZURE_CLIENT_SECRET
    // Azure ADのテナントID
    const tenantId = process.env.AZURE_TENANT_ID
    // 検索対象のAzure AD セキュリティグループのオブジェクトID
    const groupId = process.env.AZURE_USER_GROUP_ID

    const applicationTokenCredentials = await loginWithServicePrincipalSecret(
        clientId, 
        clientSecret, 
        tenantId, 
        {tokenAudience: 'https://graph.microsoft.com'}
    )

    const getAccessTokenFunc = async (): Promise<string> => {
        const tokenResponse = await applicationTokenCredentials.getToken()
        return tokenResponse.accessToken
    }

    const graphClient = Client.initWithMiddleware({
        authProvider: {
            getAccessToken: getAccessTokenFunc
        }
    })

    const result = await graphClient
        .api(`/groups/${groupId}/members`)
        .select(['id', 'displayName'])
        .top(200)
        .get()
}
```

[Authentication Provider](https://github.com/microsoftgraph/msgraph-sdk-javascript/blob/dev/docs/CustomAuthenticationProvider.md)とは、要はアクセストークンを返す関数を持つオブジェクトのインタフェースです。なのでGraph APIにアクセスできるアクセストークンさえ用意できれば、その実装は特に問われません。

今回はサーブスプリンシパルを使い[OAuth2.0のClient Credentials Grantフロー](https://docs.microsoft.com/ja-jp/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow)でアクセストークンを取得します。アクセストークンを管理するため、[@microsoft/microsoft-graph-client](https://www.npmjs.com/package/@microsoft/microsoft-graph-client)の`loginWithServicePrincipalSecret`関数を使います。

```typescript
    const applicationTokenCredentials = await loginWithServicePrincipalSecret(
        clientId, 
        clientSecret, 
        tenantId, 
        {tokenAudience: 'https://graph.microsoft.com'}
    )
```

オプションオブジェクトの`tokenAudience`にMicrosoft GraphのリソースURIを指定する必要があります。一応、ライブラリ側でも`graph`というオプションが用意されています。しかしこのオプションは古いリソースURIと結びついているのでこれを指定してもGraph APIへアクセスするためのアクセストークンは取得できないので注意してください。

![extension](/image/20200606_02.png)

`loginWithServicePrincipalSecret`関数の戻り値である`ApplicationTokenCredentials`オブジェクトは内部でアクセストークンとリフレッシュトークンをキャッシュしています。そのアクセストークンを取得するメソッドとして`getToken`メソッドが用意されています。こちらはキャッシュしているアクセストークンが有効期限以内ならそのアクセストークンを、有効期限を過ぎている場合はリフレッシュトークンを使って新しいアクセストークンを返してくれます。今回はそのメソッドを使って`Authentication Provider`インタフェースを満たすオブジェクトを構成し、`Client`オブジェクトの初期化メソッドに渡しています。

```typescript
    const getAccessTokenFunc = async (): Promise<string> => {
        const tokenResponse = await applicationTokenCredentials.getToken()
        return tokenResponse.accessToken
    }

    const graphClient = Client.initWithMiddleware({
        authProvider: {
            getAccessToken: getAccessTokenFunc
        }
    })
```

今回は一つの関数内にすべて記述しましたが、`Client`オブジェクトをキャッシュしておくことも可能です。先ほど説明したとおり、`Client`オブジェクトの初期化メソッドに渡した`Authentication Provider`はアクセストークンの有効期限が切れたら自動的に新しいアクセストークンを取得してくれるので、Graph APIのアクセス毎に`Client`オブジェクトを作り直す必要はありません。

ここまで来たら、あとはチュートリアルにあるような使い方と同じ書き方でGraph APIを操作できます。

```typescript
    const result = await graphClient
        .api(`/groups/${groupId}/members`)
        .select(['id', 'displayName'])
        .top(200)
        .get()
```

# まとめ

Node.jsからGraph APIへのアクセスする方法について、TypeScriptでの実装とあわせて紹介しました。