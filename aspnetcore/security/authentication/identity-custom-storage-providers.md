---
title: "自訂的儲存提供者，ASP.NET Core 識別 |Microsoft 文件"
author: ardalis
description: "如何設定自訂儲存 ASP.NET Core 身分識別提供者。"
keywords: "ASP.NET Core，身分識別，自訂的存放裝置提供者"
ms.author: riande
manager: wpickett
ms.date: 05/24/2017
ms.topic: article
ms.assetid: b2ace545-ecf6-4664-b31e-b65bd4a6b025
ms.technology: aspnet
ms.prod: asp.net-core
uid: security/authentication/identity-custom-storage-providers
ms.openlocfilehash: c1d974e72eab388ba7b196c4b48f21a06b59dc20
ms.sourcegitcommit: f5cf472d49c2475e4d57654efd5fc0a4ccecba4c
ms.translationtype: MT
ms.contentlocale: zh-TW
ms.lasthandoff: 09/30/2017
---
# <a name="custom-storage-providers-for-aspnet-core-identity"></a>ASP.NET Core 身分識別的自訂儲存體提供者

由[Steve Smith](https://ardalis.com/)

ASP.NET Core 身分識別是可擴充的系統可讓您建立自訂的儲存提供者，然後連接到您的應用程式。 本主題描述如何建立 ASP.NET Core 身分識別的自訂儲存體提供者。 它涵蓋建立自己的儲存體提供者的重要概念，但不是會逐步解說。

[檢視或從 GitHub 下載範例](https://github.com/aspnet/Docs/tree/master/aspnetcore/security/authentication/identity/sample)。

## <a name="introduction"></a>簡介

根據預設，ASP.NET Core 識別系統會將使用者資訊儲存在 SQL Server 資料庫中使用 Entity Framework Core。 這個方法對於許多應用程式，非常適合。 不過，您可能想要使用不同的持續性機制或資料結構描述。 例如: 

* 您使用[Azure 資料表儲存體](https://docs.microsoft.com/azure/storage/)或另一個資料存放區。
* 資料庫資料表有不同的結構。 
* 您可能想要使用不同的資料存取方法，例如[Dapper](https://github.com/StackExchange/Dapper)。 

在每個這種情況下，您可以撰寫自訂提供者的儲存機制，並插入您的應用程式中的該提供者。

ASP.NET Core 識別隨附於 Visual Studio 中的專案範本與 「 個別使用者帳戶 」 選項。

當使用.NET 核心 CLI，新增`-au Individual`:

```console
dotnet new mvc -au Individual
dotnet new webapi -au Individual
```

## <a name="the-aspnet-core-identity-architecture"></a>ASP.NET Core 識別架構

ASP.NET Core 識別類別，稱為管理員和存放區所組成。 *管理員*是高階應用程式開發人員用來執行作業，例如建立身分識別使用者的類別。 *存放區*是較低層級的類別，指定如何保存實體，例如使用者和角色。 請遵循存放區[儲存機制模式](http://deviq.com/repository-pattern/)和緊密結合的持續性機制。 管理員會分開存放區，這表示您可以取代持續性機制，而不需要變更您的應用程式程式碼 （除了組態）。

下圖顯示 web 應用程式的互動方式經理，而與資料存取層的存放區互動。

![ASP.NET Core 應用程式使用 （例如，'UserManager'、 'RoleManager'） 的管理員。 管理員可搭配存放區 (例如，' UserStore') 進行通訊與資料來源使用像 Entity Framework 核心程式庫。](identity-custom-storage-providers/_static/identity-architecture-diagram.png)

若要建立自訂的儲存提供者，建立資料來源、 資料存取層級和與此資料存取層 （綠色和灰色方塊上圖所示） 互動的存放區類別。 您不需要自訂的管理員或應用程式程式碼互動 （藍色方塊上面）。

建立的新執行個體時`UserManager`或`RoleManager`您提供使用者類別的型別，並將存放區類別的執行個體傳遞做為引數。 這個方法可讓您將 ASP.NET Core 插入您的自訂的類別。 

[重新設定應用程式使用新的存放裝置提供者](#reconfigure-app-to-use-new-storage-provider)示範如何具現化`UserManager`和`RoleManager`與自訂存放區。

## <a name="aspnet-core-identity-stores-data-types"></a>ASP.NET Core 身分識別儲存的資料類型

[ASP.NET Core 識別](https://github.com/aspnet/identity)資料類型詳述於以下各節：

### <a name="users"></a>使用者

註冊您的網站的使用者。 [IdentityUser](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnet.identity.corecompat.identityuser)類型可能會擴充，或使用您自己的自訂類型的範例。 您不需要繼承自特定的型別，以實作您自己的自訂身分識別儲存體解決方案。

### <a name="user-claims"></a>使用者宣告

一組陳述式 (或[宣告](https://docs.microsoft.com//dotnet/api/system.security.claims.claim)代表使用者的身分識別之使用者的相關。 可以啟用更高的使用者身分識別與透過角色能達到之效果的運算式。

### <a name="user-logins"></a>使用者登入

外部驗證提供者 （例如 Facebook 或 Microsoft 帳戶） 的相關資訊時登入使用者使用。 [範例](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnet.identity.corecompat.identityuserlogin)

### <a name="roles"></a>角色

您的網站的授權群組。 包含的角色識別碼和角色名稱 （例如"Admin"或"Employee"）。 [範例](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnet.identity.corecompat.identityrole)

## <a name="the-data-access-layer"></a>資料存取層

本主題假設您熟悉您要使用的持續性機制，以及如何建立實體，該機制。 本主題不提供如何建立儲存機制或資料存取類別; 詳細資料使用 ASP.NET Core 識別時，它會提供有關設計決策的一些建議。

設計自訂存放區提供者的資料存取層時，您會有很多的自由。 您只需要建立持續性機制，可用於您想要使用您的應用程式中的功能。 例如，如果您不會在您的應用程式中使用角色，您不需要建立的角色或使用者角色關聯的儲存體。 您的技術和現有的基礎結構，可能需要與 ASP.NET Core 身分識別的預設實作非常不同的結構。 資料存取層，您提供的邏輯來處理您的儲存體實作的結構。

資料存取層提供邏輯以從 ASP.NET Core 身分識別的資料儲存至資料來源。 資料存取層，您自訂的存放裝置提供者可能會包含下列類別，以儲存使用者和角色的資訊。

### <a name="context-class"></a>Context 類別

封裝連接到您的持續性機制，並執行查詢的資訊。 數個資料類別需要這個類別，通常透過相依性插入所提供的執行個體。 [範例](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnet.identity.corecompat.identitydbcontext-1)。

### <a name="user-storage"></a>使用者的儲存體

儲存和擷取使用者資訊 （例如使用者名稱和密碼雜湊）。 [範例](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnet.identity.corecompat.userstore-1)

### <a name="role-storage"></a>角色存放裝置

儲存和擷取角色資訊 （例如角色名稱）。 [範例](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.entityframeworkcore.rolestore-1)

### <a name="userclaims-storage"></a>UserClaims 的儲存體

儲存和擷取使用者宣告資訊 （例如宣告類型和值）。 [範例](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnet.identity.corecompat.userstore-1)

### <a name="userlogins-storage"></a>Serlogins 的儲存體

儲存和擷取使用者登入資訊 （例如外部驗證提供者）。 [範例](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnet.identity.corecompat.userstore-1)

### <a name="userrole-storage"></a>使用者角色存放裝置

儲存和擷取哪些角色指派給使用者。 [範例](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnet.identity.corecompat.userstore-1)

**提示：**只實作您想要使用您的應用程式中的類別。

在資料存取類別中，提供程式碼以執行資料作業的持續性機制。 比方說，在自訂的提供者，您可能必須建立新的使用者，在下列程式碼*儲存*類別：

[!code-csharp[Main](identity-custom-storage-providers/sample/CustomIdentityProviderSample/CustomProvider/CustomUserStore.cs?name=createuser&highlight=7)]

建立使用者的實作邏輯處於``_usersTable.CreateAsync``方法，如下所示。

## <a name="customize-the-user-class"></a>自訂使用者類別

當實作的儲存提供者，建立使用者類別，其相當於[`IdentityUser`類別](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnet.identity.corecompat.identityuser)。

至少必須包含在使用者類別`Id`和`UserName`屬性。

`IdentityUser`類別定義的內容，``UserManager``呼叫時執行要求的作業。 預設類型`Id`屬性是一個字串，但您可以繼承自`IdentityUser<TKey, TUserClaim, TUserRole, TUserLogin, TUserToken>`並指定不同的類型。 架構會預期要處理的資料類型轉換的儲存體實作。

## <a name="customize-the-user-store"></a>自訂使用者存放區

建立`UserStore`提供的所有使用者資料作業方式的類別。 這個類別就相當於[UserStore<TUser> ](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.entityframeworkcore.userstore-1)類別。 在您`UserStore`類別，實作`IUserStore<TUser>`和所需的選用介面。 您選取要實作的選擇性介面上根據您的應用程式中提供的功能。

### <a name="optional-interfaces"></a>選擇性介面

- IUserRoleStore https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iuserrolestore-1
- IUserClaimStore https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iuserclaimstore-1
- IUserPasswordStore https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iuserpasswordstore-1
- IUserSecurityStampStore<!-- make these all links and remove / -->
- IUserEmailStore
- IPhoneNumberStore
- IQueryableUserStore
- IUserLoginStore
- IUserTwoFactorStore
- IUserLockoutStore

選擇性的介面繼承自`IUserStore`。 您可以查看儲存部分實作的範例使用者[這裡](https://github.com/aspnet/Docs/blob/master/aspnetcore/security/authentication/identity-custom-storage-providers/sample/CustomIdentityProviderSample/CustomProvider/CustomUserStore.cs)。

內`UserStore`類別，使用您建立用來執行作業的資料存取類別。 這些會在使用相依性插入傳遞。 例如，在 SQL Server Dapper 實作，`UserStore`類別具有`CreateAsync`方法使用的執行個體`DapperUsersTable`來插入新的記錄：

[!code-csharp[Main](identity-custom-storage-providers/sample/CustomIdentityProviderSample/CustomProvider/DapperUsersTable.cs?name=createuser&highlight=7)]

### <a name="interfaces-to-implement-when-customizing-user-store"></a>若要實作自訂使用者存放區時的介面

- **IUserStore**  
 [IUserStore&lt;TUser&gt; ](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iuserstore-1)介面是只有在使用者存放區中，您必須實作的介面。 它會定義方法來建立、 更新、 刪除和擷取使用者。
- **IUserClaimStore**  
 [IUserClaimStore&lt;TUser&gt; ](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iuserclaimstore-1)介面會定義您要啟用使用者宣告實作的方法。 它包含用於加入、 移除和擷取使用者宣告的方法。
- **IUserLoginStore**  
 [IUserLoginStore&lt;TUser&gt; ](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iuserloginstore-1)定義您要啟用外部驗證提供者實作的方法。 它包含加入、 移除和擷取使用者登入和擷取使用者為基礎的登入資訊方法的方法。
- **IUserRoleStore**  
 [IUserRoleStore&lt;TUser&gt; ](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iuserrolestore-1)介面會定義您將使用者對應至角色實作的方法。 它包含方法，以新增、 移除和擷取使用者的角色，並檢查使用者是否指派給角色的方法。
- **IUserPasswordStore**  
 [IUserPasswordStore&lt;TUser&gt; ](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iuserpasswordstore-1)介面會定義您實作來保存雜湊的密碼的方法。 它包含方法來取得和設定雜湊的密碼，並指出使用者是否已設定密碼的方法。
- **IUserSecurityStampStore**  
 [IUserSecurityStampStore&lt;TUser&gt; ](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iusersecuritystampstore-1)介面會定義您實作要用於安全性戳記，指出是否已變更的使用者帳戶資訊的方法。 當使用者變更密碼，或加入或移除登入，則會更新這個戳記。 它包含方法來取得和設定的安全性戳記。
- **IUserTwoFactorStore**  
 [IUserTwoFactorStore&lt;TUser&gt; ](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iusertwofactorstore-1)介面會定義您實作以支援雙因素驗證的方法。 它包含對取得和設定是否針對使用者啟用雙因素驗證的方法。
- **IUserPhoneNumberStore**  
 [IUserPhoneNumberStore&lt;TUser&gt; ](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iuserphonenumberstore-1)介面會定義您實作以儲存使用者電話號碼的方法。 它包含對取得和設定的電話號碼和電話號碼是否已確認的方法。
- **IUserEmailStore**  
 [IUserEmailStore&lt;TUser&gt; ](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iuseremailstore-1)介面會定義您實作以儲存使用者的電子郵件地址的方法。 它包含對取得和設定電子郵件地址和電子郵件是否已確認的方法。
- **IUserLockoutStore**  
 [IUserLockoutStore&lt;TUser&gt; ](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iuserlockoutstore-1)介面會定義您實作以儲存有關鎖定的帳戶資訊的方法。 它包含追蹤失敗的存取嘗試以及鎖定的方法。
- **IQueryableUserStore**  
 [IQueryableUserStore&lt;TUser&gt; ](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iqueryableuserstore-1)介面會定義提供可查詢使用者存放區的成員實作。

您在應用程式中實作所需的介面。 例如: 

```csharp
public class UserStore : IUserStore<IdentityUser>,
                         IUserClaimStore<IdentityUser>,
                         IUserLoginStore<IdentityUser>,
                         IUserRoleStore<IdentityUser>,
                         IUserPasswordStore<IdentityUser>,
                         IUserSecurityStampStore<IdentityUser>
{
    // interface implementations not shown
}
```

### <a name="identityuserclaim-identityuserlogin-and-identityuserrole"></a>IdentityUserClaim、 IdentityUserLogin 和 IdentityUserRole

``Microsoft.AspNet.Identity.EntityFramework``命名空間包含的實作[IdentityUserClaim](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.entityframeworkcore.identityuserclaim-1)， [IdentityUserLogin](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnet.identity.corecompat.identityuserlogin)，和[IdentityUserRole](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.entityframeworkcore.identityuserrole-1)類別。 如果您使用這些功能，您可能想要建立您自己的版本，這些類別，以及定義您的應用程式的屬性。 不過，有時候很不載入這些實體記憶體時執行 （例如加入或移除使用者的宣告） 的基本作業更有效率。 請改為後端存放區類別可以執行這些作業，直接針對資料來源。 例如，``UserStore.GetClaimsAsync``方法可以呼叫``userClaimTable.FindByUserId(user.Id)``方法上執行查詢，直接資料表，並傳回清單的宣告。

## <a name="customize-the-role-class"></a>自訂角色類別

當實作角色存放裝置提供者，您可以建立自訂角色型別。 它不需要實作特定介面，但是它必須有`Id`而且通常會有`Name`屬性。

以下是範例角色類別：

[!code-csharp[Main](identity-custom-storage-providers/sample/CustomIdentityProviderSample/CustomProvider/ApplicationRole.cs)]

## <a name="customize-the-role-store"></a>自訂角色存放區

您可以建立``RoleStore``提供的角色上的所有資料作業方式的類別。 這個類別就相當於[RoleStore<TRole> ](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.entityframeworkcore.rolestore-1)類別。 在`RoleStore`類別時，實作``IRoleStore<TRole>``並選擇性地``IQueryableRoleStore<TRole>``介面。

- **IRoleStore&lt;TRole&gt;**  
 [IRoleStore](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.irolestore-1)介面會定義角色存放區類別中實作的方法。 它包含建立、 更新、 刪除及擷取角色的方法。
- **RoleStore&lt;TRole&gt;**  
 若要自訂`RoleStore`，建立類別，實作`IRoleStore`介面。 

## <a name="reconfigure-app-to-use-new-storage-provider"></a>重新設定應用程式使用新的存放裝置提供者

一旦您已實作的儲存提供者，您會設定您的應用程式使用它。 如果您的應用程式使用預設提供者，請將它取代您的自訂提供者。

1. 移除`Microsoft.AspNetCore.EntityFramework.Identity`NuGet 封裝。
1. 如果存放裝置提供者位於不同的專案或封裝中，加入的參考。
1. 取代所有參考`Microsoft.AspNetCore.EntityFramework.Identity`使用存放裝置提供者的命名空間陳述式。
1. 在``ConfigureServices``方法，變更`AddIdentity`方法，以使用您的自訂型別。 您可以建立您自己的擴充方法，針對此目的。 請參閱[IdentityServiceCollectionExtensions](https://github.com/aspnet/Identity/blob/rel/1.1.0/src/Microsoft.AspNetCore.Identity/IdentityServiceCollectionExtensions.cs)的範例。
1. 如果您使用角色時，更新`RoleManager`使用您`RoleStore`類別。
1. 更新您的應用程式設定的連接字串和認證。

範例：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Add identity types
    services.AddIdentity<ApplicationUser, ApplicationRole>()
        .AddDefaultTokenProviders();

    // Identity Services
    services.AddTransient<IUserStore<ApplicationUser>, CustomUserStore>();
    services.AddTransient<IRoleStore<ApplicationRole>, CustomRoleStore>();
    string connectionString = Configuration.GetConnectionString("DefaultConnection");
    services.AddTransient<SqlConnection>(e => new SqlConnection(connectionString));
    services.AddTransient<DapperUsersTable>();

    // additional configuration
}
```

## <a name="references"></a>參考

- [ASP.NET 識別的自訂儲存體提供者](https://docs.microsoft.com/aspnet/identity/overview/extensibility/overview-of-custom-storage-providers-for-aspnet-identity)
- [ASP.NET Core 識別](https://github.com/aspnet/identity)-這個儲存機制包含社群維護存放區提供者的連結。
