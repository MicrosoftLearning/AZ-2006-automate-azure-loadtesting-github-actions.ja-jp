---
lab:
  title: 'ラボ 02: GitHub Actions for Azure を使用して Web アプリを Azure App Service に公開する'
  module: 'Module 2: Implement GitHub Actions for Azure'
---

# 概要

このラボでは、Web アプリを Azure App Service にデプロイする GitHub Actions ワークフローを実装する方法を学習します。

このラボを完了すると、次のことができるようになります。

* CI/CD のために GitHub Actions ワークフローを実装する。
* GitHub Actions ワークフローの基本的な特性について説明する。

**推定完了時間: 40 分**

## 前提条件

* アクティブなサブスクリプションがある **Azure アカウント**。 アカウントを取得済みでない場合は、[https://azure.com/free](https://azure.com/free) から無料評価版にサインアップできます。
    * Azure Web ポータルでサポートされている[ブラウザー](https://learn.microsoft.com/azure/azure-portal/azure-portal-supported-browsers-devices)。
    * Azure サブスクリプションの共同作成者または所有者ロールを持つ Microsoft アカウントまたは Microsoft Entra アカウント。 詳細については、[「Azure portal を使用して Azure ロールの割り当てを一覧表示する」](https://docs.microsoft.com/azure/role-based-access-control/role-assignments-list-portal)および[「Azure Active Directory で管理者ロールを表示して割当てる」](https://docs.microsoft.com/azure/active-directory/roles/manage-roles-portal)を参照してください。
* GitHub アカウント。 このラボで使用できる GitHub アカウントをまだお持ちでない場合は、「[新しい GitHub アカウントのサインアップ](https://github.com/join)」にある手順に従ってアカウントを作成してください。

## 手順

## 演習 1: eShopOnWeb を GitHub リポジトリにインポートする

この演習では、[eShopOnWeb](https://github.com/MicrosoftLearning/eShopOnWeb) リポジトリを GitHub アカウントにインポートします。 リポジトリは次のように編成されています。

| フォルダー | Contents |
| -- | -- |
| **.ado** | Azure DevOps YAML Pipelines |
| **.devcontainer** | コンテナーを使って開発するための構成 (VS Code でローカルに、または GitHub Codespaces で) |
| **インフラストラクチャ** | 一部のラボ シナリオで使用されるコード テンプレートとしての Bicep と ARM |
| **.github** | YAML GitHub ワークフロー定義 |
| **src** | ラボ シナリオで使用される .NET 8 Web サイト |

### タスク 1: eShopOnWeb リポジトリをインポートする

1. Web ブラウザーで GitHub [http://github.com](http://github.com) に移動し、アカウントを使ってサインインします。
1. インポート プロセスの [https://github.com/new/import](https://github.com/new/import)を開始します。
1. **プロジェクトを GitHub にインポート** ページに次の情報を入力します。

    | 設定 | アクション |
    |--|--|
    | **ソース リポジトリの URL** | 「`https://github.com/MicrosoftLearning/eShopOnWeb`」と入力します |
    | **所有者** | GitHub エイリアスを選択する |
    | **リポジトリ名** | **eShopOnWeb** を入力 |
    | **プライバシー** | **オーナー**を選択するとプライバシー オプションが表示されます。 **[パブリック]** を選択します。 |

1. **インポートの開始** を選択し、インポート プロセスが完了するまで待ちます。
1. リポジトリ ページで **[設定]** を選択し、左側のナビゲーション ウィンドウで **[アクション] > [全般]** を選択します。
1. ページの **[アクションのアクセス許可]** セクションで、**[すべてのアクションと再利用可能なワークフローを許可する]** オプションを選択し、**[保存]** を選択します。

> **注:** eShopOnWeb は大規模なリポジトリであり、インポートが完了するまでに 5 ~ 10 分かかる場合があります。

## 演習 2: Azure リソースを作成し GitHub を構成する 

この演習では、GitHub Actions から Azure サブスクリプションにアクセスする GitHub を承認するための Azure サービス プリンシパルを作成します。 また、Web サイトをビルド、テストし、Azure にデプロイする GitHub ワークフローも確認および変更します。

### タスク 1: Azure サービス プリンシパルを作成し、GitHub シークレットとして保存する

このタスクでは、リソース グループと Azure サービス プリンシパルを作成します。 サービス プリンシパルは、目的の eShopOnWeb アプリをデプロイするために GitHub によって使用されます。

1. ブラウザーで Azure portal [https://portal.azure.com](https://portal.azure.com) に移動します。
1. **Cloud Shell** を開き、**Bash** モードを選択します。 **注:** Cloud Shell を初めて起動した場合は、永続ストレージを構成する必要があります。
1. 次の `az group create` CLI コマンドを使用してリソース グループを作成します。 `<location>` を最寄りのリージョンに置き換えます。

    ```
    az group create -n az2006-rg -l <location>
    ```

1. 次のコマンドを実行して、ラボの後半でデプロイする **Azure App Service** のリソース プロバイダーを登録します。

    ```bash
    az provider register --namespace Microsoft.Web
    ```

1. 次のコマンドを実行して、Azure App Service にデプロイする Web アプリのランダムな名前を生成します。 そのラボの後半で使用するために、コマンド出力の名前をコピーして保存します。

    ```
    myAppName=az2006app$RANDOM
    echo $myAppName
    ```

1. 次のコマンドを実行して、サブスクリプション ID を取得します。 サブスクリプション ID の値はこのラボの後半で使用するため、コマンドからの出力を必ずコピーして保存しておいてください。

    ```
    subId=$(az account list --query "[?isDefault].id" --output tsv)
    
    echo $subId
    ```

1. 次のコマンドを使用して、サービス プリンシパルを作成します。 最初のコマンドは、リソース グループの ID を変数に格納します。

    ```
    rgId=$(az group show -n az2006-rg --query "id" -o tsv)

    az ad sp create-for-rbac --name GH-Action-eshoponweb --role contributor --scopes $rgId
    ```

    >**重要:** このコマンドは、Microsoft Entra ID (サービス プリンシパル) の名前で Azure に対する認証に使用される識別子が含まれる JSON オブジェクトを出力します。 次の手順で使用するために、JSON オブジェクトをコピーします。 

1. ブラウザー ウィンドウで、**eShopOnWeb** GitHub リポジトリに移動します。
1. リポジトリ ページで **[設定]** を選択し、左側のナビゲーション ウィンドウで **[シークレットと変数] - [アクション]** を選択します。
1. **[新しいリポジトリ シークレット]** を選択し、次の情報を入力します。
    * **名前**: `AZURE_CREDENTIALS`
    * **シークレット**: サービス プリンシパルの作成時に生成された JSON オブジェクトを入力します。
1. **[Add secret](シークレットの追加)** を選択します。

### タスク 2: GitHub ワークフローを変更して実行する

このタスクでは、指定された *eshoponweb-cicd.yml* GitHub ワークフローを変更し、それを実行して独自のサブスクリプションにソリューションをデプロイします。

1. ブラウザー ウィンドウで、**eShopOnWeb** GitHub リポジトリに戻ります。
1. **<> コード**を選択して、メイン ブランチで **eShopOnWeb/.github/workflows** フォルダー内の **eshoponweb-cicd.yml** を選択します。 このワークフローでは、eShopOnWeb アプリ用の CI/CD プロセスが定義されています。

    ![フォルダー構造内のファイルの場所を示すスクリーンショット。](./media/eshop-cid-workflow.png)
1. **[このファイルの編集]** を選択します。
1. ファイルの `env:` セクションのフィールドを、次の値に変更します。

    | フィールド | アクション |
    |--|--|
    | RESOURCE-GROUP: | `az2006-rg` |
    | LOCATION: | `eastus` (または、リソース グループの作成時に選択したリージョン)。 |
    | TEMPLATE-FILE: | 変更なし |
    | SUBSCRIPTION-ID: | サブスクリプション ID。 |
    | WEBAPP-NAME: | ラボの前半で作成したランダムに生成された Web アプリ名。 |

1. ワークフローを注意深くお読みください。ワークフローの手順の理解に役立つコメントが記載されています。
1. `#` を削除して、ファイルの上部にある **on** セクションのコメントを解除します。 このワークフローはメイン ブランチへのプッシュのたびにトリガーされ、手動トリガー (`workflow_dispatch`) も実行できます。
1. ページの右上にある **[変更をコミットする...]** を選択します。
1. ポップアップ ウィンドウが表示されます。 既定値 (メイン ブランチに直接コミットする) をそのまま使用し、**[変更をコミットする]** を選択します。 ワークフローが自動的に実行されます。

### タスク 3: GitHub ワークフローの実行を確認する

このタスクでは、GitHub ワークフローの実行を確認して、実行中のアプリケーションを表示します。

1. **[アクション]** を選択すると、実行前のワークフローの設定が表示されます。

1. ページの **[すべてのワークフロー]** セクションで、**[eShopOnWeb Build and Test]** を選択します。 

1. ワークフローは、**buildandtest** と **deploy** の 2 つの操作で構成されます。 いずれかの操作を選択して進行状況を表示するか、ジョブが完了するまで待機することができます。

1. Azure portal [https://portal.azure.com](https://portal.azure.com) に移動し、先ほど作成した **az2006-rg** リソース グループに移動します。 bicep テンプレートを使用した GitHub アクションによって、Azure App Service プランと App Service が作成されている点に注意してください。 

1. App Service リソース (先ほど生成した一意のアプリ名) を選択し、ページの上部付近にある **[参照]** を選択して、デプロイされた Web アプリを表示します。

## 演習 3: リソースをクリーンアップする

この演習では、ラボの前半で作成したリソースを削除します。

1. Azure portal [https://portal.azure.com](https://portal.azure.com) に移動して、Cloud Shell を開始します。 **Bash** シェル セッションを選択します。

1. 次のコマンドを実行して、`az2006-rg` リソース グループを削除します。 また、App Service プランと App Service インスタンスも削除されます。

    ```
    az group delete -n az2006-rg --no-wait --yes
    ```

    >**注:** コマンドは非同期に実行されるので (`--no-wait` パラメーターで設定される)、同じ Bash セッション内ですぐに別の Azure CLI コマンドを実行できますが、リソース グループが実際に削除されるまでに数分かかります。

## 確認

このラボでは、Azure Web アプリをデプロイする GitHub アクション ワークフローを実装しました。
