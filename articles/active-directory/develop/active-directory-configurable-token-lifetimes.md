---
title: Configurable token lifetimes in Azure Active Directory | Microsoft Docs
description: Learn how to set lifetimes for tokens issued by Azure AD.
services: active-directory
documentationcenter: ''
author: CelesteDG
manager: mtillman
editor: ''

ms.assetid: 06f5b317-053e-44c3-aaaa-cf07d8692735
ms.service: active-directory
ms.workload: identity
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 10/05/2018
ms.author: celested
ms.custom: aaddev
ms.reviewer: hirsin

---
# Configurable token lifetimes in Azure Active Directory (Preview)

You can specify the lifetime of a token issued by Azure Active Directory (Azure AD). You can set token lifetimes for all apps in your organization, for a multi-tenant (multi-organization) application, or for a specific service principal in your organization.

> [!IMPORTANT]
> After hearing from customers during the preview, we're planning to replace this functionality with a new feature in Azure Active Directory Conditional Access.  Once the new feature is complete, this functionality will eventually be deprecated after a notification period. If you use the Configurable Token Lifetime policy, be prepared to switch to the new Conditional Access feature once it's available.

In Azure AD, a policy object represents a set of rules that are enforced on individual applications or on all applications in an organization. Each policy type has a unique structure, with a set of properties that are applied to objects to which they are assigned.

You can designate a policy as the default policy for your organization. The policy is applied to any application in the organization, as long as it is not overridden by a policy with a higher priority. You also can assign a policy to specific applications. The order of priority varies by policy type.

