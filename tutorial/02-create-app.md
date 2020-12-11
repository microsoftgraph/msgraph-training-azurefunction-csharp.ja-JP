---
ms.openlocfilehash: 17c93f353c84ea2db28cd2e0203d30c5f320e36e
ms.sourcegitcommit: 141fe5c30dea84029ef61cf82558c35f2a744b65
ms.translationtype: MT
ms.contentlocale: ja-JP
ms.lasthandoff: 12/11/2020
ms.locfileid: "49655234"
---
<!-- markdownlint-disable MD002 MD041 -->

<span data-ttu-id="80c72-101">このチュートリアルでは、Microsoft Graph を呼び出す HTTP トリガー関数を実装する単純な Azure 関数を作成します。</span><span class="sxs-lookup"><span data-stu-id="80c72-101">In this tutorial, you will create a simple Azure Function that implements HTTP trigger functions that call Microsoft Graph.</span></span> <span data-ttu-id="80c72-102">これらの関数は、次のシナリオに対応します。</span><span class="sxs-lookup"><span data-stu-id="80c72-102">These functions will cover the following scenarios:</span></span>

- <span data-ttu-id="80c72-103">代理フロー認証を使用してユーザーの受信トレイにアクセスする API [を実装](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-on-behalf-of-flow) します。</span><span class="sxs-lookup"><span data-stu-id="80c72-103">Implements an API to access a user's inbox using [on-behalf-of flow](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-on-behalf-of-flow) authentication.</span></span>
- <span data-ttu-id="80c72-104">クライアント資格情報の付与フロー認証を使用して、ユーザーの受信トレイの通知をサブスクライブおよび購読解除する API [を実装](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow) します。</span><span class="sxs-lookup"><span data-stu-id="80c72-104">Implements an API to subscribe and unsubscribe for notifications on a user's inbox, using using [client credentials grant flow](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow) authentication.</span></span>
- <span data-ttu-id="80c72-105">Webhook を実装して、Microsoft Graph から変更通知を受信し、クライアント資格情報の付与フローを使用してデータにアクセスします。 [](https://docs.microsoft.com/graph/webhooks)</span><span class="sxs-lookup"><span data-stu-id="80c72-105">Implements a webhook to receive [change notifications](https://docs.microsoft.com/graph/webhooks) from Microsoft Graph and access data using client credentials grant flow.</span></span>

<span data-ttu-id="80c72-106">また、Azure 関数に実装されている API を呼び出す単純な JavaScript シングル ページ アプリケーション (SPA) を作成します。</span><span class="sxs-lookup"><span data-stu-id="80c72-106">You will also create a simple JavaScript single-page application (SPA) to call the APIs implemented in the Azure Function.</span></span>

## <a name="create-azure-functions-project"></a><span data-ttu-id="80c72-107">Azure Functions プロジェクトを作成する</span><span class="sxs-lookup"><span data-stu-id="80c72-107">Create Azure Functions project</span></span>

1. <span data-ttu-id="80c72-108">プロジェクトを作成するディレクトリでコマンド ライン インターフェイス (CLI) を開きます。</span><span class="sxs-lookup"><span data-stu-id="80c72-108">Open your command-line interface (CLI) in a directory where you want to create the project.</span></span> <span data-ttu-id="80c72-109">次のコマンドを実行します。</span><span class="sxs-lookup"><span data-stu-id="80c72-109">Run the following command.</span></span>

    ```Shell
    func init GraphTutorial --dotnet
    ```

1. <span data-ttu-id="80c72-110">CLI の現在のディレクトリを **GraphTu clil** ディレクトリに変更し、次のコマンドを実行してプロジェクトに 3 つの関数を作成します。</span><span class="sxs-lookup"><span data-stu-id="80c72-110">Change the current directory in your CLI to the **GraphTutorial** directory and run the following commands to create three functions in the project.</span></span>

    ```Shell
    func new --name GetMyNewestMessage --template "HTTP trigger" --language C#
    func new --name SetSubscription --template "HTTP trigger" --language C#
    func new --name Notify --template "HTTP trigger" --language C#
    ```

1. <span data-ttu-id="80c72-111">テスト **local.settings.jsを開** き、次のコードをファイルに追加して CORS を許可し、テスト アプリケーションの `http://localhost:8080` URL を指定します。</span><span class="sxs-lookup"><span data-stu-id="80c72-111">Open **local.settings.json** and add the following to the file to allow CORS from `http://localhost:8080`, the URL for the test application.</span></span>

    ```json
    "Host": {
      "CORS": "http://localhost:8080"
    }
    ```

1. <span data-ttu-id="80c72-112">次のコマンドを実行して、プロジェクトをローカルで実行します。</span><span class="sxs-lookup"><span data-stu-id="80c72-112">Run the following command to run the project locally.</span></span>

    ```Shell
    func start
    ```

1. <span data-ttu-id="80c72-113">すべてが動作している場合は、次の出力が表示されます。</span><span class="sxs-lookup"><span data-stu-id="80c72-113">If everything is working, you will see the following output:</span></span>

    ```Shell
    Http Functions:

        GetMyNewestMessage: [GET,POST] http://localhost:7071/api/GetMyNewestMessage

        Notify: [GET,POST] http://localhost:7071/api/Notify

        SetSubscription: [GET,POST] http://localhost:7071/api/SetSubscription
    ```

1. <span data-ttu-id="80c72-114">ブラウザーを開き、出力に表示される関数 URL を参照して、関数が正しく動作しているのを確認します。</span><span class="sxs-lookup"><span data-stu-id="80c72-114">Verify that the functions are working correctly by opening your browser and browsing to the function URLs shown in the output.</span></span> <span data-ttu-id="80c72-115">ブラウザーに次のメッセージが表示されます `This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response.` 。</span><span class="sxs-lookup"><span data-stu-id="80c72-115">You should see the following message in your browser: `This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response.`.</span></span>

## <a name="create-single-page-application"></a><span data-ttu-id="80c72-116">単一ページ アプリケーションを作成する</span><span class="sxs-lookup"><span data-stu-id="80c72-116">Create single-page application</span></span>

1. <span data-ttu-id="80c72-117">プロジェクトを作成するディレクトリで CLI を開きます。</span><span class="sxs-lookup"><span data-stu-id="80c72-117">Open your CLI in a directory where you want to create the project.</span></span> <span data-ttu-id="80c72-118">HTML ファイルと **JavaScript ファイルを保持する TestClient** という名前のディレクトリを作成します。</span><span class="sxs-lookup"><span data-stu-id="80c72-118">Create a directory named **TestClient** to hold your HTML and JavaScript files.</span></span>

1. <span data-ttu-id="80c72-119">**TestClient** ディレクトリに **index.htmという** 名前の新しいファイルを作成し、次のコードを追加します。</span><span class="sxs-lookup"><span data-stu-id="80c72-119">Create a new file named **index.html** in the **TestClient** directory and add the following code.</span></span>

    :::code language="html" source="../demo/TestClient/index.html" id="indexSnippet":::

    <span data-ttu-id="80c72-120">これにより、ナビゲーション バーを含むアプリの基本的なレイアウトが定義されます。</span><span class="sxs-lookup"><span data-stu-id="80c72-120">This defines the basic layout of the app, including a navigation bar.</span></span> <span data-ttu-id="80c72-121">また、次の追加も行います。</span><span class="sxs-lookup"><span data-stu-id="80c72-121">It also adds the following:</span></span>

    - <span data-ttu-id="80c72-122">[ブートストラップ](https://getbootstrap.com/) とそのサポート JavaScript</span><span class="sxs-lookup"><span data-stu-id="80c72-122">[Bootstrap](https://getbootstrap.com/) and its supporting JavaScript</span></span>
    - [<span data-ttu-id="80c72-123">FontAwesome</span><span class="sxs-lookup"><span data-stu-id="80c72-123">FontAwesome</span></span>](https://fontawesome.com/)
    - [<span data-ttu-id="80c72-124">Microsoft Authentication Library for JavaScript (MSAL.js) 2.0</span><span class="sxs-lookup"><span data-stu-id="80c72-124">Microsoft Authentication Library for JavaScript (MSAL.js) 2.0</span></span>](https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/lib/msal-browser)

    > [!TIP]
    > <span data-ttu-id="80c72-125">ページには、ファビコン ( ) が含まれます `<link rel="shortcut icon" href="g-raph.png">` 。</span><span class="sxs-lookup"><span data-stu-id="80c72-125">The page includes a favicon, (`<link rel="shortcut icon" href="g-raph.png">`).</span></span> <span data-ttu-id="80c72-126">この行を削除するか、GitHub から **g-raph.pngファイル**[をダウンロードできます](https://github.com/microsoftgraph/g-raph)。</span><span class="sxs-lookup"><span data-stu-id="80c72-126">You can remove this line, or you can download the **g-raph.png** file from [GitHub](https://github.com/microsoftgraph/g-raph).</span></span>

1. <span data-ttu-id="80c72-127">**TestClient** ディレクトリに **style.css という** 名前の新しいファイルを作成し、次のコードを追加します。</span><span class="sxs-lookup"><span data-stu-id="80c72-127">Create a new file named **style.css** in the **TestClient** directory and add the following code.</span></span>

    :::code language="css" source="../demo/TestClient/style.css":::

1. <span data-ttu-id="80c72-128">**TestClient** ディレクトリに **ui.jsという** 名前の新しいファイルを作成し、次のコードを追加します。</span><span class="sxs-lookup"><span data-stu-id="80c72-128">Create a new file named **ui.js** in the **TestClient** directory and add the following code.</span></span>

    :::code language="javascript" source="../demo/TestClient/ui.js" id="uiJsSnippet":::

    <span data-ttu-id="80c72-129">このコードは、JavaScript を使用して、選択したビューに基づいて現在のページをレンダリングします。</span><span class="sxs-lookup"><span data-stu-id="80c72-129">This code uses JavaScript to render the current page based on the selected view.</span></span>

### <a name="test-the-single-page-application"></a><span data-ttu-id="80c72-130">単一ページ アプリケーションをテストする</span><span class="sxs-lookup"><span data-stu-id="80c72-130">Test the single-page application</span></span>

> [!NOTE]
> <span data-ttu-id="80c72-131">このセクションでは、開発用コンピューターで簡単なテスト HTTP サーバーを実行するために [dotnet-serve](https://github.com/natemcmaster/dotnet-serve) を使用する手順について説明します。</span><span class="sxs-lookup"><span data-stu-id="80c72-131">This section includes instructions for using [dotnet-serve](https://github.com/natemcmaster/dotnet-serve) to run a simple testing HTTP server on your development machine.</span></span> <span data-ttu-id="80c72-132">この特定のツールを使用する必要はありません。</span><span class="sxs-lookup"><span data-stu-id="80c72-132">Using this specific tool is not required.</span></span> <span data-ttu-id="80c72-133">TestClient ディレクトリを提供する任意のテスト サーバー **を使用** できます。</span><span class="sxs-lookup"><span data-stu-id="80c72-133">You can use any testing server you prefer to serve the **TestClient** directory.</span></span>

1. <span data-ttu-id="80c72-134">CLI で次のコマンドを実行して **、dotnet-serve をインストールします**。</span><span class="sxs-lookup"><span data-stu-id="80c72-134">Run the following command in your CLI to install **dotnet-serve**.</span></span>

    ```Shell
    dotnet tool install --global dotnet-serve
    ```

1. <span data-ttu-id="80c72-135">CLI の現在のディレクトリを **TestClient** ディレクトリに変更し、次のコマンドを実行して HTTP サーバーを起動します。</span><span class="sxs-lookup"><span data-stu-id="80c72-135">Change the current directory in your CLI to the **TestClient** directory and run the following command to start an HTTP server.</span></span>

    ```Shell
    dotnet serve -h "Cache-Control: no-cache, no-store, must-revalidate" -p 8080
    ```

1. <span data-ttu-id="80c72-136">ブラウザーを開き、`http://localhost:8080` に移動します。</span><span class="sxs-lookup"><span data-stu-id="80c72-136">Open your browser and navigate to `http://localhost:8080`.</span></span> <span data-ttu-id="80c72-137">ページはレンダリングされますが、現在は機能するボタンはいずれも表示されません。</span><span class="sxs-lookup"><span data-stu-id="80c72-137">The page should render, but none of the buttons currently work.</span></span>

## <a name="add-nuget-packages"></a><span data-ttu-id="80c72-138">NuGet パッケージを追加する</span><span class="sxs-lookup"><span data-stu-id="80c72-138">Add NuGet packages</span></span>

<span data-ttu-id="80c72-139">次に進む前に、後で使用する追加の NuGet パッケージをインストールします。</span><span class="sxs-lookup"><span data-stu-id="80c72-139">Before moving on, install some additional NuGet packages that you will use later.</span></span>

- <span data-ttu-id="80c72-140">Azure Functions プロジェクトで依存関係挿入を有効にする[Microsoft.Azure.Functions.Extensions。](https://www.nuget.org/packages/Microsoft.Azure.Functions.Extensions)</span><span class="sxs-lookup"><span data-stu-id="80c72-140">[Microsoft.Azure.Functions.Extensions](https://www.nuget.org/packages/Microsoft.Azure.Functions.Extensions) to enable dependency injection in the Azure Functions project.</span></span>
- <span data-ttu-id="80c72-141">[Microsoft.Extensions.Configuriation。.NET 開発シークレット](https://www.nuget.org/packages/Microsoft.Extensions.Configuration.UserSecrets) ストアからアプリケーション構成を読 [み取る](https://docs.microsoft.com/aspnet/core/security/app-secrets)UserSecrets。</span><span class="sxs-lookup"><span data-stu-id="80c72-141">[Microsoft.Extensions.Configuration.UserSecrets](https://www.nuget.org/packages/Microsoft.Extensions.Configuration.UserSecrets) to read application configuration from the [.NET development secret store](https://docs.microsoft.com/aspnet/core/security/app-secrets).</span></span>
- <span data-ttu-id="80c72-142">[Microsoft.Graph](https://www.nuget.org/packages/Microsoft.Graph/): Microsoft Graph を呼び出すためのものです。</span><span class="sxs-lookup"><span data-stu-id="80c72-142">[Microsoft.Graph](https://www.nuget.org/packages/Microsoft.Graph/) for making calls to Microsoft Graph.</span></span>
- <span data-ttu-id="80c72-143">[トークンを認証および管理するための Microsoft.Identity.Client。](https://www.nuget.org/packages/Microsoft.Identity.Client/)</span><span class="sxs-lookup"><span data-stu-id="80c72-143">[Microsoft.Identity.Client](https://www.nuget.org/packages/Microsoft.Identity.Client/) for authenticating and managing tokens.</span></span>
- <span data-ttu-id="80c72-144">[トークン検証用の OpenID 構成を取得するための Microsoft.IdentityModel.Protocols.OpenIdConnect。](https://www.nuget.org/packages/Microsoft.IdentityModel.Protocols.OpenIdConnect)</span><span class="sxs-lookup"><span data-stu-id="80c72-144">[Microsoft.IdentityModel.Protocols.OpenIdConnect](https://www.nuget.org/packages/Microsoft.IdentityModel.Protocols.OpenIdConnect) for retrieving OpenID configuration for token validation.</span></span>
- <span data-ttu-id="80c72-145">Web API に送信されたトークンを検証するための[System.IdentityModel.Tokens.Jwt。](https://www.nuget.org/packages/System.IdentityModel.Tokens.Jwt)</span><span class="sxs-lookup"><span data-stu-id="80c72-145">[System.IdentityModel.Tokens.Jwt](https://www.nuget.org/packages/System.IdentityModel.Tokens.Jwt) for validating tokens sent to the web API.</span></span>

1. <span data-ttu-id="80c72-146">CLI の現在のディレクトリを **GraphTu clil** ディレクトリに変更し、次のコマンドを実行します。</span><span class="sxs-lookup"><span data-stu-id="80c72-146">Change the current directory in your CLI to the **GraphTutorial** directory and run the following commands.</span></span>

    ```Shell
    dotnet add package Microsoft.Azure.Functions.Extensions --version 1.0.0
    dotnet add package Microsoft.Extensions.Configuration.UserSecrets --version 3.1.5
    dotnet add package Microsoft.Graph --version 3.8.0
    dotnet add package Microsoft.Identity.Client --version 4.15.0
    dotnet add package Microsoft.IdentityModel.Protocols.OpenIdConnect --version 6.7.1
    dotnet add package System.IdentityModel.Tokens.Jwt --version 6.7.1
    ```
