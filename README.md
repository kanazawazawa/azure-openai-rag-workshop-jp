<!-- prettier-ignore -->

下記を複製して日本語化し簡略化して作成しております。  
本リポジトリご利用の際は下記リンクのドキュメント、ライセンス等をご確認ください。  
https://github.com/Azure-Samples/azure-openai-rag-workshop



<div align="center">

# 🤖 Azure OpenAI RAG ワークショップ - Node.js バージョン

[![Open project in GitHub Codespaces](https://img.shields.io/badge/Codespaces-Open-blue?style=flat-square&logo=github)](https://codespaces.new/Azure-Samples/azure-openai-rag-workshop?hide_repo_select=true&ref=main&quickstart=true)
![Node version](https://img.shields.io/badge/Node.js->=20-3c873a?style=flat-square)
[![Ollama + Mistral](https://img.shields.io/badge/Ollama-Mistral-ff7000?style=flat-square)](https://ollama.com/library/mistral)
[![TypeScript](https://img.shields.io/badge/TypeScript-blue?style=flat-square&logo=typescript&logoColor=white)](https://www.typescriptlang.org)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

:star: このサンプルが気に入ったら、GitHubでスターを付けてください — 大変助かります！

[概要](#概要) • [サンプルの実行](#サンプルの実行) • [他のバージョン](#他のバージョン) • [参考資料](#参考資料)

</div>

このサンプルは、LangChain.jsとOpenAI言語モデルを使用して、検索強化生成（RAG）を用いたAIチャット体験を構築する方法を示しています。アプリケーションは[Azure Static Web Apps](https://learn.microsoft.com/azure/static-web-apps/overview)と[Azure Container Apps](https://learn.microsoft.com/azure/container-apps/overview)にホストされ、ベクトルデータベースとして[Azure AI Search](https://learn.microsoft.com/azure/search/search-what-is-azure-search)を使用しています。より複雑なAIアプリケーションを構築するための出発点として使用できます。

> [!IMPORTANT]
> 👉 **このサンプルの構築方法と実行およびデプロイ方法を学ぶには、[フルレングスワークショップ](https://aka.ms/ws/openai-rag)に従ってください。**

## 概要

このサンプルは、[Fastify](https://fastify.dev)を使用して、[OpenAI SDK](https://platform.openai.com/docs/libraries/)と[LangChain](https://js.langchain.com/)を活用したチャットボットを構築する[Node.js](https://nodejs.org/)サービスを作成します。このチャットボットは、ドキュメントのコーパスに基づいて質問に回答します。APIと対話するためのウェブサイトも含まれています。

このプロジェクトはモノレポとして構成されており、すべてのパッケージのソースコードは`src/`フォルダーにあります。
アプリケーションのアーキテクチャは次のとおりです：

<div align="center">
  <img src="./docs/assets/architecture.png" alt="アーキテクチャ図" width="640px" />
</div>

## サンプルの実行

[GitHub Codespaces](https://github.com/features/codespaces)を使用して、このプロジェクトをブラウザから直接作業できます：

[![Open in GitHub Codespaces](https://img.shields.io/badge/Codespaces-Open-blue?style=flat-square&logo=github)](https://codespaces.new/Azure-Samples/azure-openai-rag-workshop?hide_repo_select=true&ref=main&quickstart=true)

また、[Docker](https://www.docker.com/products/docker-desktop)と[VS CodeのDev Containers拡張機能](https://aka.ms/vscode/ext/devcontainer)を使用して、ローカルで準備済みの開発環境を使用して作業することもできます：

[![Open in Dev Containers](https://img.shields.io/static/v1?style=flat-square&label=Dev%20Containers&message=Open&color=blue&logo=visualstudiocode)](https://vscode.dev/redirect?url=vscode://ms-vscode-remote.remote-containers/cloneInVolume?url=https://github.com/Azure-Samples/azure-openai-rag-workshop)

すべてのツールをローカルにインストールすることを好む場合は、これらの[セットアップ手順](https://aka.ms/ws?src=gh%3AAzure-Samples%2Fazure-openai-rag-workshop%2Fdocs%2Fworkshop.md&step=2#optional-working-locally-without-the-dev-container)に従ってください。

<!--
> [!TIP]
> [Ollama](https://ollama.com/)を使用して、完全にローカルでこのサンプルを実行できます。ローカルでツールをセットアップする手順に従って開始してください。
-->

### Azureの前提条件

- **Azureアカウント**。Azureを初めて使用する場合は、[無料のAzureアカウントを取得](https://azure.microsoft.com/free)して、無料のAzureクレジットを取得して開始できます。学生の場合は、[Azure for Students](https://aka.ms/azureforstudents)で無料のクレジットを取得することもできます。
- **Azure OpenAIサービスへのアクセスが有効なAzureサブスクリプション**。この[フォーム](https://aka.ms/oaiapply)でアクセスをリクエストできます。
- **Azureアカウントの権限**：
  - Azureアカウントには、[ロールベースアクセス制御管理者](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles#role-based-access-control-administrator-preview)、[ユーザーアクセス管理者](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles#user-access-administrator)、または[所有者](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles#owner)などの`Microsoft.Authorization/roleAssignments/write`権限が必要です。サブスクリプションレベルの権限がない場合は、既存のリソースグループに対して[RBAC](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles#role-based-access-control-administrator-preview)を付与され、その既存のグループに[デプロイする](docs/deploy_existing.md#resource-group)必要があります。
  - Azureアカウントには、サブスクリプションレベルで`Microsoft.Resources/deployments/write`権限も必要です。

### サンプルのデプロイ

ターミナルを開き、次のコマンドを実行します：

```bash
azd auth login
azd up
```

これらのコマンドは、最初にAzureにログインするように求めます。その後、Azureリソースをプロビジョニングし、サービスをパッケージ化してAzureにデプロイします。

### クリーンアップ

このサンプルによって作成されたすべてのAzureリソースをクリーンアップするには：

1. `azd down --purge`を実行します
2. 続行するかどうかを尋ねられたら、`y`と入力します

リソースグループとすべてのリソースが削除されます。

## 他のバージョン

このサンプルとワークショップには、さまざまなバージョンがあります：

- [**Node.js + Azure AI Search**](https://aka.ms/ws/openai-rag)
- [**Node.js + Qdrant**](https://aka.ms/ws/openai-rag-qdrant)
- [**Java/Quarkus + Qdrant**](https://aka.ms/ws/openai-rag-quarkus)

## 参考資料

このサンプルで使用されている技術について学ぶためのリソースをいくつか紹介します：

- [LangChain.js ドキュメント](https://js.langchain.com/)
- [Generative AI For Beginners](https://github.com/microsoft/generative-ai-for-beginners)
- [Azure OpenAI Service](https://learn.microsoft.com/azure/ai-services/openai/overview)

他の[Azure AI サンプルはこちら](https://github.com/Azure-Samples/azureai-samples)で見つけることができます。

このサンプル/ワークショップは、エンタープライズ対応のサンプル**ChatGPT + Enterprise data with Azure OpenAI and AI Search**に基づいています：

- [JavaScript バージョン](https://github.com/Azure-Samples/azure-search-openai-javascript) / [サーバーレス JavaScript バージョン](https://github.com/Azure-Samples/serverless-chat-langchainjs)
- [Python バージョン](https://github.com/Azure-Samples/azure-search-openai-demo/)
- [Java バージョン](https://github.com/Azure-Samples/azure-search-openai-demo-java)
- [C# バージョン](https://github.com/Azure-Samples/azure-search-openai-demo-csharp)

より高度なユースケース、認証、履歴などを使用してさらに進みたい場合は、ぜひチェックしてください！

## コントリビューション

本リポジトリは、下記を複製して日本語化し簡略化して作成しております。  
コントリビューションについては下記リンクをご確認いただき、リンク先リポジトリに対してご提案などをお願いします。
https://github.com/Azure-Samples/azure-openai-rag-workshop


## 商標

本リポジトリは、下記を複製して日本語化し簡略化して作成しております。  
ご利用の際は下記リンクの商標に関する記載をご確認ください。  
https://github.com/Azure-Samples/azure-openai-rag-workshop
