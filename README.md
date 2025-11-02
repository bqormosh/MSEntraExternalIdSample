# ASP.NET Core MVC with Microsoft Entra External ID (CIAM)

This sample shows how to sign in users to an ASP.NET Core MVC app using **Microsoft Entra External ID for customers** (the “new B2C”). It assumes you start with a **personal** Microsoft account and no existing B2C tenant.

## 1. Prerequisites

- Visual Studio 2022 (latest) with **ASP.NET and web development** workload
- .NET 8 SDK
- An Azure subscription (personal / pay-as-you-go is fine)
- A browser that can reach `https://localhost:5001`

## 2. Create the External ID (CIAM) tenant

1. Go to **https://entra.microsoft.com** and sign in.
2. Switch to **Directories + subscriptions** → **Create tenant**  
   (or in Azure portal: **Create a resource** → **Entra External ID**).
3. Choose **External** type.
4. Name it, e.g. `myOrg`.
5. Initial domain, e.g. `myb2cpersonal.onmicrosoft.com`.

   This gives you the CIAM host:

   ```text
   https://myb2cpersonal.ciamlogin.com
   ```

6. Finish creation, then **switch** to this new tenant.

## 3. Create a user flow

1. In the external tenant go to **External Identities → User flows**.
2. **New user flow**.
3. Choose **Sign up and sign in**.
4. Name it, for example:

   ```text
   TestUserFlow
   ```

5. Enable Email sign-up.
6. Create.

> Later, make sure your app is added to this user flow:  
> **User flow → Applications → Add → choose your web app.**

## 4. Register the web app

1. In the same tenant: **App registrations → New registration**.
2. Name: `ExternalIDSample`.
3. Supported account types: **Accounts in this organizational directory only**.
4. Redirect URI (web): `https://localhost:5001/signin-oidc`
5. Click **Register**.
6. Go to **Authentication**:
   - Add **Front-channel logout URL**: `https://localhost:5001/signout-callback-oidc`
   - Tick **ID tokens**.
7. Go to **Certificates & secrets** → **New client secret** → copy the **Value**.
8. Note these values:
   - Application (client) ID → used as `ClientId`
   - Secret value → used as `ClientSecret`
   - Tenant domain → `myb2cpersonal.onmicrosoft.com`
   - CIAM host → `https://myb2cpersonal.ciamlogin.com`

## 5. Configure the app

Create or edit `appsettings.json`:

```json
{
  "AzureAd": {
    "Authority": "https://myb2cpersonal.ciamlogin.com",
    "ClientId": "YOUR-CLIENT-ID",
    "ClientSecret": "YOUR-CLIENT-SECRET",
    "CallbackPath": "/signin-oidc"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

Replace:

- `YOUR-CLIENT-ID` → from App registration → Overview
- `YOUR-CLIENT-SECRET` → from App registration → Certificates & secrets

> Why not `Instance` + `Domain` + `SignUpSignInPolicyId` like classic B2C?  
> Because Entra External ID publishes metadata under `https://<tenant>.ciamlogin.com/...` and the B2C-style settings produce URLs that 404 in CIAM.

## 6. Program.cs

```csharp
using Microsoft.AspNetCore.Authentication.OpenIdConnect;
using Microsoft.Identity.Web;
using Microsoft.Identity.Web.UI;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(builder.Configuration.GetSection("AzureAd"));

builder.Services.AddControllersWithViews()
    .AddMicrosoftIdentityUI();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

## 7. Add a secured page

**Controllers/HomeController.cs**

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace ExternalIDSample.Controllers
{
    public class HomeController : Controller
    {
        public IActionResult Index()
        {
            return View();
        }

        [Authorize]
        public IActionResult Secure()
        {
            return View();
        }
    }
}
```

**Views/Home/Secure.cshtml**

```html
@{
    ViewData["Title"] = "Secure";
}
<h1>Secure</h1>
<p>@User.Identity?.Name</p>
```

## 8. Add sign in / sign out UI

Create `Views/Shared/_LoginPartial.cshtml`:

```html
@using Microsoft.AspNetCore.Mvc.TagHelpers
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers

@if (User.Identity?.IsAuthenticated ?? false)
{
    <form asp-area="MicrosoftIdentity" asp-controller="Account" asp-action="SignOut" method="post" class="d-flex">
        <button type="submit" class="btn btn-link nav-link p-0">Sign out (@User.Identity.Name)</button>
    </form>
}
else
{
    <a asp-area="MicrosoftIdentity" asp-controller="Account" asp-action="SignIn" class="nav-link">Sign in</a>
}
```

In `Views/Shared/_Layout.cshtml`, inside the navbar:

```html
<ul class="navbar-nav ms-auto">
    <partial name="_LoginPartial" />
</ul>
```

## 9. Run the app

1. Run the app from Visual Studio (IIS Express or Kestrel).
2. Open `https://localhost:5001/`.
3. Click **Sign in**.
4. You should be redirected to `https://myb2cpersonal.ciamlogin.com/...`.
5. Complete sign-up/sign-in.
6. Navigate to `/Home/Secure` → you should see the protected view.

## 10. Common issues

- **404 on `/.well-known/openid-configuration`**  
  You’re probably using B2C-style config. Use only `Authority: https://<tenant>.ciamlogin.com`.

- **Your app is not configured for this user flow**  
  In **External Identities → User flows → TestUserFlow → Applications**, add your app.

- **Wrong redirect URI**  
  Make sure the URI in the app registration is exactly `https://localhost:5001/signin-oidc` and matches `CallbackPath`.

## 11. GitHub

Do **not** commit your real `appsettings.json`. Add this to `.gitignore`:

```text
appsettings.json
appsettings.Development.json
```

Add a sample file:

```json
{
  "AzureAd": {
    "Authority": "https://YOUR-TENANT.ciamlogin.com",
    "ClientId": "YOUR-CLIENT-ID",
    "ClientSecret": "YOUR-CLIENT-SECRET",
    "CallbackPath": "/signin-oidc"
  }
}
```
