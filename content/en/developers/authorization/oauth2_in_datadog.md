---
title: OAuth2 in Datadog
kind: documentation
description: Learn about how Datadog uses OAuth 2.0.
further_reading:
- link: "/developers/authorization/oauth2_endpoints"
  tag: "Documentation"
  text: "Learn how to use the OAuth 2.0 authorization endpoints"
---

{{< beta-callout btn_hidden="true" >}}
  The Datadog Developer Platform is in beta. If you don't have access, contact apps@datadoghq.com.
{{< /beta-callout >}} 

## Overview

This page provides a step-by-step overview on how to implement the OAuth protocol end-to-end on your application once your **confidential** client is created. 

### Implement the OAuth protocol

1. Create and configure your OAuth client in the [Developer Platform][16]. 

2. After a user installs your integration, they can click the **Connect Accounts** button to connect their account in the **Configure** tab of the integration tile. 

   When a user clicks this button, they are directed to the `onboarding_url` that you provided as a part of the OAuth client creation process. This page should be the sign-in page for your platform.

3. Once a user signs in, redirect them to the [OAuth2 Authorization endpoint][6] with the appropriate URL parameters, which includes the added `code_challenge` parameter generated by your application.

   To learn how to derive the `code_challenge` parameter, see the [PKCE](#authorization-code-grant-flow-with-pkce) section. Your application must save `code_verifier` for the token request in Step 5.

   - In order to build the URL for this post request, use the `site` query parameter that is provided on the redirect to your `onboarding_url`. 
   - This parameter is only provided if the user initiates authorization from the Datadog integration tile. See the [Initiate authorization from a third-party location](#Initiate-authorization-from-a-third-party-location) section for more options if the user chooses to initiate authorization externally.  
   - For example, `site` may be `https://app.datadoghq.com`, `https://app.datadoghq.eu`, `https://us5.datadoghq.com`, or it might have a custom subdomain that must be accounted for, such as `https://<custom_subdomain>.datadoghq.com`. 
   - To handle multiple Datadog sites dynamically, you can append the Datadog `authorize` path to this constructed URL.

4. Once a user clicks **Authorize**, Datadog makes a POST request to the authorize endpoint. The user is redirected to the `redirect_uri` that you provided when setting up the OAuth Client with the authorization `code` parameter in the query component.

5. From the `redirect_uri`, make a POST request to the [Datadog token endpoint][10] that includes the authorization code from Step 4, the `code_verifier` from Step 3, your OAuth client ID, and client secret.

   - In order to build the URL for this post request, use the `site` query parameter that is provided on the redirect to your `redirect_uri`. 
   - For example, `site` might be `https://app.datadoghq.com`, `https://app.datadoghq.eu`, `https://us5.datadoghq.com`, or it might have a custom subdomain that must be accounted for, such as `https://<custom_subdomain>.datadoghq.com`. 
   - To handle multiple Datadog sites dynamically, you can append the Datadog `token` path to this constructed URL.

6. Upon success, you receive your `access_token` and `refresh_token` in the response body. Your application should display a confirmation page with the following message: `You may now close this tab`.

7. Use the `access_token` to make calls to Datadog API endpoints by sending it in as a part of the authorization header of your request: ```headers = {"Authorization": "Bearer {}".format(access_token)}```.
    - When making calls to API endpoints, ensure that the user's Datadog site is taken into account. For example, if a user is in the EU region, the events endpoint is `https://api.datadoghq.eu/api/v1/events`, while for users in US1, the events endpoint is `https://api.datadoghq.com/api/v1/events`. Some endpoints may also require an API key, which is created in Step 8 below. 

8. Call the [API Key Creation endpoint][7] to generate an API key that allows you to send data on behalf of Datadog users.

   If the `API_KEYS_WRITE` scope has not been added to your client, this step fails. This endpoint generates an API Key that is only shown once. **Store this value in a secure database or location**. 

For more information about client creation and publishing, see [OAuth for Datadog Integrations][5].

### Initiate authorization from a third-party location 

You can start the authorization process in Datadog by clicking **Connect Accounts** in the integration tile, or from the integration's external website. For example, if there's an integration configuration page on your website that Datadog users need to use, you can give users the option to start the authorization process from there.

When kicking off authorization from a third-party location—anywhere outside of the Datadog integration tile—you need to account for the [Datadog site][17] (such as EU, US1, US3, or US5) when routing users through the authorization flow and building out the URL for the `authorization` and `token` endpoints. 

To ensure that users are authorizing in the correct site, always direct them to the US1 Datadog site (`app.datadoghq.com`), and from there, they can select their region. Once the authorization flow is complete, ensure that all followup API calls use the correct site that is returned as a query parameter with the `redirect_uri` (See Step 5 in [Implement the OAuth protocol](#Implement-the-oauth-protocol)).

When users initiate authorization from the Datadog integration tile, they are required to have corresponding permissions for all requested scopes. If authorization is initiated from somewhere other than the integration tile, users without all of the required permissions may complete authorization (but are prompted to re-authorize with proper permissions when returning to the integration tile). To avoid this, users should be routed from third-party platforms to the Datadog integration tile to begin authorization. 

## Authorization code grant flow with PKCE

While the OAuth2 protocol supports several grant flows, the [authorization code grant flow][8] [with PKCE](#authorization-code-grant-flow-with-pkce) is the recommended grant type for long-running applications where a user grants explicit consent once and client credentials can be securely stored. 

This grant type allows applications to securely obtain a unique authorization code and exchange it for an access token that enables them to make requests to Datadog APIs. The authorization code grant flow consists of three steps:

1. The application [requests authorization from a user][6] to access a set of Datadog scopes.
2. When a user clicks **Authorize**, the application [obtains a unique authorization code][12].
3. The application [exchanges the authorization code for an access token][10], which is used to access Datadog APIs.

### Use the PKCE extension

[Proof key for code exchange (PKCE)][11] is an extension of the OAuth2 authorization code grant flow to protect OAuth2 clients from interception attacks. If an attacker intercepts the flow and gains access to the authorization code before it is returned to the application, they can obtain access tokens and gain access to Datadog APIs.

To mitigate such attacks, the PKCE extension includes the following parameters to securely correlate an authorization request with a token request throughout the grant flow: 

| Parameter             | Definition                                                                                                                           |
|-----------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Code Verifier         | A dynamically-generated cryptographic random string.                                                                                 |
| Code Challenge        | A transformation of the code verifier.                                                                                               |
| Code Challenge Method | The method used to derive the `code_challenge` from the `code_verifier`. You must use [SHA-256][16] to compute the `code_challenge`. |

The [PKCE protocol][11] integrates with the authorization code grant flow by completing the following actions:

- The application generates a `code_verifier` random string and derives the corresponding `code_challenge` using the `code_challenge_method`.

- The application sends an authorization request to Datadog with the `code_challenge` and `code_challenge_method` parameters to obtain an authorization code.

- The application sends a token request to Datadog with the authorization code and `code_verifier` to obtain an access token. The token endpoint verifies the authorization code by transforming the `code_verifier` using the `code_challenge_method` and comparing it with the original `code_challenge` value.

## Further Reading

{{< partial name="whats-next/whats-next.html" >}}

[1]: https://datatracker.ietf.org/doc/html/rfc6749
[2]: /api/latest/scopes/
[3]: /developers/datadog_apps/#oauth-api-access
[4]: https://datatracker.ietf.org/doc/html/rfc6749#section-3.2.1
[5]: /developers/integrations/oauth_for_integrations
[6]: /developers/authorization/oauth2_endpoints/?tab=authorizationendpoints#request-authorization-from-a-user
[7]: /developers/authorization/oauth2_endpoints/?tab=apikeycreationendpoints#create-an-api-key-on-behalf-of-a-user
[8]: https://tools.ietf.org/html/rfc6749#section-4.1
[9]: /developers/authorization/oauth2_endpoints/?tab=authorizationendpoints#obtain-an-authorization-code
[10]: /developers/authorization/oauth2_endpoints/?tab=tokenendpoints#exchange-authorization-code-for-access-token
[11]: https://datatracker.ietf.org/doc/html/rfc7636
[12]: https://datatracker.ietf.org/doc/html/rfc7636#section-4.1
[13]: https://datatracker.ietf.org/doc/html/rfc7636#section-4.2
[14]: https://datatracker.ietf.org/doc/html/rfc7636#section-4.3
[15]: https://datatracker.ietf.org/doc/html/rfc6234#section-4.1
[16]: https://app.datadoghq.com/apps
[17]: /getting_started/site/
