
<head>
    Despliegue de proyecto .NET9 WebApplication Razor Pages a WebApp en azure, con configuración de web app como "code", sin usar dockerFile, desplegando según el ambiente, usandi Federated credentials por ambiente, usando artifact de git hub
</head>



Se despliega mediante un Action en GitHUb, el cual usa Microsoft Entra ID para autenticar.

Así funciona el flujo(son dos pasos, build-web-app debe ser correcto para que continue con lo siguiente):

    build-web-app
      -compila
      -publica
      -guarda /publish como artifact
      
    deploy-web-app
      -espera a que termine build (needs)
      -descarga el artifact
      -despliega a Azure

  

Pasos para tener acceso al servicio en azure y configuraciones:
https://macelabs.com/deploying-to-azure-using-github-actions/?utm_source=youtube&utm_medium=video&utm_campaign=azure-github-actions-deploy&utm_content=description
https://youtu.be/xUsmCRBsyZM




1. Create Azure Resources via Portal
  1.1 Create a Resource Group
    A resource group is a container within Azure that holds related resources for a solution. Using a Resource Group helps organize and manage your Azure assets by grouping them together logically. All resources in a group can share a lifecycle, allowing you to easily deploy, update, or delete them together.
    
    Go to the Azure Portal and sign in.
    In the left-hand menu, select Resource groups.
    Click + Create.
    Provide:
    Subscription: Choose your subscription.
    Resource group name: e.g., MyDotNet9RG.
    Region: e.g., East US.
    Click Review + create, then Create.
   
  1.2 Create an App Service Plan (Linux)
    An App Service Plan defines the compute resources that your Web App runs on. It determines the pricing tier, scalability options, and more. You need an App Service Plan to host your Web App on Azure.
  
    In the Portal, click Create a resource.
    Search for App Service Plan and select Create.
    On the Basics tab:
    Subscription: Same as above.
    Resource group: MyDotNet9RG.
    Name: e.g., MyDotNet9Plan.
    Operating System: Linux.
    Region: e.g., East US (same region as resource group).
    SKU and size: e.g., Standard (S1).
    Click Review + create, then Create.
    Note: If you see a quota error (e.g., “This region has quota of 0 instances”), try a different region, SKU (like Basic/Free), or request a quota increase.
    
  1.3 Create the Web App (Linux)
    A Web App is a managed Azure service for hosting your .NET (or other) application. By deploying to a Web App, you benefit from Azure’s scaling, monitoring, and runtime management without having to manage your own servers.
    
    In the Portal, click Create a resource again.
    Search for Web App and click Create.
    On the Basics tab:
    Subscription: Same as before.
    Resource group: MyDotNet9RG.
    Name: e.g., MyDotNet9LinuxApp (must be unique).
    Publish: Code.
    Runtime stack: .NET 9 (or the nearest available version).
    Operating System: Linux.
    Region: e.g., East US.
    Linux Plan: Select MyDotNet9Plan.
    Review + create, then Create.
    At this point you should have a:
    
    A Resource Group: MyDotNet9RG
    An App Service Plan: MyDotNet9Plan
    A Web App (Linux): MyDotNet9LinuxApp
  2.1 Register an App in Microsoft Entra ID
    Microsoft Entra ID (formerly Azure Active Directory) is a cloud-based identity and access management service from Microsoft. It provides features for authentication, authorization, and centralized identity management across your applications and services. In the context of this deployment scenario, we use Microsoft Entra ID to issue secure, short-lived OpenID Connect (OIDC) tokens to GitHub Actions instead of relying on static credentials. This approach minimizes the security risk by avoiding long-lived secrets and ensures that your Azure resources are accessed only as needed, on a per-workflow basis.
    
    In the Azure Portal, search for Microsoft Entra ID (formerly Azure Active Directory).
    Click App registrations in the left menu.
    Click New registration.
    Provide:
    Name: e.g., MyDotNet9AppOIDC.
    Supported account types: Typically “Accounts in this organizational directory only”.
    Redirect URI: leave blank (not needed for GitHub Actions).
    Click Register.
  2.2 Add a Federated Credential for GitHub (SE AGREGA UNO POR AMBIENTE dev/prod)
    A Federated Credential is an identity link that allows GitHub Actions (from a specific repo/branch) to authenticate against Microsoft Entra ID without a static secret. It relies on short-lived OpenID Connect tokens, enhancing security by avoiding permanent credentials.
    
    In your newly created App Registration, go to Certificates & secrets.
    Under Federated credentials, click Add credential.
    Select GitHub as the identity provider.
    Provide:
    Organization: Your GitHub org or username.
    Repository: e.g., YourUsername/dotnet9-sample.
    Branch: e.g., main.
    Identifier: e.g., “All workflows” (or your chosen identifier).
    Click Add.
    This step allows GitHub Actions for that repo + branch to request a short-lived OIDC token from Microsoft Entra ID.
    
  2.3 Assign a Role to the Service Principal
    By default, a newly registered app (service principal) has no access to your Azure subscription. You need to grant at least Contributor role:
    
    Go to Subscriptions in the portal’s left menu.
    Select your subscription.
    Click Access control (IAM) → + Add → Add role assignment.
    Role: Choose Contributor (or Owner, if you need full control).
    Assign access to: “User, group, or service principal.”
    Select your app name (e.g., MyDotNet9AppOIDC), then Review + assign.
    Alternatively, you can do this at the Resource Group level if you only want to grant access to resources in MyDotNet9RG.
    
  3. Prepare GitHub Repository
    3.1 Add Repo Variables/Secrets for Client/Tenant/Subscription IDs
      In your GitHub repository, go to Settings → Secrets and variables → Actions.
      Create a new repository variable (or secret) for:
      AZURE_CLIENT_ID = (Application (client) ID)
      AZURE_TENANT_ID = (Directory (tenant) ID)
      AZURE_SUBSCRIPTION_ID = (the GUID of your subscription)
      Click Add after each.
      (If you use secrets instead of variables, adjust the workflow syntax accordingly.)
      3.2 Create the GitHub Actions Workflow (OIDC)
      Add a file in your repo, e.g.: .github/workflows/deploy-azure-oidc.yml:
