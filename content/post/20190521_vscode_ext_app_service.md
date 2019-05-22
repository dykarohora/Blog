---
title: "VSCode拡張 Azure App Serviceを使ってWebアプリをリモートデバッグしてみた"
slug: "20190521_vscode_ext_app_service"
author: "d_yama"
date: 2019-05-22T11:30:16+09:00
draft: false
categories: ["Azure"]
tags: ["Azure", "App Service", "Visual Studio Code", "Node.js"]
---

# はじめに
Node.jsで書いたWebアプリをAzure AppService上で動かすことが多いのですが、デプロイしたときの挙動を確認したい時にはWebSSHで繋いでログ確認して…、みたいなことがまどろっこしいので、もう少し良い方法はないものかと考えていたところ、使えそうなVSCode拡張を見つけましたので使ってみました。

[Azure App Service](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azureappservice)

# 手順
## テスト用のアプリを作る
Node.jsでHello, Worldをテキストで返すだけの簡単なアプリを作りました。
```javascript
const http = require('http');

const server = http.createServer((request, response) => {
    response.writeHead(200, {"Content-Type": "text/plain"});
    response.write("Hello, World!");
    response.end();
});

const port = process.env.PORT || 1337;
server.listen(port);
```

## Azureにデプロイする
Azure CLIでAzureにログインします。
```
az login
```

リソースグループを作ります。
```
az group create --name "<YourResourceGroupName>" --location "Japan East"
```

AppService Planを作ります。Windowsだと拡張機能の方がサポートしていないようだった(2019/5/21時点)ので、Linuxで作りました。
```
az appservice plan create --name "<YourAppServicePlanName>" --resource-group "<YourResourceGroupName>" --is-linux
```

NodeでAppServiceを作ります。デプロイ方法はlocal-gitにしました。
```
az webapp create --resource-group "<YourResourceGroupName>" --plan "<YourAppServicePlanName>" --name "<YourAppName>" --runtime "NODE|10.14" --deployment-local-git
```

上のコマンドの結果、`deploymentLocalGitUrl`にリポジトリのURLが記載されているので、そちらに作ったアプリをpushすればデプロイが始まります。完了したら`https://<YourAppName>.azurewebsites.net/`にアクセスしてみて、Hello, Wolrdが表示されればデプロイ完了です。

## デバッグする
以下のVSCode拡張をインストールします。  
[Azure App Service](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azureappservice)

そうするとVSCodeの左にAzureのロゴが追加されますので、こちらのウィンドウを開くとAzure上にデプロイされているAppServiceの一覧が表示されます。デバッグしたいAppServiceを右クリックし`Start Remote Debuggig`を選択すればデバッグが始まります。
![extension](/image/20190521_vscode.png)

作成したコード上にブレークポイントを貼っておけば、AppServiceにアクセスしたときに該当コードで停止するようになります。
{{< youtube q2Ac1DVL0UM >}}

## まとめ
VSCode拡張を使ってAzure AppService上にデプロイしたWebアプリのリモートデバッグを試してみました。ローカル開発環境ではちゃんと動くのにクラウドリソース上だと動作しない！というケースは少なくないので、挙動を確認するのには役立つと思います。

また、デバッグだけでなくコンテナ上に記録されているログもVSCodeで見ることができるのも何気に便利です。
