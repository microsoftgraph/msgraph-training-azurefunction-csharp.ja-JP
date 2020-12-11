---
ms.openlocfilehash: 17c93f353c84ea2db28cd2e0203d30c5f320e36e
ms.sourcegitcommit: 141fe5c30dea84029ef61cf82558c35f2a744b65
ms.translationtype: MT
ms.contentlocale: ja-JP
ms.lasthandoff: 12/11/2020
ms.locfileid: "49655234"
---
<!-- markdownlint-disable MD002 MD041 -->

このチュートリアルでは、Microsoft Graph を呼び出す HTTP トリガー関数を実装する単純な Azure 関数を作成します。 これらの関数は、次のシナリオに対応します。

- 代理フロー認証を使用してユーザーの受信トレイにアクセスする API [を実装](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-on-behalf-of-flow) します。
- クライアント資格情報の付与フロー認証を使用して、ユーザーの受信トレイの通知をサブスクライブおよび購読解除する API [を実装](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow) します。
- Webhook を実装して、Microsoft Graph から変更通知を受信し、クライアント資格情報の付与フローを使用してデータにアクセスします。 [](https://docs.microsoft.com/graph/webhooks)

また、Azure 関数に実装されている API を呼び出す単純な JavaScript シングル ページ アプリケーション (SPA) を作成します。

## <a name="create-azure-functions-project"></a>Azure Functions プロジェクトを作成する

1. プロジェクトを作成するディレクトリでコマンド ライン インターフェイス (CLI) を開きます。 次のコマンドを実行します。

    ```Shell
    func init GraphTutorial --dotnet
    ```

1. CLI の現在のディレクトリを **GraphTu clil** ディレクトリに変更し、次のコマンドを実行してプロジェクトに 3 つの関数を作成します。

    ```Shell
    func new --name GetMyNewestMessage --template "HTTP trigger" --language C#
    func new --name SetSubscription --template "HTTP trigger" --language C#
    func new --name Notify --template "HTTP trigger" --language C#
    ```

1. テスト **local.settings.jsを開** き、次のコードをファイルに追加して CORS を許可し、テスト アプリケーションの `http://localhost:8080` URL を指定します。

    ```json
    "Host": {
      "CORS": "http://localhost:8080"
    }
    ```

1. 次のコマンドを実行して、プロジェクトをローカルで実行します。

    ```Shell
    func start
    ```

1. すべてが動作している場合は、次の出力が表示されます。

    ```Shell
    Http Functions:

        GetMyNewestMessage: [GET,POST] http://localhost:7071/api/GetMyNewestMessage

        Notify: [GET,POST] http://localhost:7071/api/Notify

        SetSubscription: [GET,POST] http://localhost:7071/api/SetSubscription
    ```

1. ブラウザーを開き、出力に表示される関数 URL を参照して、関数が正しく動作しているのを確認します。 ブラウザーに次のメッセージが表示されます `This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response.` 。

## <a name="create-single-page-application"></a>単一ページ アプリケーションを作成する

1. プロジェクトを作成するディレクトリで CLI を開きます。 HTML ファイルと **JavaScript ファイルを保持する TestClient** という名前のディレクトリを作成します。

1. **TestClient** ディレクトリに **index.htmという** 名前の新しいファイルを作成し、次のコードを追加します。

    :::code language="html" source="../demo/TestClient/index.html" id="indexSnippet":::

    これにより、ナビゲーション バーを含むアプリの基本的なレイアウトが定義されます。 また、次の追加も行います。

    - [ブートストラップ](https://getbootstrap.com/) とそのサポート JavaScript
    - [FontAwesome](https://fontawesome.com/)
    - [Microsoft Authentication Library for JavaScript (MSAL.js) 2.0](https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/lib/msal-browser)

    > [!TIP]
    > ページには、ファビコン ( ) が含まれます `<link rel="shortcut icon" href="g-raph.png">` 。 この行を削除するか、GitHub から **g-raph.pngファイル**[をダウンロードできます](https://github.com/microsoftgraph/g-raph)。

1. **TestClient** ディレクトリに **style.css という** 名前の新しいファイルを作成し、次のコードを追加します。

    :::code language="css" source="../demo/TestClient/style.css":::

1. **TestClient** ディレクトリに **ui.jsという** 名前の新しいファイルを作成し、次のコードを追加します。

    :::code language="javascript" source="../demo/TestClient/ui.js" id="uiJsSnippet":::

    このコードは、JavaScript を使用して、選択したビューに基づいて現在のページをレンダリングします。

### <a name="test-the-single-page-application"></a>単一ページ アプリケーションをテストする

> [!NOTE]
> このセクションでは、開発用コンピューターで簡単なテスト HTTP サーバーを実行するために [dotnet-serve](https://github.com/natemcmaster/dotnet-serve) を使用する手順について説明します。 この特定のツールを使用する必要はありません。 TestClient ディレクトリを提供する任意のテスト サーバー **を使用** できます。

1. CLI で次のコマンドを実行して **、dotnet-serve をインストールします**。

    ```Shell
    dotnet tool install --global dotnet-serve
    ```

1. CLI の現在のディレクトリを **TestClient** ディレクトリに変更し、次のコマンドを実行して HTTP サーバーを起動します。

    ```Shell
    dotnet serve -h "Cache-Control: no-cache, no-store, must-revalidate" -p 8080
    ```

1. ブラウザーを開き、`http://localhost:8080` に移動します。 ページはレンダリングされますが、現在は機能するボタンはいずれも表示されません。

## <a name="add-nuget-packages"></a>NuGet パッケージを追加する

次に進む前に、後で使用する追加の NuGet パッケージをインストールします。

- Azure Functions プロジェクトで依存関係挿入を有効にする[Microsoft.Azure.Functions.Extensions。](https://www.nuget.org/packages/Microsoft.Azure.Functions.Extensions)
- [Microsoft.Extensions.Configuriation。.NET 開発シークレット](https://www.nuget.org/packages/Microsoft.Extensions.Configuration.UserSecrets) ストアからアプリケーション構成を読 [み取る](https://docs.microsoft.com/aspnet/core/security/app-secrets)UserSecrets。
- [Microsoft.Graph](https://www.nuget.org/packages/Microsoft.Graph/): Microsoft Graph を呼び出すためのものです。
- [トークンを認証および管理するための Microsoft.Identity.Client。](https://www.nuget.org/packages/Microsoft.Identity.Client/)
- [トークン検証用の OpenID 構成を取得するための Microsoft.IdentityModel.Protocols.OpenIdConnect。](https://www.nuget.org/packages/Microsoft.IdentityModel.Protocols.OpenIdConnect)
- Web API に送信されたトークンを検証するための[System.IdentityModel.Tokens.Jwt。](https://www.nuget.org/packages/System.IdentityModel.Tokens.Jwt)

1. CLI の現在のディレクトリを **GraphTu clil** ディレクトリに変更し、次のコマンドを実行します。

    ```Shell
    dotnet add package Microsoft.Azure.Functions.Extensions --version 1.0.0
    dotnet add package Microsoft.Extensions.Configuration.UserSecrets --version 3.1.5
    dotnet add package Microsoft.Graph --version 3.8.0
    dotnet add package Microsoft.Identity.Client --version 4.15.0
    dotnet add package Microsoft.IdentityModel.Protocols.OpenIdConnect --version 6.7.1
    dotnet add package System.IdentityModel.Tokens.Jwt --version 6.7.1
    ```
