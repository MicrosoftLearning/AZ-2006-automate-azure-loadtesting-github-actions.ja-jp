---
lab:
  title: 'ラボ 03: GitHub Actions を使用して Azure Load Testing を自動化する '
  module: 'Module 3: Implement Azure Load Testing'
---

# 概要

この演習では、サンプル Web アプリをデプロイし、Azure Load Testing を使用してロード テストを開始するように GitHub Actions を構成する方法について説明します。

このラボでは、次のことを行います。

* Azure で App Service とロード テストのリソースを作成します。
* GitHub Actions ワークフローが Azure アカウントでアクションを実行できるように、サービス プリンシパルを作成して構成します。
* GitHub Actions ワークフローを使用して、.NET 8 アプリケーションを Azure App Service にデプロイします。
* GitHub Actions ワークフローを更新して、URL ベースのロード テストを呼び出します。

**推定完了時間: 40 分**

## 前提条件

* アクティブなサブスクリプションがある **Azure アカウント**。 アカウントを取得済みでない場合は、[https://azure.com/free](https://azure.com/free) から無料評価版にサインアップできます。
    * Azure Web ポータルでサポートされている[ブラウザー](https://learn.microsoft.com/azure/azure-portal/azure-portal-supported-browsers-devices)。
    * Azure サブスクリプションの共同作成者または所有者ロールを持つ Microsoft アカウントまたは Microsoft Entra アカウント。 詳細については、[「Azure portal を使用して Azure ロールの割り当てを一覧表示する」](https://docs.microsoft.com/azure/role-based-access-control/role-assignments-list-portal)および[「Azure Active Directory で管理者ロールを表示して割当てる」](https://docs.microsoft.com/azure/active-directory/roles/manage-roles-portal)を参照してください。
* GitHub アカウント。 このラボで使用できる GitHub アカウントをまだお持ちでない場合は、「[新しい GitHub アカウントのサインアップ](https://github.com/join)」にある手順に従ってアカウントを作成してください。


## 手順

## 演習 1: サンプル アプリを GitHub リポジトリにインポートする

この演習では、[Azure ロード テスト サンプル アプリ](https://github.com/MicrosoftLearning/azure-load-test-sample-app) リポジトリを独自の GitHub アカウントにインポートします。

### タスク 1: eShopOnWeb リポジトリをインポートする

1. Web ブラウザーで GitHub [http://github.com](http://github.com) に移動し、アカウントを使ってサインインします。
1. インポート プロセスの [https://github.com/new/import](https://github.com/new/import)を開始します。
1. **プロジェクトを GitHub にインポート** ページに次の情報を入力します。

    | 設定 | アクション |
    |--|--|
    | **ソース リポジトリの URL** | 「`https://github.com/MicrosoftLearning/azure-load-test-sample-app`」と入力します |
    | **所有者** | GitHub エイリアスを選択する |
    | **リポジトリ名** | リポジトリに名前を付けます |
    | **プライバシー** | **オーナー**を選択するとプライバシー オプションが表示されます。 **[パブリック]** を選択します。 |

1. **インポートの開始** を選択し、インポート プロセスが完了するまで待ちます。
1. 新しいリポジトリ ページで **[設定]** を選択し、左側のナビゲーション ウィンドウで **[アクション] - [全般]** を選択します。
1. ページの **[アクションのアクセス許可]** セクションで、**[すべてのアクションと再利用可能なワークフローを許可する]** オプションを選択し、**[保存]** を選択します。

## 演習 2: Azure でリソースを作成する

この演習では、アプリをデプロイしてテストを実行するために必要なリソースを Azure に作成します。 

### タスク 1: Azure CLI を使用してリソースを作成する

このタスクでは、次の Azure リソースを作成します。

* リソース グループ
* App Service プラン
* App Service インスタンス
* Load Testing インスタンス

1. ブラウザーで Azure portal [https://portal.azure.com](https://portal.azure.com) に移動します。
1. **Cloud Shell** を開き、**Bash** モードを選択します。 **注:** Cloud Shell を初めて起動する際には、永続ストレージを構成する必要がある場合があります。

1. 次のコマンドを一度に 1 つずつ実行して、残りの手順のコマンドで使用される変数を作成します。 `<mylocation>` を希望の場所に置き換えます。

    ```
    myLocation=<mylocation>
    myAppName=az2006app$RANDOM
    ```
1. 次のコマンドを実行して、他のリソースを含むリソース グループを作成します。

    ```
    az group create -n az2006-rg -l $myLocation
    ```

1. 次のコマンドを実行して、**Azure App Service** のリソース プロバイダーを登録します。

    ```bash
    az provider register --namespace Microsoft.Web
    ```

1. 次のコマンドを実行して、App Service プランを作成します。 **注:** App Service プランで使用される B1 プランでは、コストが発生する可能性があります。 

    ```
    az appservice plan create -g az2006-rg -n az2006webapp-plan --sku B1
    ```

1. 次のコマンドを実行して、アプリの App Service インスタンスを作成します。

    ```
    az webapp create -g az2006-rg -p az2006webapp-plan -n $myAppName --runtime "dotnet:8"
    ```

1. 次のコマンドを実行して、ロード テスト リソースを作成します。 **ロード**拡張機能のインストールを求めるメッセージが表示された場合は、[はい] を選択します。

    ```
    az load create -n az2006loadtest -g az2006-rg --location $myLocation
    ```

1. 次のコマンドを実行して、サブスクリプション ID を取得します。 **注:** サブスクリプション ID の値はこのラボの後半で使用するため、コマンドからの出力を必ずコピーして保存しておいてください。

    ```
    subId=$(az account list --query "[?isDefault].id" --output tsv)
    
    echo $subId
    ```

### タスク 2. サービス プリンシパルを作成し、認証を構成する

このタスクでは、アプリのサービス プリンシパルを作成し、OpenID Connect フェデレーション認証用に構成します。

1. Azure portal で、**Microsoft Entra ID** を検索してサービスに移動します。

1. 左側のナビゲーション ウィンドウで、**[管理]** グループにある **[アプリの登録]** を選択します。 

1. メイン パネルで **[+ 新規登録]** を選択して、名前として「`GH-Action-webapp`」と入力し、**[登録]** を選択します。

    >**重要:** このラボの後半で使用できるように、**アプリケーション (クライアント) ID** と**ディレクトリ (テナント) ID** の両方の値をコピーして保存します。


1. 左側のナビゲーション ウィンドウで、**[管理]** グループの **[証明書とシークレット]** を選択し、メインウィンドウで **[フェデレーション資格情報]** を選択します。 

1. **[資格情報の追加]** を選択し、選択ドロップダウンで **[Azure リソースをデプロイする GitHub Actions]** を選択します。

1. **[GitHub アカウントに接続]** セクションに次の情報を入力します。 **注:**  これらのフィールドは大文字と小文字が区別されます。 

    | フィールド | アクション |
    |--|--|
    | 組織 | ユーザーまたは組織名を入力します。 |
    | リポジトリ | ラボの前半で作成したリポジトリの名前を入力します。 |
    | エンティティ型 | **[ブランチ]** を選択します。 |
    | GitHub ブランチ名 | 「 **main**」と入力します。 |

1. **[資格情報の詳細]** セクションで、資格情報に名前を付けて **[追加]** を選択します。

### タスク 3: サービス プリンシパルにロールを割り当てる

このタスクでは、リソースにアクセスするために必要なロールをサービス プリンシパルに割り当てます。

1. GitHub ワークフローが実行するリソース テストを送信できるように、次のコマンドを実行して "ロード テスト共同作成者" ロールを割り当てます。 

    ```
    spAppId=$(az ad sp list --display-name GH-Action-webapp --query "[].{spID:appId}" --output tsv)

    loadTestId=$(az resource show -g az2006-rg -n az2006loadtest --resource-type "Microsoft.LoadTestService/loadtests" --query "id" -o tsv)

    az role assignment create --assignee $spAppId --role "Load Test Contributor"  --scope $loadTestId
    ```

1. GitHub ワークフローが App Service にアプリをデプロイできるように、次のコマンドを実行して "共同作成者" ロールを割り当てます。 

    ```
    rgId=$(az group show -n az2006-rg --query "id" -o tsv)
    
    az role assignment create --assignee $spAppId --role contributor --scope $rgId
    ```

## 演習 3: GitHub Actions を使用して Web アプリをデプロイしてテストする

この演習では、含まれているワークフローを実行するようにリポジトリを構成します。

* ワークフローは、リポジトリの *.github/workflows* フォルダーにあります。
* ワークフロー、*deploy.yml*、*loadtest.yml* のすべてが、手動で実行するように構成されます。

この演習では、ブラウザーでリポジトリ ファイルを編集します。 編集するファイルを選択した後、次のいずれかを実行できます。
* **[インプレース編集]** を選択して、編集が完了したら変更をコミットします。 
* **github.dev** でファイルを開き、ブラウザーの Visual Studio Code で編集します。 このオプションを選択した場合は、上部のメニューで **[リポジトリに戻る]** を選択すると、既定のリポジトリ エクスペリエンスに戻ることができます。

    ![編集オプションのスクリーンショット。](./media/github-edit-options.png)

### タスク 1: シークレットを構成する

このタスクでは、リポジトリにシークレットを追加して、ワークフローがユーザーに代わって Azure にログインし、アクションを実行できるようにします。

1. Web ブラウザーで [GitHub](https://github.com) に移動し、このラボ用に作成したリポジトリを選択します。 
1. リポジトリの上部にある **[設定]** を選択します。
1. 左側のナビゲーション ウィンドウで、**[シークレットと変数]** を選択し、**[アクション]** を選択します。
1. **[リポジトリ シークレット]** セクションで、次の 3 つのシークレットを追加します。 **[新しいリポジトリ シークレット]** を選択して、シークレットを追加します。

    | Name | Secret |
    |--|--|
    | `AZURE_CLIENT_ID` | ラボの前半で保存した**アプリケーション (クライアント) ID** を入力します。 |
    | `AZURE_TENANT_ID` | ラボの前半で保存した **ディレクトリ (テナント) ID** を入力します。 |
    | `AZURE_SUBSCRIPTION_ID` | ラボの前半で保存したサブスクリプション ID の値を入力します。 |

### タスク 2: Web アプリをデプロイする

1. *.github/workflows* フォルダーにある *deploy.yml* ファイルを選択します。

1. ファイルを編集し、**env:** セクションで、`AZURE_WEB_APP` 変数の値を変更します。 `<your web app name>` を、ラボの前半で作成した Web アプリの名前に置き換えます。 変更を [コミット] します。

1. 少し時間をかけてワークフローの内容を確認します。

1. リポジトリの上部のナビゲーションで、**[アクション]** を選択します。 

1. 左側のナビゲーション ウィンドウで、**[ビルドして発行する]** を選択します。

1. **[ワークフローの実行]** ドロップダウンを選択し、既定の **[Branch: main]** 設定のままにして **[ワークフローの実行]** を選択します。 ワークフローの開始には少し時間がかかる場合があります。

ワークフローの正常な完了に問題がある場合は、**[ビルドして発行する]** ワークフローを選択し、次の画面で **[ビルド]** を選択します。 ワークフローに関する詳細情報が提供され、正常に完了できなくなる問題を診断するのに役立ちます。

### タスク 3: ロード テストを実行する

1. *.github/workflows* フォルダーにある *build.yml* ファイルを選択します。

1. ファイルを編集し、**env:** セクションで、`AZURE_WEB_APP` 変数の値を変更します。 `<your web app name>**` を、ラボの前半で作成した Web アプリの名前に置き換えます。 変更を [コミット] します。

1. 少し時間をかけてワークフローの内容を確認します。

1. リポジトリの上部のナビゲーションで、**[アクション]** を選択します。 

1. 左側のナビゲーション ウィンドウで、**[ロード テスト]** を選択します。

1. **[ワークフローの実行]** ドロップダウンを選択し、既定の **[Branch: main]** 設定のままにして **[ワークフローの実行]** を選択します。 ワークフローの開始には少し時間がかかる場合があります。

    >**注:** ワークフローが完了するまでに、5 分から 10 分程度かかる場合があります。 テストは 2 分間実行され、ロード テストが Azure でキューに登録されて開始されるまでに数分かかる場合があります。 

ワークフローの正常な完了に問題がある場合は、**[ロード テスト]** ワークフローを選択し、次の画面で **[ビルド]** を選択します。 ワークフローに関する詳細情報が提供され、正常に完了できなくなる問題を診断するのに役立ちます。

#### 省略可能

リポジトリのルートにある *config.yaml* ファイルは、ロード テストの不合格条件を指定します。 ロード テストを強制的に失敗させる場合は、次の手順を実行します。

1. リポジトリのルートにある *config.yaml* ファイルを編集します。
1. `- p90(response_time_ms) > 4000` フィールドの値を低い値に変更します。 `- p90(response_time_ms) > 50` に変更すると、テストが失敗する可能性が最も高くなります。 これは、アプリが 90% の確率で 50 ミリ秒以内に応答することを表します。 

### タスク 4: ロード テストの結果の表示

CI/CD パイプラインからロード テストを実行すると、CI/CD 出力ログでサマリー結果を直接表示できます。 テスト結果はパイプライン成果物として保存されたため、CSV ファイルをダウンロードしてさらにレポートすることもできます。

![ワークフローのログ情報を示すスクリーンショット。](./media/github-actions-workflow-completed.png)

## 演習 4: リソースをクリーンアップする

この演習では、ラボの前半で作成したリソースを削除します。

1. Azure portal [https://portal.azure.com](https://portal.azure.com) に移動して、Cloud Shell を開始します。 **Bash** シェル セッションを選択します。

1. 次のコマンドを実行して、`az2006-rg` リソース グループを削除します。 また、App Service プランと App Service インスタンスも削除されます。

    ```
    az group delete -n az2006-rg --no-wait --yes
    ```

    >**注:** コマンドは非同期に実行されるので (`--no-wait` パラメーターで設定される)、同じ Bash セッション内ですぐに別の Azure CLI コマンドを実行できますが、リソース グループが実際に削除されるまでに数分かかります。

## 確認

このラボでは、Azure Web アプリをデプロイする GitHub アクション ワークフローを実装しました。
