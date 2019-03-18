---
title: "Azure DevOps Pipelinesを使ってみる"
date: 2019-03-11T15:52:15+09:00
draft: true
categories: ["Azure"]
tags: ["Azure", "Azure DevOps", "Pipelines", "Storage Accout"]
---

# はじめに

今回はAzure DevOps Pipelinesを使って、Azureサブスクリプション上に新しいストレージアカウントを作って見ようと思います。

# Azure DevOpsとAzureサブスクリプションを紐づける
https://docs.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=azure-devops

サービスプリンシパルを作っておきましょう
az login
az ad sp create-for-rbac -n [なにか]

Project Settings→Pipelines→Service connectionsからNew service connection→Azure Resource Managerを選択する

Connection Name:タスクからAzureサブスクリプションを参照するための名前？
Environment: AzureCloud
Scope level:管理グループかサブスクリプションかを選ぶ
SubscriptionID:サブスクリプションID
Subscription name:サブスクリプション名
Service principal client ID
Service principal key
Tenant ID
Verify Connectionでチェックしておくといい

# インフラ構築コードを作る
## テンプレートの定義
## パラメータの定義
## シェルスクリプトの定義

# Azure DevOps Pipelinesの設定
Pipelines→ReleasesからNew→New release pipelineを選択する
Artifactsはリポジトリを選択する
StagesはEmptyJobを選択する
シェルスクリプト(.sh)が実行できるエージェントを選択する。ここではUbuntuを選択した。
Azure CLI TaskをJOBに追加する

SubScriptionはドロップボックスで選択（うまく紐付けできていれば）
今回はシェルスクリプトを作ってGitで管理しているので、Script Pathを選択し、アーティファクトから対象のシェルスクリプトを選択する
シェルスクリプトのコマンドライン引数を選択する