> [!NOTE]
> Configurable token lifetime policy is not supported for SharePoint Online. Even though you have the ability to create this policy via PowerShell, SharePoint Online will not acknowledge this policy. Refer to the [SharePoint Online blog](https://techcommunity.microsoft.com/t5/SharePoint-Blog/Introducing-Idle-Session-Timeout-in-SharePoint-and-OneDrive/ba-p/119208) to learn more about configuring idle session timeouts.
>* The default lifetime for the SharePoint Online access token is 1 hour.
>* The default max inactive time of the SharePoint Online refresh token is 90 days.

## Token types

You can set token lifetime policies for refresh tokens, access tokens, session tokens, and ID tokens.

### Access tokens

Clients use access tokens to access a protected resource. An access token can be used only for a specific combination of user, client, and resource. Access tokens cannot be revoked and are valid until their expiry. A malicious actor that has obtained an access token can use it for extent of its lifetime. Adjusting the lifetime of an access token is a trade-off between improving system performance and increasing the amount of time that the client retains access after the user’s account is disabled. Improved system performance is achieved by reducing the number of times a client needs to acquire a fresh access token. The default is 1 hour - after 1 hour, the client must use the refresh token to (usually silently) acquire a new refresh token and access token.

### Refresh tokens
When a client acquires an access token to access a protected resource, the client also receives a refresh token. The refresh token is used to obtain new access/refresh token pairs when the current access token expires. A refresh token is bound to a combination of user and client. A refresh token can be [revoked at any time](access-tokens.md#token-revocation), and the token's validity is checked every time the token is used.

It's important to make a distinction between confidential clients and public clients, as this impacts how long refresh tokens can be used. For more information about different types of clients, see [RFC 6749](https://tools.ietf.org/html/rfc6749#section-2.1).

#### Token lifetimes with confidential client refresh tokens
Confidential clients are applications that can securely store a client password (secret). They can prove that requests are coming from the secured client application and not from a malicious actor. For example, a web app is a confidential client because it can store a client secret on the web server. It is not exposed. Because these flows are more secure, the default lifetimes of refresh tokens issued to these flows is `until-revoked`, cannot be changed by using policy, and will not be revoked on voluntary password resets.

#### Token lifetimes with public client refresh tokens

Public clients cannot securely store a client password (secret). For example, an iOS/Android app cannot obfuscate a secret from the resource owner, so it is considered a public client. You can set policies on resources to prevent refresh tokens from public clients older than a specified period from obtaining a new access/refresh token pair. (To do this, use the Refresh Token Max Inactive Time property (`MaxInactiveTime`).) You also can use policies to set a period beyond which the refresh tokens are no longer accepted. (To do this, use the Refresh Token Max Age property.) You can adjust the lifetime of a refresh token to control when and how often the user is required to reenter credentials, instead of being silently reauthenticated, when using a public client application.

### ID tokens
ID tokens are passed to websites and native clients. ID tokens contain profile information about a user. An ID token is bound to a specific combination of user and client. ID tokens are considered valid until their expiry. Usually, a web application matches a user’s session lifetime in the application to the lifetime of the ID token issued for the user. You can adjust the lifetime of an ID token to control how often the web application expires the application session, and how often it requires the user to be reauthenticated with Azure AD (either silently or interactively).

### Single sign-on session tokens
When a user authenticates with Azure AD, a single sign-on session (SSO) is established with the user’s browser and Azure AD. The SSO token, in the form of a cookie, represents this session. Note that the SSO session token is not bound to a specific resource/client application. SSO session tokens can be revoked, and their validity is checked every time they are used.

Azure AD uses two kinds of SSO session tokens: persistent and nonpersistent. Persistent session tokens are stored as persistent cookies by the browser. Nonpersistent session tokens are stored as session cookies. (Session cookies are destroyed when the browser is closed.)
Usually, a nonpersistent session token is stored. But, when the user selects the **Keep me signed in** check box during authentication, a persistent session token is stored.

Nonpersistent session tokens have a lifetime of 24 hours. Persistent tokens have a lifetime of 180 days. Any time an SSO session token is used within its validity period, the validity period is extended another 24 hours or 180 days, depending on the token type. If an SSO session token is not used within its validity period, it is considered expired and is no longer accepted.

You can use a policy to set the time after the first session token was issued beyond which the session token is no longer accepted. (To do this, use the Session Token Max Age property.) You can adjust the lifetime of a session token to control when and how often a user is required to reenter credentials, instead of being silently authenticated, when using a web application.

### Token lifetime policy properties
A token lifetime policy is a type of policy object that contains token lifetime rules. Use the properties of the policy to control specified token lifetimes. If no policy is set, the system enforces the default lifetime value.

### Configurable token lifetime properties
| Property | Policy property string | Affects | Default | Minimum | Maximum |
| --- | --- | --- | --- | --- | --- |
| Access Token Lifetime |AccessTokenLifetime |Access tokens, ID tokens, SAML2 tokens |1 hour |10 minutes |1 day |
| Refresh Token Max Inactive Time |MaxInactiveTime |Refresh tokens |90 days |10 minutes |90 days |
| Single-Factor Refresh Token Max Age |MaxAgeSingleFactor |Refresh tokens (for any users) |Until-revoked |10 minutes |Until-revoked<sup>1</sup> |
| Multi-Factor Refresh Token Max Age |MaxAgeMultiFactor |Refresh tokens (for any users) |Until-revoked |10 minutes |Until-revoked<sup>1</sup> |
| Single-Factor Session Token Max Age |MaxAgeSessionSingleFactor<sup>2</sup> |Session tokens (persistent and nonpersistent) |Until-revoked |10 minutes |Until-revoked<sup>1</sup> |
| Multi-Factor Session Token Max Age |MaxAgeSessionMultiFactor<sup>3</sup> |Session tokens (persistent and nonpersistent) |Until-revoked |10 minutes |Until-revoked<sup>1</sup> |

* <sup>1</sup>365 days is the maximum explicit length that can be set for these attributes.
* <sup>2</sup>If **MaxAgeSessionSingleFactor** is not set, this value takes the **MaxAgeSingleFactor** value. If neither parameter is set, the property takes the default value (until-revoked).
* <sup>3</sup>If **MaxAgeSessionMultiFactor** is not set, this value takes the **MaxAgeMultiFactor** value. If neither parameter is set, the property takes the default value (until-revoked).

### Exceptions
| Property | Affects | Default |
| --- | --- | --- |
| Refresh Token Max Age (issued for federated users who have insufficient revocation information<sup>1</sup>) |Refresh tokens (issued for federated users who have insufficient revocation information<sup>1</sup>) |12 hours |
| Refresh Token Max Inactive Time (issued for confidential clients) |Refresh tokens (issued for confidential clients) |90 days |
| Refresh Token Max Age (issued for confidential clients) |Refresh tokens (issued for confidential clients) |Until-revoked |

* <sup>1</sup>Federated users who have insufficient revocation information include any users who do not have the "LastPasswordChangeTimestamp" attribute synced. These users are given this short Max Age because AAD is unable to verify when to revoke tokens that are tied to an old credential (such as a password that has been changed) and must check back in more frequently to ensure that the user and associated tokens are still in good standing. To improve this experience, tenant admins must ensure that they are syncing the “LastPasswordChangeTimestamp” attribute (this can be set on the user object using Powershell or through AADSync).

### Policy evaluation and prioritization
You can create and then assign a token lifetime policy to a specific application, to your organization, and to service principals. Multiple policies might apply to a specific application. The token lifetime policy that takes effect follows these rules:

* If a policy is explicitly assigned to the service principal, it is enforced.
* If no policy is explicitly assigned to the service principal, a policy explicitly assigned to the parent organization of the service principal is enforced.
* If no policy is explicitly assigned to the service principal or to the organization, the policy assigned to the application is enforced.
* If no policy has been assigned to the service principal, the organization, or the application object, the default values is enforced. (See the table in [Configurable token lifetime properties](#configurable-token-lifetime-properties).)

For more information about the relationship between application objects and service principal objects, see [Application and service principal objects in Azure Active Directory](app-objects-and-service-principals.md).

A token’s validity is evaluated at the time the token is used. The policy with the highest priority on the application that is being accessed takes effect.

All timespans used here are formatted according to the C# [TimeSpan](https://msdn.microsoft.com/library/system.timespan) object - D.HH:MM:SS. So 80 days and 30 minutes would be `80.00:30:00`. The leading D can be dropped if zero, so 90 minutes would be `00:90:00`.

> [!NOTE]
> Here's an example scenario.
>
> A user wants to access two web applications: Web Application A and Web Application B.
>
> Factors:
> * Both web applications are in the same parent organization.
> * Token Lifetime Policy 1 with a Session Token Max Age of eight hours is set as the parent organization’s default.
> * Web Application A is a regular-use web application and isn’t linked to any policies.
> * Web Application B is used for highly sensitive processes. Its service principal is linked to Token Lifetime Policy 2, which has a Session Token Max Age of 30 minutes.
>
> At 12:00 PM, the user starts a new browser session and tries to access Web Application A. The user is redirected to Azure AD and is asked to sign in. This creates a cookie that has a session token in the browser. The user is redirected back to Web Application A with an ID token that allows the user to access the application.
>
> At 12:15 PM, the user tries to access Web Application B. The browser redirects to Azure AD, which detects the session cookie. Web Application B’s service principal is linked to Token Lifetime Policy 2, but it's also part of the parent organization, with default Token Lifetime Policy 1. Token Lifetime Policy 2 takes effect because policies linked to service principals have a higher priority than organization default policies. The session token was originally issued within the last 30 minutes, so it is considered valid. The user is redirected back to Web Application B with an ID token that grants them access.
>
> At 1:00 PM, the user tries to access Web Application A. The user is redirected to Azure AD. Web Application A is not linked to any policies, but because it is in an organization with default Token Lifetime Policy 1, that policy takes effect. The session cookie that was originally issued within the last eight hours is detected. The user is silently redirected back to Web Application A with a new ID token. The user is not required to authenticate.
>
> Immediately afterward, the user tries to access Web Application B. The user is redirected to Azure AD. As before, Token Lifetime Policy 2 takes effect. Because the token was issued more than 30 minutes ago, the user is prompted to reenter their sign-in credentials. A brand-new session token and ID token are issued. The user can then access Web Application B.
>
>

## Configurable policy property details
### Access Token Lifetime
**String:** AccessTokenLifetime

**Affects:** Access tokens, ID tokens

**Summary:** This policy controls how long access and ID tokens for this resource are considered valid. Reducing the Access Token Lifetime property mitigates the risk of an access token or ID token being used by a malicious actor for an extended period of time. (These tokens cannot be revoked.) The trade-off is that performance is adversely affected, because the tokens have to be replaced more often.

### Refresh Token Max Inactive Time
**String:** MaxInactiveTime

**Affects:** Refresh tokens

**Summary:** This policy controls how old a refresh token can be before a client can no longer use it to retrieve a new access/refresh token pair when attempting to access this resource. Because a new refresh token usually is returned when a refresh token is used, this policy prevents access if the client tries to access any resource by using the current refresh token during the specified period of time.

This policy forces users who have not been active on their client to reauthenticate to retrieve a new refresh token.

The Refresh Token Max Inactive Time property must be set to a lower value than the Single-Factor Token Max Age and the Multi-Factor Refresh Token Max Age properties.

### Single-Factor Refresh Token Max Age
**String:** MaxAgeSingleFactor

**Affects:** Refresh tokens

**Summary:** This policy controls how long a user can use a refresh token to get a new access/refresh token pair after they last authenticated successfully by using only a single factor. After a user authenticates and receives a new refresh token, the user can use the refresh token flow for the specified period of time. (This is true as long as the current refresh token is not revoked, and it is not left unused for longer than the inactive time.) At that point, the user is forced to reauthenticate to receive a new refresh token.

Reducing the max age forces users to authenticate more often. Because single-factor authentication is considered less secure than multi-factor authentication, we recommend that you set this property to a value that is equal to or lesser than the Multi-Factor Refresh Token Max Age property.

### Multi-Factor Refresh Token Max Age
**String:** MaxAgeMultiFactor

**Affects:** Refresh tokens

**Summary:** This policy controls how long a user can use a refresh token to get a new access/refresh token pair after they last authenticated successfully by using multiple factors. After a user authenticates and receives a new refresh token, the user can use the refresh token flow for the specified period of time. (This is true as long as the current refresh token is not revoked, and it is not unused for longer than the inactive time.) At that point, users are forced to reauthenticate to receive a new refresh token.

Reducing the max age forces users to authenticate more often. Because single-factor authentication is considered less secure than multi-factor authentication, we recommend that you set this property to a value that is equal to or greater than the Single-Factor Refresh Token Max Age property.

### Single-Factor Session Token Max Age
**String:** MaxAgeSessionSingleFactor

**Affects:** Session tokens (persistent and nonpersistent)

**Summary:** This policy controls how long a user can use a session token to get a new ID and session token after they last authenticated successfully by using only a single factor. After a user authenticates and receives a new session token, the user can use the session token flow for the specified period of time. (This is true as long as the current session token is not revoked and has not expired.) After the specified period of time, the user is forced to reauthenticate to receive a new session token.

Reducing the max age forces users to authenticate more often. Because single-factor authentication is considered less secure than multi-factor authentication, we recommend that you set this property to a value that is equal to or less than the Multi-Factor Session Token Max Age property.

### Multi-Factor Session Token Max Age
**String:** MaxAgeSessionMultiFactor

**Affects:** Session tokens (persistent and nonpersistent)

**Summary:** This policy controls how long a user can use a session token to get a new ID and session token after the last time they authenticated successfully by using multiple factors. After a user authenticates and receives a new session token, the user can use the session token flow for the specified period of time. (This is true as long as the current session token is not revoked and has not expired.) After the specified period of time, the user is forced to reauthenticate to receive a new session token.

Reducing the max age forces users to authenticate more often. Because single-factor authentication is considered less secure than multi-factor authentication, we recommend that you set this property to a value that is equal to or greater than the Single-Factor Session Token Max Age property.

## Example token lifetime policies
Many scenarios are possible in Azure AD when you can create and manage token lifetimes for apps, service principals, and your overall organization. In this section, we walk through a few common policy scenarios that can help you impose new rules for:

* Token Lifetime
* Token Max Inactive Time
* Token Max Age

In the examples, you can learn how to:

* Manage an organization's default policy
* Create a policy for web sign-in
* Create a policy for a native app that calls a web API
* Manage an advanced policy

### Prerequisites
In the following examples, you create, update, link, and delete policies for apps, service principals, and your overall organization. If you are new to Azure AD, we recommend that you learn about [how to get an Azure AD tenant](quickstart-create-new-tenant.md) before you proceed with these examples.

To get started, do the following steps:

1. Download the latest [Azure AD PowerShell Module Public Preview release](https://www.powershellgallery.com/packages/AzureADPreview).
2. Run the `Connect` command to sign in to your Azure AD admin account. Run this command each time you start a new session.

    ```PowerShell
    Connect-AzureAD -Confirm
    ```

3. To see all policies that have been created in your organization, run the following command. Run this command after most operations in the following scenarios. Running the command also helps you get the **
** of your policies.

    ```PowerShell
    Get-AzureADPolicy
    ```

### Example: Manage an organization's default policy
In this example, you create a policy that lets your users sign in less frequently across your entire organization. To do this, create a token lifetime policy for Single-Factor Refresh Tokens, which is applied across your organization. The policy is applied to every application in your organization, and to each service principal that doesn’t already have a policy set.

1. Create a token lifetime policy.

    1. Set the Single-Factor Refresh Token to "until-revoked." The token doesn't expire until access is revoked. Create the following policy definition:

        ```PowerShell
        @('{
            "TokenLifetimePolicy":
            {
                "Version":1,
                "MaxAgeSingleFactor":"until-revoked"
            }
        }')
        ```

    2. To create the policy, run the following command:

        ```PowerShell
        New-AzureADPolicy -Definition @('{"TokenLifetimePolicy":{"Version":1, "MaxAgeSingleFactor":"until-revoked"}}') -DisplayName "OrganizationDefaultPolicyScenario" -IsOrganizationDefault $true -Type "TokenLifetimePolicy"
        ```

    3. To see your new policy, and to get the policy's **ObjectId**, run the following command:

        ```PowerShell
        Get-AzureADPolicy
        ```

2. Update the policy.

    You might decide that the first policy you set in this example is not as strict as your service requires. To set your Single-Factor Refresh Token to expire in two days, run the following command:

    ```PowerShell
    Set-AzureADPolicy -Id <ObjectId FROM GET COMMAND> -DisplayName "OrganizationDefaultPolicyUpdatedScenario" -Definition @('{"TokenLifetimePolicy":{"Version":1,"MaxAgeSingleFactor":"2.00:00:00"}}')
    ```

### Example: Create a policy for web sign-in

In this example, you create a policy that requires users to authenticate more frequently in your web app. This policy sets the lifetime of the access/ID tokens and the max age of a multi-factor session token to the service principal of your web app.

1. Create a token lifetime policy.

    This policy, for web sign-in, sets the access/ID token lifetime and the max single-factor session token age to two hours.

    1. To create the policy, run this command:

        ```PowerShell
        New-AzureADPolicy -Definition @('{"TokenLifetimePolicy":{"Version":1,"AccessTokenLifetime":"02:00:00","MaxAgeSessionSingleFactor":"02:00:00"}}') -DisplayName "WebPolicyScenario" -IsOrganizationDefault $false -Type "TokenLifetimePolicy"
        ```

    2. To see your new policy, and to get the policy **ObjectId**, run the following command:

        ```PowerShell
        Get-AzureADPolicy
        ```

2. Assign the policy to your service principal. You also need to get the **ObjectId** of your service principal.

    1. To see all your organization's service principals, you can query either the [Microsoft Graph](https://developer.microsoft.com/graph/docs/api-reference/beta/resources/serviceprincipal#properties) or the [Azure AD Graph](https://msdn.microsoft.com/Library/Azure/Ad/Graph/api/entity-and-complex-type-reference#serviceprincipal-entity). Also, you can test this in the [Azure AD Graph Explorer](https://graphexplorer.azurewebsites.net/), and the [Microsoft Graph Explorer](https://developer.microsoft.com/graph/graph-explorer) by using your Azure AD account.

    2. When you have the **ObjectId** of your service principal, run the following command:

        ```PowerShell
        Add-AzureADServicePrincipalPolicy -Id <ObjectId of the ServicePrincipal> -RefObjectId <ObjectId of the Policy>
        ```


### Example: Create a policy for a native app that calls a web API
In this example, you create a policy that requires users to authenticate less frequently. The policy also lengthens the amount of time a user can be inactive before the user must reauthenticate. The policy is applied to the web API. When the native app requests the web API as a resource, this policy is applied.

1. Create a token lifetime policy.

    1. To create a strict policy for a web API, run the following command:

        ```PowerShell
        New-AzureADPolicy -Definition @('{"TokenLifetimePolicy":{"Version":1,"MaxInactiveTime":"30.00:00:00","MaxAgeMultiFactor":"until-revoked","MaxAgeSingleFactor":"180.00:00:00"}}') -DisplayName "WebApiDefaultPolicyScenario" -IsOrganizationDefault $false -Type "TokenLifetimePolicy"
        ```

    2. To see your new policy, and to get the policy **ObjectId**, run the following command:

        ```PowerShell
        Get-AzureADPolicy
        ```

2. Assign the policy to your web API. You also need to get the **ObjectId** of your application. The best way to find your app's **ObjectId** is to use the [Azure portal](https://portal.azure.com/).

   When you have the **ObjectId** of your app, run the following command:

    ```PowerShell
    Add-AzureADApplicationPolicy -Id <ObjectId of the Application> -RefObjectId <ObjectId of the Policy>
    ```


### Example: Manage an advanced policy
In this example, you create a few policies, to learn how the priority system works. You also can learn how to manage multiple policies that are applied to several objects.

1. Create a token lifetime policy.

    1. To create an organization default policy that sets the Single-Factor Refresh Token lifetime to 30 days, run the following command:

        ```PowerShell
        New-AzureADPolicy -Definition @('{"TokenLifetimePolicy":{"Version":1,"MaxAgeSingleFactor":"30.00:00:00"}}') -DisplayName "ComplexPolicyScenario" -IsOrganizationDefault $true -Type "TokenLifetimePolicy"
        ```

    2. To see your new policy, and to get the policy's **ObjectId**, run the following command:

        ```PowerShell
        Get-AzureADPolicy
        ```

2. Assign the policy to a service principal.

    Now, you have a policy that applies to the entire organization. You might want to preserve this 30-day policy for a specific service principal, but change the organization default policy to the upper limit of "until-revoked."

    1. To see all your organization's service principals, you can query either the [Microsoft Graph](https://developer.microsoft.com/graph/docs/api-reference/beta/resources/serviceprincipal#properties) or the [Azure AD Graph](https://msdn.microsoft.com/Library/Azure/Ad/Graph/api/entity-and-complex-type-reference#serviceprincipal-entity). Also, you can test this in the [Azure AD Graph Explorer](https://graphexplorer.azurewebsites.net/), and the [Microsoft Graph Explorer](https://developer.microsoft.com/graph/graph-explorer) by using your Azure AD account.

    2. When you have the **ObjectId** of your service principal, run the following command:

        ```PowerShell
        Add-AzureADServicePrincipalPolicy -Id <ObjectId of the ServicePrincipal> -RefObjectId <ObjectId of the Policy>
        ```

3. Set the `IsOrganizationDefault` flag to false:

    ```PowerShell
    Set-AzureADPolicy -Id <ObjectId of Policy> -DisplayName "ComplexPolicyScenario" -IsOrganizationDefault $false
    ```

4. Create a new organization default policy:

    ```PowerShell
    New-AzureADPolicy -Definition @('{"TokenLifetimePolicy":{"Version":1,"MaxAgeSingleFactor":"until-revoked"}}') -DisplayName "ComplexPolicyScenarioTwo" -IsOrganizationDefault $true -Type "TokenLifetimePolicy"
    ```

    You now have the original policy linked to your service principal, and the new policy is set as your organization default policy. It's important to remember that policies applied to service principals have priority over organization default policies.

## Cmdlet reference

### Manage policies

You can use the following cmdlets to manage policies.

#### New-AzureADPolicy

Creates a new policy.

```PowerShell
New-AzureADPolicy -Definition <Array of Rules> -DisplayName <Name of Policy> -IsOrganizationDefault <boolean> -Type <Policy Type>
```

| Parameters | Description | Example |
| --- | --- | --- |
| <code>&#8209;Definition</code> |Array of stringified JSON that contains all the policy's rules. | `-Definition @('{"TokenLifetimePolicy":{"Version":1,"MaxInactiveTime":"20:00:00"}}')` |
| <code>&#8209;DisplayName</code> |String of the policy name. |`-DisplayName "MyTokenPolicy"` |
| <code>&#8209;IsOrganizationDefault</code> |If true, sets the policy as the organization's default policy. If false, does nothing. |`-IsOrganizationDefault $true` |
| <code>&#8209;Type</code> |Type of policy. For token lifetimes, always use "TokenLifetimePolicy." | `-Type "TokenLifetimePolicy"` |
| <code>&#8209;AlternativeIdentifier</code> [Optional] |Sets an alternative ID for the policy. |`-AlternativeIdentifier "myAltId"` |

#### Get-AzureADPolicy
Gets all Azure AD policies or a specified policy.

```PowerShell
Get-AzureADPolicy
```

| Parameters | Description | Example |
| --- | --- | --- |
| <code>&#8209;Id</code> [Optional] |**ObjectId (Id)** of the policy you want. |`-Id <ObjectId of Policy>` |

#### Get-AzureADPolicyAppliedObject
Gets all apps and service principals that are linked to a policy.

```PowerShell
Get-AzureADPolicyAppliedObject -Id <ObjectId of Policy>
```

| Parameters | Description | Example |
| --- | --- | --- |
| <code>&#8209;Id</code> |**ObjectId (Id)** of the policy you want. |`-Id <ObjectId of Policy>` |

#### Set-AzureADPolicy
Updates an existing policy.

```PowerShell
Set-AzureADPolicy -Id <ObjectId of Policy> -DisplayName <string>
```

| Parameters | Description | Example |
| --- | --- | --- |
| <code>&#8209;Id</code> |**ObjectId (Id)** of the policy you want. |`-Id <ObjectId of Policy>` |
| <code>&#8209;DisplayName</code> |String of the policy name. |`-DisplayName "MyTokenPolicy"` |
| <code>&#8209;Definition</code> [Optional] |Array of stringified JSON that contains all the policy's rules. |`-Definition @('{"TokenLifetimePolicy":{"Version":1,"MaxInactiveTime":"20:00:00"}}')` |
| <code>&#8209;IsOrganizationDefault</code> [Optional] |If true, sets the policy as the organization's default policy. If false, does nothing. |`-IsOrganizationDefault $true` |
| <code>&#8209;Type</code> [Optional] |Type of policy. For token lifetimes, always use "TokenLifetimePolicy." |`-Type "TokenLifetimePolicy"` |
| <code>&#8209;AlternativeIdentifier</code> [Optional] |Sets an alternative ID for the policy. |`-AlternativeIdentifier "myAltId"` |

#### Remove-AzureADPolicy
Deletes the specified policy.

```PowerShell
 Remove-AzureADPolicy -Id <ObjectId of Policy>
```

| Parameters | Description | Example |
| --- | --- | --- |
| <code>&#8209;Id</code> |**ObjectId (Id)** of the policy you want. | `-Id <ObjectId of Policy>` |

### Application policies
You can use the following cmdlets for application policies.

#### Add-AzureADApplicationPolicy
Links the specified policy to an application.

```PowerShell
Add-AzureADApplicationPolicy -Id <ObjectId of Application> -RefObjectId <ObjectId of Policy>
```

| Parameters | Description | Example |
| --- | --- | --- |
| <code>&#8209;Id</code> |**ObjectId (Id)** of the application. | `-Id <ObjectId of Application>` |
| <code>&#8209;RefObjectId</code> |**ObjectId** of the policy. | `-RefObjectId <ObjectId of Policy>` |

#### Get-AzureADApplicationPolicy
Gets the policy that is assigned to an application.

```PowerShell
Get-AzureADApplicationPolicy -Id <ObjectId of Application>
```

| Parameters | Description | Example |
| --- | --- | --- |
| <code>&#8209;Id</code> |**ObjectId (Id)** of the application. | `-Id <ObjectId of Application>` |

#### Remove-AzureADApplicationPolicy
Removes a policy from an application.

```PowerShell
Remove-AzureADApplicationPolicy -Id <ObjectId of Application> -PolicyId <ObjectId of Policy>
```

| Parameters | Description | Example |
| --- | --- | --- |
| <code>&#8209;Id</code> |**ObjectId (Id)** of the application. | `-Id <ObjectId of Application>` |
| <code>&#8209;PolicyId</code> |**ObjectId** of the policy. | `-PolicyId <ObjectId of Policy>` |

### Service principal policies
You can use the following cmdlets for service principal policies.

#### Add-AzureADServicePrincipalPolicy
Links the specified policy to a service principal.

```PowerShell
Add-AzureADServicePrincipalPolicy -Id <ObjectId of ServicePrincipal> -RefObjectId <ObjectId of Policy>
```

| Parameters | Description | Example |
| --- | --- | --- |
| <code>&#8209;Id</code> |**ObjectId (Id)** of the application. | `-Id <ObjectId of Application>` |
| <code>&#8209;RefObjectId</code> |**ObjectId** of the policy. | `-RefObjectId <ObjectId of Policy>` |

#### Get-AzureADServicePrincipalPolicy
Gets any policy linked to the specified service principal.

```PowerShell
Get-AzureADServicePrincipalPolicy -Id <ObjectId of ServicePrincipal>
```

| Parameters | Description | Example |
| --- | --- | --- |
| <code>&#8209;Id</code> |**ObjectId (Id)** of the application. | `-Id <ObjectId of Application>` |

#### Remove-AzureADServicePrincipalPolicy
Removes the policy from the specified service principal.

```PowerShell
Remove-AzureADServicePrincipalPolicy -Id <ObjectId of ServicePrincipal>  -PolicyId <ObjectId of Policy>
```

| Parameters | Description | Example |
| --- | --- | --- |
| <code>&#8209;Id</code> |**ObjectId (Id)** of the application. | `-Id <ObjectId of Application>` |
| <code>&#8209;PolicyId</code> |**ObjectId** of the policy. | `-PolicyId <ObjectId of Policy>` |
