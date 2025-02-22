---
title: Create provider-hosted SharePoint Add-ins to access SAP data
description: Design a SharePoint Add-in to get authorized access to SAP by using the SAP Gateway for Microsoft.
ms.date: 05/01/2020
ms.prod: sharepoint
ms.localizationpriority: medium
---

# Create provider-hosted SharePoint Add-ins to access SAP data by using the SAP Gateway for Microsoft

You can create a SharePoint Add-in that reads and writes SAP data, and optionally reads and writes SharePoint data, by using the SAP Gateway for Microsoft and the Azure Active Directory Authentication Library for .NET. This article provides the procedures for how you can design the SharePoint Add-in to get authorized access to SAP.

## Prerequisites

The following are the prerequisites to the procedures in this article:

- **An Office 365 Developer Site** in an Office 365 domain that is associated with a Microsoft Azure Active Directory  (Azure AD) subscription. See [Set up a development environment for SharePoint Add-ins on Office 365](set-up-a-development-environment-for-sharepoint-add-ins-on-office-365.md) or [Create a developer site on an existing Office 365 subscription](create-a-developer-site-on-an-existing-office-365-subscription.md).
- **Visual Studio 2013 Update 2** or later, which you can obtain from [Welcome to Visual Studio](https://msdn.microsoft.com/library/dd831853.aspx).
- **Microsoft Office Developer Tools for Visual Studio**. The version that is included in Update 2 of Visual Studio 2013 or later.
- **SAP Gateway for Microsoft** is deployed and configured in Azure. See the documentation for [SAP Gateway for Microsoft](https://help.sap.com/saphelp_nwgwpam_1/helpdata/en/b5/74d3ad2f33428193a32c09c351e0b3/frameset.htm).
- **An organizational account in Azure**. See [Manually register an Office 365 API app in Azure AD](https://github.com/jasonjoh/office365-azure-guides/blob/master/RegisterAnAppInAzure.md).

    > [!NOTE]
    > Sign in to your Office 365 account (login.microsoftonline.com) to change the temporary password after the account is created.

- **A SAP OData endpoint** with sample data in it. See the documentation for [SAP Gateway for Microsoft](https://help.sap.com/saphelp_nwgwpam_1/helpdata/en/b5/74d3ad2f33428193a32c09c351e0b3/frameset.htm).
- **A basic familiarity with Azure AD.** See [Get started with Azure AD](/azure/active-directory/get-started-azure-ad).
- **A basic familiarity with creating SharePoint Add-ins.** See [Get started creating provider-hosted SharePoint Add-ins](get-started-creating-provider-hosted-sharepoint-add-ins.md).
- **A basic familiarity with OAuth 2.0 in Azure AD**. See [Authorize access to web applications using OAuth 2.0 and Azure Active Directory](/azure/active-directory/develop/active-directory-protocols-oauth-code) and its child topics.
- **Code sample:** [SharePoint: Using the SAP Gateway to Microsoft in a SharePoint Add-in](https://code.msdn.microsoft.com/office/sharepoint-2013-using-the-0931abce)

## Understand authentication and authorization to SAP Gateway for Microsoft and SharePoint

OAuth 2.0 in Azure AD enables applications to access multiple resources hosted by Azure, and SAP Gateway for Microsoft is one of them. With OAuth 2.0, applications, in addition to users, are security principals. Application principals require authentication and authorization to protected resources in addition to (and sometimes instead of) users.

The process involves an OAuth "flow" in which the application, which can be a SharePoint Add-in, obtains an access token (and refresh token) that is accepted by all of the Azure-hosted services and applications that are configured to use Azure AD as an OAuth 2.0 authorization server. The process is very similar to the way that the remote components of a provider-hosted SharePoint Add-in gets authorization to SharePoint as described in [Creating SharePoint Add-ins that use low-trust authorization](creating-sharepoint-add-ins-that-use-low-trust-authorization.md) and its child articles. However, the ACS authorization system uses Azure Access Control Service (ACS) as the trusted token issuer rather than Azure AD.

> [!TIP]
> If your SharePoint Add-in accesses SharePoint in addition to accessing SAP Gateway for Microsoft, it needs to use *both* systems: Azure AD to get an access token to SAP Gateway for Microsoft, and the ACS authorization system to get an access token to SharePoint. The tokens from the two sources are not interchangeable. For more information, see [Add SharePoint access to the ASP.NET application (optional)](#add-sharepoint-access-to-the-aspnet-application-optional).

> [!IMPORTANT]
> Azure Access Control (ACS), a service of Azure Active Directory (Azure AD), will be retired on November 7, 2018. This retirement does not impact the SharePoint Add-in model, which uses the `https://accounts.accesscontrol.windows.net` hostname (which is not impacted by this retirement). For more information, see [Impact of Azure Access Control retirement for SharePoint Add-ins](https://developer.microsoft.com/office/blogs/impact-of-azure-access-control-deprecation-for-sharepoint-add-ins).

For a detailed description and diagram of the OAuth flow used by OAuth 2.0 in Azure AD, see [Authorize access to web applications using OAuth 2.0 and Azure Active Directory](/azure/active-directory/develop/active-directory-protocols-oauth-code).

For a similar description and a diagram of the flow for accessing SharePoint, see [the steps in the Context Token flow](context-token-oauth-flow-for-sharepoint-add-ins.md#context-token-flow-steps).

## Create the SharePoint Add-in

### To create the Visual Studio solution

1. Create a **SharePoint Add-in** project in Visual Studio with the following steps. (The continuing example in this article assumes you are using C#, but you can start a SharePoint Add-in project in the **Visual Basic** section of the new project templates as well.)

    1. In the New SharePoint Add-in Wizard, name the project, and select **OK**. For the continuing example, use **SAP2SharePoint**.
    1. Specify the domain URL of your Office 365 Developer Site (including a final forward slash) as the debugging site; for example, `https://<O365_domain>.sharepoint.com/`. Specify **Provider-hosted** as the add-in type, and then select **Next**.
    1. Select a project type. For the continuing example, select **ASP.NET Web Forms Application**. (You can also make ASP.NET MVC applications that access SAP Gateway for Microsoft.) Select **Next**.
    1. Select Azure ACS as the authentication system. (Your SharePoint Add-in uses this system if it accesses SharePoint. It does not use this system when it accesses SAP Gateway for Microsoft.) Select **Finish**.

1. After the project is created, you are prompted to sign in to the Office 365 account. Use the credentials of an account administrator; for example `Bob@<O365_domain>.onmicrosoft.com`.
1. There are two projects in the Visual Studio solution: the SharePoint Add-in proper project and an ASP.NET web forms project. Add the **Microsoft Authentication Library** (MSAL) package to the ASP.NET project with these steps:

    1. Right-click the **References** folder in the ASP.NET project (named **SAP2SharePointWeb** in the continuing example), and select **Manage NuGet Packages**.
    1. In the dialog that opens, select **Online** on the left. Enter **Microsoft.IdentityModel.Clients.ActiveDirectory** in the search box.
    1. When the MSAL library appears in the search results, select the **Install** button, and accept the license when prompted.

1. Add the Json.net package to the ASP.NET project with these steps:

    1. Enter **Json.net** in the search box. If this produces too many hits, try searching on **Newtonsoft.json**.
    1. When Json.net appears in the search results, select the **Install** button.

1. Select **Close**.

### To register your web application with Azure AD

1. Sign in to the [Azure portal](https://ms.portal.azure.com) with your Azure administrator account.

  > [!NOTE]
  > For security purposes, we recommend against using an administrator account when developing add-ins.

1. Select **Active Directory** on the left.
1. Select your directory.
1. Select **APPLICATIONS** (on the top navigation bar).
1. Select **Add** on the toolbar at the bottom of the screen.
1. On the dialog that opens, select **Add an application my organization is developing**.
1. In the **ADD APPLICATION** dialog, give the application a name. For the continuing example, use **ContosoAutomobileCollection**.
1. Select **Web Application And/Or Web API** as the application type, and then select the right arrow button.
1. On the second page of the dialog, use the SSL debugging URL from the ASP.NET project in the Visual Studio solution as the **SIGN-ON URL**. You can find the URL using the following steps. *(You need to register the add-in initially with the debugging URL so that you can run the Visual Studio debugger (F5). When your add-in is ready for staging, you will re-register it with its staging Azure website URL. See [Modify the add-in and stage it to Azure and Office 365](#modify-the-add-in-and-stage-it-to-azure-and-office-365).)*

    1. Highlight the ASP.NET project in **Solution Explorer**.
    1. In the **Properties** window, copy the value of the **SSL URL** property. An example is `https://localhost:44300/`.
    1. Paste it into the **SIGN-ON URL** on the **ADD APPLICATION** dialog.

1. For the **APP ID URI**, give the application a unique URI, such as the application name appended to the end of the SSL URL; for example `https://localhost:44300/ContosoAutomobileCollection`.
1. Select the checkmark button. The Azure dashboard for the application opens with a success message.
1. Select **CONFIGURE** on the top of the page.
1. Scroll to the **CLIENT ID** and make a copy of it. You will need it for a later procedure.
1. In the **keys** section, create a key. It won't appear initially. Select **SAVE** at the bottom of the page and the key will be visible. Make a copy of it. You will need it for a later procedure.
1. Scroll to **permissions to other applications**, and select your SAP Gateway for Microsoft service application.
1. Open the **Delegated Permissions** drop-down list and enable the boxes for the permissions to the SAP Gateway for Microsoft service that your SharePoint Add-in will need.
1. Select **SAVE** at the bottom of the screen.

### To configure the application to communicate with Azure AD

1. In Visual Studio, open the web.config file in the ASP.NET project.
1. In the `<appSettings>` section, the Office Developer Tools for Visual Studio have added elements for the **ClientID** and **ClientSecret** of the SharePoint Add-in. (These are used in the Azure ACS authorization system if the ASP.NET application accesses SharePoint. You can ignore them for the continuing example, but do not delete them. They are required in provider-hosted SharePoint Add-ins even if the add-in is not accessing SharePoint data. Their values change each time you select F5 in Visual Studio.)

  Add the following two elements to the section. These are used by the application to authenticate to Azure AD. (Remember that applications, as well as users, are security principals in OAuth-based authentication and authorization systems.)

  ```xml
  <add key="ida:ClientID" value="" />
  <add key="ida:ClientKey" value="" />
  ```

1. Insert the Client ID that you saved from your Azure AD directory in the earlier procedure as the value of the **ida:ClientID** key. Leave the casing and punctuation exactly as you copied it, and be careful not to include a space character at the beginning or end of the value. For the **ida:ClientKey** key, use the *key*  that you saved from the directory. Again, be careful not to introduce any space characters or change the value in any way. The `<appSettings>` section should now look something like the following (the **ClientId** value may have a GUID or an empty string):

    ```xml
    <appSettings>
      <add key="ClientId" value="" />
      <add key="ClientSecret" value="LypZu2yVajlHfPLRn5J2hBrwCk5aBOHxE4PtKCjIQkk=" />
      <add key="ida:ClientID" value="4da99afe-08b5-4bce-bc66-5356482ec2df" />
      <add key="ida:ClientKey" value="URwh/oiPay/b5jJWYHgkVdoE/x7gq3zZdtcl/cG14ss=" />
    </appSettings>
    ```

    > [!NOTE]
    > Your application is known to Azure AD by the "localhost" URL you used to register it. The client ID and client key are associated with that identity. When you are ready to stage your application to an Azure website, you will re-register it with a new URL.

1. Still in the **appSettings** section, add an **Authority** key and set its value to the Office 365 domain (*some_domain*.onmicrosoft.com) of your organizational account. In the continuing example, the organizational account is `Bob@<O365_domain>.onmicrosoft.com`, so the authority is `<O365_domain>.onmicrosoft.com`.

    ```xml
    <add key="Authority" value="<O365_domain>.onmicrosoft.com" />
    ```

1. Still in the **appSettings** section, add an **AppRedirectUrl** key and set its value to the page that the user's browser should be redirected to after the ASP.NET add-in has obtained an authorization code from Azure AD. Usually, this is the same page that the user was on when the call to Azure AD was made. In the continuing example, use the SSL URL value with "/Pages/Default.aspx" appended to it as shown in the following code (this is another value that you will change for staging):

    ```xml
    <add key="AppRedirectUrl" value="https://localhost:44322/Pages/Default.aspx" />
    ```

1. Still in the **appSettings** section, add a **ResourceUrl** key and set its value to the APP ID URI of SAP Gateway for Microsoft (*not*  the APP ID URI of your ASP.NET application). Obtain this value from the SAP Gateway for Microsoft administrator. The following is an example.

    ```xml
    <add key="ResourceUrl" value="http://<SAP_gateway_domain>.cloudapp.net/" />
    ```

      The `<appSettings>` section should now look something like this:

    ```xml
    <appSettings>
      <add key="ClientId" value="06af1059-8916-4851-a271-2705e8cf53c6" />
      <add key="ClientSecret" value="LypZu2yVajlHfPLRn5J2hBrwCk5aBOHxE4PtKCjIQkk=" />
      <add key="ida:ClientID" value="4da99afe-08b5-4bce-bc66-5356482ec2df" />
      <add key="ida:ClientKey" value="URwh/oiPay/b5jJWYHgkVdoE/x7gq3zZdtcl/cG14ss=" />
      <add key="Authority" value="<O365_domain>.onmicrosoft.com" />
      <add key="AppRedirectUrl" value="https://localhost:44322/Pages/Default.aspx" />
      <add key="ResourceUrl" value="http://<SAP_gateway_domain>.cloudapp.net/" />
    </appSettings>
    ```

1. Save and close the web.config file.

    > [!TIP]
    > Do not leave the web.config file open when you run the Visual Studio debugger (F5). The Office Developer Tools for Visual Studio change the **ClientId** value (not the **ida:ClientID**) every time you select F5. This requires you to respond to a prompt to reload the web.config file, if it is open, before debugging can execute.

### To add a helper class to authenticate to Azure AD

1. Right-click the ASP.NET project and use the Visual Studio item adding process to add a new class file to the project named **AADAuthHelper.cs**.
1. Add the following **using** statements to the file.

    ```csharp
    using Microsoft.IdentityModel.Clients.ActiveDirectory;
    using System.Configuration;
    using System.Web.UI;
    ```

1. Change the access keyword from **public** to **internal**, and add the **static** keyword to the class declaration.

    ```csharp
    internal static class AADAuthHelper
    ```

1. Add the following fields to the class. These fields store information that your ASP.NET application uses to get access tokens from Azure AD.

    ```csharp
    private static readonly string _authority = ConfigurationManager.AppSettings["Authority"];
    private static readonly string _appRedirectUrl = ConfigurationManager.AppSettings["AppRedirectUrl"];
    private static readonly string _resourceUrl = ConfigurationManager.AppSettings["ResourceUrl"];
    private static readonly string _clientId = ConfigurationManager.AppSettings["ida:ClientID"];
    private static readonly ClientCredential _clientCredential = new ClientCredential(
                              ConfigurationManager.AppSettings["ida:ClientID"],
                              ConfigurationManager.AppSettings["ida:ClientKey"]);

    private static readonly AuthenticationContext _authenticationContext =
                new AuthenticationContext("https://login.windows.net/common/" +
                                          ConfigurationManager.AppSettings["Authority"]);
    ```

1. Add the following property to the class. This property holds the URL to the Azure AD sign-in screen.

    ```csharp
    private static string AuthorizeUrl
    {
      get
      {
        return string.Format("https://login.windows.net/{0}/oauth2/authorize?response_type=code&amp;redirect_uri={1}&amp;client_id={2}&amp;state={3}",
            _authority,
            _appRedirectUrl,
            _clientId,
            Guid.NewGuid().ToString());
      }
    }
    ```

1. Add the following properties to the class. These cache the access and refresh tokens and check their validity.

    ```csharp
    public static Tuple<string, DateTimeOffset> AccessToken
    {
      get {
        return HttpContext.Current.Session["AccessTokenWithExpireTime-" + _resourceUrl]
            as Tuple<string, DateTimeOffset>;
      }

      set { HttpContext.Current.Session["AccessTokenWithExpireTime-" + _resourceUrl] = value; }
    }

    private static bool IsAccessTokenValid
    {
      get
      {
        return AccessToken != null
          && !string.IsNullOrEmpty(AccessToken.Item1)
          && AccessToken.Item2 > DateTimeOffset.UtcNow;
      }
    }

    private static string RefreshToken
    {
      get { return HttpContext.Current.Session["RefreshToken" + _resourceUrl] as string; }
      set { HttpContext.Current.Session["RefreshToken-" + _resourceUrl] = value; }
    }

    private static bool IsRefreshTokenValid
    {
      get { return !string.IsNullOrEmpty(RefreshToken); }
    }
    ```

1. Add the following methods to the class. These are used to check the validity of the authorization code and to obtain an access token from Azure AD by using either an authentication code or a refresh token.

    ```csharp
    private static bool IsAuthorizationCodeNotNull(string authCode)
    {
      return !string.IsNullOrEmpty(authCode);
    }

    private static Tuple<Tuple<string,DateTimeOffset>,string> AcquireTokensUsingAuthCode(string authCode)
    {
      var authResult = _authenticationContext.AcquireTokenByAuthorizationCode(
                  authCode,
                  new Uri(_appRedirectUrl),
                  _clientCredential,
                  _resourceUrl);

      return new Tuple<Tuple<string, DateTimeOffset>, string>(
                  new Tuple<string, DateTimeOffset>(authResult.AccessToken, authResult.ExpiresOn),
                  authResult.RefreshToken);
    }

    private static Tuple<string, DateTimeOffset> RenewAccessTokenUsingRefreshToken()
    {
      var authResult = _authenticationContext.AcquireTokenByRefreshToken(
                          RefreshToken,
                          _clientCredential.OwnerId,
                          _clientCredential,
                          _resourceUrl);

      return new Tuple<string, DateTimeOffset>(authResult.AccessToken, authResult.ExpiresOn);
    }
    ```

1. Add the following method to the class. It is called from the ASP.NET code-behind to obtain a valid access token before a call is made to get SAP data via SAP Gateway for Microsoft.

    ```csharp
    internal static void EnsureValidAccessToken(Page page)
    {
      if (IsAccessTokenValid)
      {
        return;
      }
      else if (IsRefreshTokenValid)
      {
        AccessToken = RenewAccessTokenUsingRefreshToken();
        return;
      }
      else if (IsAuthorizationCodeNotNull(page.Request.QueryString["code"]))
      {
          Tuple<Tuple<string, DateTimeOffset>, string> tokens = null;
          try
          {
            tokens = AcquireTokensUsingAuthCode(page.Request.QueryString["code"]);
          }
          catch
          {
            page.Response.Redirect(AuthorizeUrl);
          }
          AccessToken = tokens.Item1;
          RefreshToken = tokens.Item2;
        return;
      }
      else
      {
        page.Response.Redirect(AuthorizeUrl);
      }
    }
    ```

> [!TIP]
> The `AADAuthHelper` class has only minimal error handling. For a robust, production quality SharePoint Add-in, add more error handling as described in this MSDN node: [Authorize access to web applications using OAuth 2.0 and Azure Active Directory](/azure/active-directory/develop/active-directory-protocols-oauth-code).

### To create data model classes

1. Create one or more classes to model the data that your add-in gets from SAP. In the continuing example, there is just one data model class. Right-click the ASP.NET project and use the Visual Studio item-adding process to add a new class file to the project named **Automobile.cs**.
1. Add the following code to the body of the class:

    ```csharp
    public string Price;
    public string Brand;
    public string Model;
    public int Year;
    public string Engine;
    public int MaxPower;
    public string BodyStyle;
    public string Transmission;
    ```

### To add code-behind to get data from SAP via the SAP Gateway for Microsoft

1. Open the Default.aspx.cs file, and add the following **using** statements.

    ```csharp
    using System.Net;
    using Newtonsoft.Json.Linq;
    ```

1. Add a **const** declaration to the Default class whose value is the base URL of the SAP OData endpoint that the add-in will be accessing. The following is an example:

    ```csharp
    private const string SAP_ODATA_URL = @"https://<SAP_gateway_domain>.cloudapp.net:8081/perf/sap/opu/odata/sap/ZCAR_POC_SRV/";
    ```

2. The Office Developer Tools for Visual Studio have added a **Page\_PreInit** method and a **Page\_Load** method. Comment out the code inside the **Page\_Load** method and comment out the whole **Page\_Init** method. This code is not used in this sample. (If your SharePoint Add-in is going to access SharePoint, restore this code. See [Add SharePoint access to the ASP.NET application (optional)](#add-sharepoint-access-to-the-aspnet-application-optional).)
1. Add the following line to the top of the **Page_Load** method. This eases the process of debugging because your ASP.NET application is communicating with SAP Gateway for Microsoft using SSL (HTTPS), but your "localhost:port" server is not configured to trust the certificate of SAP Gateway for Microsoft. Without this line of code, you would get an invalid certificate warning before Default.aspx opens. Some browsers allow you to click past this error, but some will not let you open Default.aspx at all.

    ```csharp
    ServicePointManager.ServerCertificateValidationCallback = (s, cert, chain, errors) => true;
    ```

    > [!IMPORTANT]
    > Delete this line when you are ready to deploy the ASP.NET application to staging. See [Modify the add-in and stage it to Azure and Office 365](#modify-the-add-in-and-stage-it-to-azure-and-office-365).

1. Add the following code to the **Page_Load** method. The string you pass to the `GetSAPData` method is an OData query.

    ```csharp
    if (!IsPostBack)
    {
      GetSAPData("DataCollection?$top=3");
    }
    ```

1. Add the following method to the Default class. This method first ensures that the cache for the access token has a valid access token in it that has been obtained from Azure AD. It then creates an HTTP **GET** Request that includes the access token and sends it to the SAP OData endpoint. The result is returned as a JSON object that is converted to a .NET **List** object. Three properties of the items are used in an array that is bound to the **DataListView**.

    ```csharp
    private void GetSAPData(string oDataQuery)
    {
      AADAuthHelper.EnsureValidAccessToken(this);

      using (WebClient client = new WebClient())
      {
        client.Headers[HttpRequestHeader.Accept] = "application/json";
        client.Headers[HttpRequestHeader.Authorization] = "Bearer " + AADAuthHelper.AccessToken.Item1;
        var jsonString = client.DownloadString(SAP_ODATA_URL + oDataQuery);
        var jsonValue = JObject.Parse(jsonString)["d"]["results"];
        var dataCol = jsonValue.ToObject<List<Automobile>>();

        var dataList = dataCol.Select((item) => {
            return item.Brand + " " + item.Model + " " + item.Price;
            }).ToArray();

        DataListView.DataSource = dataList;
        DataListView.DataBind();
      }
    }
    ```

### To create the user interface

1. Open the Default.aspx file and add the following markup to the **form** of the page:

    ```xml
    <div>
    <h3>Data from SAP via SAP Gateway for Microsoft</h3>

    <asp:ListView runat="server" ID="DataListView">
      <ItemTemplate>
        <tr runat="server">
          <td runat="server">
            <asp:Label ID="DataLabel" runat="server"
              Text="<%# Container.DataItem.ToString()%>" /><br />
          </td>
        </tr>
      </ItemTemplate>
    </asp:ListView>
    </div>
    ```

1. Optionally, give the webpage the "look 'n' feel" of a SharePoint page with the SharePoint [chrome control](use-the-client-chrome-control-in-sharepoint-add-ins.md) and [the host SharePoint website's style sheet](use-a-sharepoint-website-s-style-sheet-in-sharepoint-add-ins.md).

### To test the add-in with F5 in Visual Studio

1. Select F5 in Visual Studio.
1. The first time that you use F5, you may be prompted to sign in to the Developer Site that you are using. Use the site administrator credentials. In the continuing example, it is `Bob@<O365_domain>.onmicrosoft.com`.
1. The first time that you use F5, you are prompted to grant permissions to the add-in. Select **Trust It**.
1. After a brief delay while the access token is being obtained, the Default.aspx page opens. Verify that the SAP data appears.

## Add SharePoint access to the ASP.NET application (optional)

Of course, your SharePoint Add-in doesn't have to expose only SAP data in a webpage launched from SharePoint. It can also create, read, update, and delete (CRUD) SharePoint data. Your code-behind can do this by using either the SharePoint client object model (CSOM) or the REST APIs of SharePoint. The CSOM is deployed as a pair of assemblies that the Office Developer Tools for Visual Studio automatically included in the ASP.NET project and set to **Copy Local** in Visual Studio so that they are included in the ASP.NET application package.

For information about using CSOM, start with [Complete basic operations using SharePoint client library code](complete-basic-operations-using-sharepoint-client-library-code.md). For information about using the REST APIs, start with  [Understanding and Using the SharePoint REST Interface](https://msdn.microsoft.com/magazine/dn198245.aspx).

Regardless of whether you use CSOM or the REST APIs to access SharePoint, your ASP.NET application must get an access token to SharePoint, just as it does to SAP Gateway for Microsoft. See [Understand authentication and authorization to SAP Gateway for Microsoft and SharePoint](#understand-authentication-and-authorization-to-sap-gateway-for-microsoft-and-sharepoint) earlier in this article.

The following procedure provides some basic guidance about how to do this, but we recommend that you first read these articles:

- [Get started creating provider-hosted SharePoint Add-ins](get-started-creating-provider-hosted-sharepoint-add-ins.md)
- [Authorization and authentication of SharePoint Add-ins](authorization-and-authentication-of-sharepoint-add-ins.md)
- [Three authorization systems for SharePoint Add-ins](three-authorization-systems-for-sharepoint-add-ins.md)
- [Creating SharePoint Add-ins that use low-trust authorization](creating-sharepoint-add-ins-that-use-low-trust-authorization.md)
- [Context Token OAuth flow for SharePoint Add-ins](context-token-oauth-flow-for-sharepoint-add-ins.md)

### To add SharePoint access to the ASP.NET application

1. Open the Default.aspx.cs file and uncomment the `Page_PreInit` method. Also uncomment the code that the Office Developer Tools for Visual Studio added to the `Page_Load` method.
1. If your SharePoint Add-in is going to access SharePoint data, you have to cache the SharePoint context token that is POSTed to the Default.aspx page when the add-in is launched in SharePoint. This is to ensure that the SharePoint context token is not lost when the browser is redirected following the Azure AD authentication. (You have several options for how to cache this context.) The Office Developer Tools for Visual Studio add a SharePointContext.cs file to the ASP.NET project that does most of the work. To use the session cache, you simply add the following code inside the `if (!IsPostBack)` block *before* the code that calls out to SAP Gateway for Microsoft.

    ```csharp
    if (HttpContext.Current.Session["SharePointContext"] == null)
    {
        HttpContext.Current.Session["SharePointContext"]
            = SharePointContextProvider.Current.GetSharePointContext(Context);
    }
    ```

1. The **SharePointContext.cs** file makes calls to another file that the Office Developer Tools for Visual Studio added to the project: TokenHelper.cs. This file provides most of the code needed to obtain and use access tokens for SharePoint. However, it does not provide any code for renewing an expired access token or an expired refresh token. Nor does it contain any token caching code. For a production quality SharePoint Add-in, you need to add such code. The caching logic in the preceding step is an example. Your code should also cache the access token and reuse it until it expires. When the access token is expired, your code should use the refresh token to get a new access token.
1. Add the data calls to SharePoint by using either CSOM or REST. The following example is a modification of CSOM code that Office Developer Tools for Visual Studio adds to the **Page_Load** method. In this example, the code has been moved to a separate method and it begins by retrieving the cached context token.

    ```csharp
    private void GetSharePointTitle()
    {
        var spContext = HttpContext.Current.Session["SharePointContext"] as SharePointContext;
        using (var clientContext = spContext.CreateUserClientContextForSPHost())
        {
            clientContext.Load(clientContext.Web, web => web.Title);
            clientContext.ExecuteQuery();
            SharePointTitle.Text = "SharePoint website title is: " + clientContext.Web.Title;
        }
    }
    ```

1. Add UI elements to render the SharePoint data. The following shows the HTML control that is referenced in the preceding method:

    ```html
    <h3>SharePoint title</h3>
    <asp:Label ID="SharePointTitle" runat="server"></asp:Label><br />
    ```

    > [!NOTE]
    > While you are debugging the SharePoint Add-in, the Office Developer Tools for Visual Studio re-register it with Azure ACS each time you select F5 in Visual Studio. When you stage the SharePoint Add-in, you have to give it a long-term registration. See [Modify the add-in and stage it to Azure and Office 365](#modify-the-add-in-and-stage-it-to-azure-and-office-365).

## Modify the add-in and stage it to Azure and Office 365

When you have finished debugging the SharePoint Add-in using F5 in Visual Studio, you need to deploy the ASP.NET application to an actual Azure website.

### To create the Azure website

1. In the Microsoft Azure portal, open **websites** on the left navigation bar.
1. Select **NEW** at the bottom of the page, and in the **NEW** dialog, select **website** |**QUICK CREATE**.

1. Enter a domain name and select **CREATE website**. Make a copy of the new site's URL. It has the form  **my_domain.azurewebsites.net**.

### To modify the code and markup in the application

1. In Visual Studio, remove the line `ServicePointManager.ServerCertificateValidationCallback = (s, cert, chain, errors) => true;` from the **Default.aspx.cs** file.
1. Open the **web.config** file of the ASP.NET project and change the domain part of the value of the `AppRedirectUrl` key in the `appSettings` section to the domain of the Azure website. For example, change `<add key="AppRedirectUrl" value="https://localhost:44322/Pages/Default.aspx" />` to `<add key="AppRedirectUrl" value="https://my_domain.azurewebsites.net/Pages/Default.aspx" />`.
1. Right-click the **AppManifest.xml** file in the SharePoint Add-in project, and select **View Code**.
1. In the `StartPage` value, replace the string `~remoteAppUrl` with the full domain of the Azure website including the protocol; for example `https://my_domain.azurewebsites.net`. The entire `StartPage` value should now be `https://my_domain.azurewebsites.net/Pages/Default.aspx` (usually, the `StartPage` value is exactly the same as the value of the `AppRedirectUrl` key in the **web.config** file).

### To modify the Azure AD registration and register the add-in with ACS

1. Sign in to the [Azure portal](https://ms.portal.azure.com) with your Azure administrator account.
1. Select **Active Directory** on the left side.
1. Select your directory.
1. Select **APPLICATIONS** (on the top navigation bar).
1. Open the application you created. In the continuing example, it is **ContosoAutomobileCollection**.
1. For each of the following values, change the "localhost:*port*" part of the value to the domain of your new Azure website:

    - **SIGN-ON URL**
    - **APP ID URI**
    - **REPLY URL**

    For example, if the **APP ID URI** is `https://localhost:44304/ContosoAutomobileCollection`, change it to `https://<my_domain>.azurewebsites.net/ContosoAutomobileCollection`.

1. Select **SAVE** at the bottom of the screen.
1. Register the add-in with Azure ACS. This must be done even if the add-in does not access SharePoint and does not use tokens from ACS because the same process also registers the add-in with the Add-in Management Service of the Office 365 subscription, which is a requirement. (It is called "Add-in Management Service" because SharePoint Add-ins were originally called "apps for SharePoint.")

    You perform the registration on the AppRegNew.aspx page of any SharePoint website in the Office 365 subscription. For details, see [Register SharePoint Add-ins](register-sharepoint-add-ins.md).

    As part of this process, you obtain a new Client ID and Client Secret. Insert these values in the web.config for the **ClientId** (not **ida:ClientID**) and **ClientSecret** keys.

    > [!WARNING]
    > If for any reason you select F5 after making this change, the Office Developer Tools for Visual Studio overwrites one or both of these values. For that reason, you should keep a record of the values obtained with AppRegNew.aspx and always verify that the values in the web.config are correct just before you publish the ASP.NET application.

### Publish the ASP.NET application to Azure and install the add-in to SharePoint

1. There are several ways to publish an ASP.NET application to an Azure website. For more information, see [Local Git Deployment to Azure App Service](/azure/app-service/app-service-deploy-local-git).
1. In Visual Studio, right-click the SharePoint Add-in project and select **Package**. On the **Publish your add-in** page that opens, select **Package the add-in**. File Explorer opens to the folder with the add-in package.
1. Sign in to Office 365 as a global administrator, and navigate to the organization add-in catalog site collection. (If there isn't one, create it. See [Use the Add-in Catalog to make custom business add-ins available for your SharePoint Online environment](https://support.office.com/article/Use-the-App-Catalog-to-make-custom-business-apps-available-for-your-SharePoint-Online-environment-0b6ab336-8b83-423f-a06b-bcc52861cba0?CorrelationId=40e9dc6b-213e-4496-8c5f-40ca27922fa1&ui=en-US&rs=en-US&ad=US&ocmsassetID=HA102772362).)
1. Upload the add-in package to the add-in catalog.
1. Navigate to the **Site Contents** page of any website in the subscription, and select **add an add-in**.
1. On the **Your Add-ins** page, scroll to the **Add-ins you can add** section and select the icon for your add-in.
1. After the add-in has installed, select its icon on the **Site Contents** page to launch the add-in.

For more information about installing SharePoint Add-ins, see [Deploying and installing SharePoint Add-ins: methods and options](deploying-and-installing-sharepoint-add-ins-methods-and-options.md).

## Deploy the add-in to production

When you have finished all testing, you can deploy the add-in in production. This may require some changes.

1. If the production domain of the ASP.NET application is different from the staging domain, you have to change the `AppRedirectUrl` value in the **web.config** and the `StartPage` value in the **AppManifest.xml** file, and repackage the SharePoint Add-in. See the procedure [To modify the code and markup in the application](#to-modify-the-code-and-markup-in-the-application) earlier in this article.
1. The change in domain also requires that you edit the add-ins registration with Azure AD. See the procedure [To modify the Azure AD registration and register the add-in with ACS](#to-modify-the-azure-ad-registration-and-register-the-add-in-with-acs) earlier in this article.
1. The change in domain also requires that you re-register the add-in with ACS (and the subscription's Add-in Management Service) as described in the same procedure. (There is no way to *edit* an add-in's registration with ACS.) However, it is not necessary to generate a new Client ID or Client Secret on the **AppRegNew.aspx** page. You can copy the original values from the `ClientId` (not `ida:ClientID`) and `ClientSecret` keys of the **web.config** into the AppRegNew form. *If you do generate new ones, be sure to copy the new values to the keys in **web.config**.*

## See also

- [Develop SharePoint Add-ins](develop-sharepoint-add-ins.md)
