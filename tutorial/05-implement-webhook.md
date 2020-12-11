---
ms.openlocfilehash: eb227079656e2a57550511c3abfacb49935fe46a
ms.sourcegitcommit: 141fe5c30dea84029ef61cf82558c35f2a744b65
ms.translationtype: MT
ms.contentlocale: ja-JP
ms.lasthandoff: 12/11/2020
ms.locfileid: "49655241"
---
<!-- markdownlint-disable MD002 MD041 -->

この演習では、Azure Functions の実装を完了し、テスト アプリケーションを更新して、ユーザーの受信トレイの変更をサブスクライブおよびサブスクライブ解除 `SetSubscription` `Notify` します。

- この関数は API として機能し、テスト アプリがユーザーの受信トレイの変更に対するサブスクリプションを作成 `SetSubscription` または削除できます。 [](https://docs.microsoft.com/graph/webhooks)
- この `Notify` 関数は、サブスクリプションによって生成された変更通知を受信する Webhook として機能します。

どちらの関数も、クライアント資格情報 [の付与](https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow) フローを使用して、Microsoft Graph を呼び出すアプリ専用トークンを取得します。 管理者が必要なアクセス許可のスコープに管理者の同意を与えたため、トークンを取得するためにユーザーの操作は必要ありません。

## <a name="add-client-credentials-authentication-to-the-azure-functions-project"></a>Azure Functions プロジェクトにクライアント資格情報認証を追加する

このセクションでは、Azure Functions プロジェクトにクライアント資格情報フローを実装して、Microsoft Graph と互換性のあるアクセス トークンを取得します。

1. **GraphTu clil.csproj** を含むディレクトリで CLI を開きます。

1. 次のコマンドを使用して、Webhook アプリケーション ID とシークレットをシークレット ストアに追加します。 `YOUR_WEBHOOK_APP_ID_HERE`Graph Azure 関数 Webhook のアプリケーション ID に **置き換える**。 Graph Azure 関数 Webhook の Azure ポータルで作成したアプリケーション `YOUR_WEBHOOK_APP_SECRET_HERE` **シークレットに置き換える**。

    ```Shell
    dotnet user-secrets set webHookId "YOUR_WEBHOOK_APP_ID_HERE"
    dotnet user-secrets set webHookSecret "YOUR_WEBHOOK_APP_SECRET_HERE"
    ```

### <a name="create-a-client-credentials-authentication-provider"></a>クライアント資格情報認証プロバイダーを作成する

1. **./GraphTu ClientCredentialsAuthProvider.cs ディレクトリ** に新しいファイルを作成し **、次の** コードを追加します。

    :::code language="csharp" source="../demo/GraphTutorial/Authentication/ClientCredentialsAuthProvider.cs" id="AuthProviderSnippet":::

コードの動作を少し **ClientCredentialsAuthProvider.cs** してください。

- コンストラクターでは、パッケージから **ConfidentialClientApplication** を初期化 `Microsoft.Identity.Client` します。 この関数を `WithAuthority(AadAuthorityAudience.AzureAdMyOrg, true)` 使用 `.WithTenantId(tenantId)` して、ログイン対象ユーザーを指定された Microsoft 365 組織にのみ制限します。
- 関数では `GetAccessToken` 、アプリケーションの `AcquireTokenForClient` トークンを取得するために呼び出されます。 クライアント資格情報トークン フローは常に非対話型です。
- インターフェイスを実装し、送信要求を認証するコンストラクターでこのクラス `Microsoft.Graph.IAuthenticationProvider` `GraphServiceClient` を渡すことができます。

## <a name="update-graphclientservice"></a>GraphClientService の更新

1. クラス **GraphClientService.cs** を開き、次のプロパティをクラスに追加します。

    :::code language="csharp" source="../demo/GraphTutorial/Services/GraphClientService.cs" id="AppGraphClientMembers":::

1. 既存の `GetAppGraphClient` 関数を、以下の関数で置き換えます。

    :::code language="csharp" source="../demo/GraphTutorial/Services/GraphClientService.cs" id="AppGraphClientFunctions":::

## <a name="implement-notify-function"></a>Notify 関数を実装する

このセクションでは、変更通知の通知 URL として使用される関数 `Notify` を実装します。

1. Models という名前の **GraphTu graphls** ディレクトリに新しいディレクトリを **作成します**。

1. Models ディレクトリに新しいファイルを作成 **し、ResourceData.cs****コードを** 追加します。

    :::code language="csharp" source="../demo/GraphTutorial/Models/ResourceData.cs" id="ResourceDataSnippet":::

1. Models ディレクトリに新しいファイルを作成 **し、ChangeNotification.cs****コードを** 追加します。

    :::code language="csharp" source="../demo/GraphTutorial/Models/ChangeNotification.cs" id="ChangeNotificationSnippet":::

1. Models ディレクトリに新しいファイルを作成 **し、NotificationList.cs****コードを** 追加します。

    :::code language="csharp" source="../demo/GraphTutorial/Models/NotificationList.cs" id="NotificationListSnippet":::

1. **./GraphTu graphl/Notify.cs** を開き、その内容全体を次の内容に置き換えてください。

    :::code language="csharp" source="../demo/GraphTutorial/Notify.cs" id="NotifySnippet":::

コードの動作を少し **Notify.cs** してください。

- この `Run` 関数は、クエリ パラメーターの有無を `validationToken` 確認します。 このパラメーターが指定されている場合は、要求を検証要求として処理 [し、それに](https://docs.microsoft.com/graph/webhooks#notification-endpoint-validation)応じて応答します。
- 要求が検証要求ではない場合、JSON ペイロードは逆シリアル化され、 `NotificationList` .
- 一覧内の各通知で、予期されるクライアント状態の値がチェックされ、処理されます。
- 通知をトリガーしたメッセージは、Microsoft Graph で取得されます。

## <a name="implement-setsubscription-function"></a>SetSubscription 関数を実装する

このセクションでは、SetSubscription 関数を実装します。 この関数は、ユーザーの受信トレイでサブスクリプションを作成または削除するためにテスト アプリケーションによって呼び出される API として機能します。

1. Models ディレクトリに新しいファイルを作成 **し、SetSubscriptionPayload.cs****コードを** 追加します。

    :::code language="csharp" source="../demo/GraphTutorial/Models/SetSubscriptionPayload.cs" id="SetSubscriptionPayloadSnippet":::

1. **./GraphTu graphl/SetSubscription.cs** を開き、そのコンテンツ全体を次の内容に置き換えてください。

    :::code language="csharp" source="../demo/GraphTutorial/SetSubscription.cs" id="SetSubscriptionSnippet":::

コードの動作を少し **SetSubscription.cs** してください。

- この関数は、POST 要求で送信された JSON ペイロードを読み取り、要求の種類 (サブスクライブまたはサブスクライブ解除)、サブスクライブするユーザー ID、登録を解除するサブスクリプション ID を `Run` 決定します。
- 要求がサブスクライブ要求の場合は、Microsoft Graph SDK を使用して、指定したユーザーの受信トレイに新しいサブスクリプションを作成します。 サブスクリプションは、メッセージが作成または更新された場合に通知します。 新しいサブスクリプションは、応答の JSON ペイロードで返されます。
- 要求が登録解除要求の場合は、Microsoft Graph SDK を使用して指定されたサブスクリプションを削除します。

## <a name="call-setsubscription-from-the-test-app"></a>テスト アプリから SetSubscription を呼び出す

このセクションでは、テスト アプリでサブスクリプションを作成および削除する関数を実装します。

1. **./TestClient/azurefunctions.js** 開き、次の関数を追加します。

    :::code language="javascript" source="../demo/TestClient/azurefunctions.js" id="createSubscriptionSnippet":::

    このコードは、Azure 関数を呼び出してサブスクライブし、セッション内のサブスクリプションの配列に新しい `SetSubscription` サブスクリプションを追加します。

1. 次の関数を追加 **して** azurefunctions.js。

    :::code language="javascript" source="../demo/TestClient/azurefunctions.js" id="deleteSubscriptionSnippet":::

    このコードは、Azure 関数を呼び出して、セッション内のサブスクリプションの配列からサブスクリプションを解除 `SetSubscription` し、削除します。

1. ngrok が実行されていない場合は、ngrok ( ) を実行し、HTTPS 転送 `ngrok http 7071` URL をコピーします。

1. 次のコマンドを実行して、ngrok URL をユーザー シークレット ストアに追加します。

    ```Shell
    dotnet user-secrets set ngrokUrl "YOUR_NGROK_URL_HERE"
    ```

    > [!IMPORTANT]
    > ngrok を再起動する場合は、このコマンドを繰り返して ngrok URL を更新する必要があります。

1. CLI の現在のディレクトリを **./GraphTu clil** ディレクトリに変更し、次のコマンドを実行して Azure 関数をローカルで開始します。

    ```Shell
    func start
    ```

1. SPA を更新し、[サブスクリプション] ナビゲーション **項目を** 選択します。 Exchange Online メールボックスを持つ Microsoft 365 組織のユーザーのユーザー ID を入力します。 これは、ユーザー (Microsoft Graph から) またはユーザーのいずれか `id` です `userPrincipalName` 。 [サブスクライブ **] をクリックします**。

1. ページが更新され、表に新しいサブスクリプションが表示されます。

1. ユーザーにメールを送信します。 しばらくすると、関数 `Notify` が呼び出されます。 これは、ngrok Web インターフェイス ( ) または Azure Function プロジェクトの `http://localhost:4040` デバッグ出力で確認できます。

    ```Shell
    ...
    [7/8/2020 7:33:57 PM] The following message was created:
    [7/8/2020 7:33:57 PM] Subject: Hi Megan!, ID: AAMkAGUyN2I4N2RlLTEzMTAtNDBmYy1hODdlLTY2NTQwODE2MGEwZgBGAAAAAAA2J9QH-DvMRK3pBt_8rA6nBwCuPIFjbMEkToHcVnQirM5qAAAAAAEMAACuPIFjbMEkToHcVnQirM5qAACHmpAsAAA=
    [7/8/2020 7:33:57 PM] Executed 'Notify' (Succeeded, Id=9c40af0b-e082-4418-aa3a-aee624f30e7a)
    ...
    ```

1. テスト アプリで、サブスクリプション **の表の** 行で [削除] をクリックします。 ページが更新され、サブスクリプションがテーブルに表示されなくなりました。
