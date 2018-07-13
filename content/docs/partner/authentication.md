+++
date = "2016-02-14T16:11:58+05:30"
title = "Authentication"
slug = "authentication"
+++

Strava uses [OAuth2](http://oauthbible.com/#oauth-2-three-legged) for authentication to our V3 API. It allows external applications to request authorization to a user’s private data without requiring their Strava username and password. It allows users to grant and revoke API access on a per-application basis and keeps users’ authentication details safe.

All developers need to register their application before getting started. A registered application will be assigned a Client id and Client secret. The secret is used for authentication and should never be shared.

## OAuth Overview

In this flow, the user is prompted by the application to log in on the Strava website and to give consent to the requesting application. A user can opt out of the scopes requested by the application.

If the user authorizes the application, Strava redirects the user to a URL specified by the application. This URL will include an authorization code and the scope accepted by the athlete. The application must complete the authentication process by exchanging the authorization code for a refresh token and short-lived access token.

Access tokens are used by applications to obtain and modify Strava resources on behalf of the authenticated athlete. Refresh tokens are used to obtain new access tokens when older ones expire.

Users can revoke all access tokens and refresh tokens for a specific application on their [apps settings](https://www.strava.com/settings/apps) page. Per the [Strava API Agreement](https://www.strava.com/legal/api), application owners must immediately delete all data about users who revoke access to their applications. Application owners can integrate against the [Strava Webhook Events API](../../webhooks/) to receive events when an athlete revokes access to their application.

Note: Google Sign-in will not work for apps that use a mobile webview; see [Google’s blog post](https://developers.googleblog.com/2016/08/modernizing-oauth-interactions-in-native-apps.html) for further information and ways to work around that limitation.

Applications can be managed from the owner's [API settings page](https://www.strava.com/settings/api).

## Request access

To initiate the flow, applications must redirect the user to Strava’s authorization page, `GET https://www.strava.com/oauth/authorize`. The page will prompt the user to consent access of their data to the application. Scopes requested by the application are shown as checked boxes, but the user may opt out of any requested scopes. If an application relies on specific scopes to function properly, the application should make that clear before and after authentication.

<table class="parameters">
  <tr>
    <td width="200px">
        <span class="parameter-name">client_id</span>
      <br>
      <span class="parameter-description">
        required integer, in query
      </span>
    </td>
    <td>
        The application’s ID.
    </td>
  </tr>
  <tr>
    <td width="200px">
      <span class="parameter-name">redirect_uri</span>
      <br>
      <span class="parameter-description">
        required string, in query
      </span>
    </td>
    <td>
        URL to which the user will be redirected after authentication. Must be within the callback domain specified by the application. `localhost` and `127.0.0.1` are white-listed.
    </td>
  </tr>
  <tr>
    <td width="200px">
      <span class="parameter-name">response_type</span>
      <br>
      <span class="parameter-description">
        required string, in query
      </span>
    </td>
    <td>
        Must be `code`.
    </td>
  </tr>
  <tr>
    <td width="200px">
      <span class="parameter-name">approval_prompt</span>
      <br>
      <span class="parameter-description">
        string, in query
      </span>
    </td>
    <td>
        `force` or `auto`, use `force` to always show the authorization prompt even if the user has already authorized the current application, default is `auto`.
    </td>
  </tr>
  <tr>
    <td width="200px">
      <span class="parameter-name">scope</span>
      <br>
      <span class="parameter-description">
        string, in query
      </span>
    </td>
    <td>
      <p>
        Requested scopes, as a comma delimited string, e.g. "read,write,view_private". By default, applications only have access to the athlete profile and generic Strava data. Applications should request only the scopes required for the application to function normally.
      </p>
      <ul>
        <li>"read": read athlete data, such as activities, KOMs, clubs, etc</li>
        <li>"write": modify entities and upload activities</li>
        <li>"view_private": view private activities and data within privacy zones</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td width="200px">
      <span class="parameter-name">state</span>
      <br>
      <span class="parameter-description">
        string, in query
      </span>
    </td>
    <td>
        Returned in the redirect URI. Useful if the authentication is done from various points in an app.
    </td>
  </tr>
</table>

## Token exchange

Strava will respond to the authorization request by redirecting the user agent to the `redirect_uri` provided.

If access is denied, `error=access_denied` will be included in the query string.

If access is accepted, `code` and `scope` parameters will be included in the query string. The `code` parameter contains the authorization code required to complete the authentication process. The application must now call the `POST https://www.strava.com/oauth/token` with its client ID and client secret to exchange the authorization code for a refresh token and short-lived access token.

The `state` parameter will be always included in the response if it was initially provided by the application.

<table class="parameters">
  <tr>
    <td width="200px">
        <span class="parameter-name">client_id</span>
      <br>
      <span class="parameter-description">
        required integer, in query
      </span>
    </td>
    <td>
        The application’s ID, obtained during registration.
    </td>
  </tr>
  <tr>
    <td width="200px">
      <span class="parameter-name">client_secret</span>
      <br>
      <span class="parameter-description">
        required string, in query
      </span>
    </td>
    <td>
        The application’s secret, obtained during registration.
    </td>
  </tr>
  <tr>
    <td width="200px">
      <span class="parameter-name">code</span>
      <br>
      <span class="parameter-description">
        required string, in query
      </span>
    </td>
    <td>
        The `code` parameter obtained in the redirect.
    </td>
  </tr>
  <tr>
    <td width="200px">
      <span class="parameter-name">grant_type</span>
      <br>
      <span class="parameter-description">
        required string, in query
      </span>
    </td>
    <td>
        The grant type for the request. For initial authentication, must always be "authorization_code".
    </td>
  </tr>
</table>

A refresh token, access token, and access token expiration date will be issued upon successful authentication. The `expires_at` field contains the number of seconds since the Epoch when the provided access token will expire.

###### Example Response

    {
      "token_type": "Bearer",
      "access_token": "987654321234567898765432123456789",
      "athlete": {
        #{summary athlete representation}
      },
      "refresh_token": "1234567898765432112345678987654321",
      "expires_at": 1531378346,
      "state": "STRAVA"
    }


## Refresh expired access tokens

Access tokens expire six hours after they are created, so they must be refreshed in order for an application to maintain access to a user's resources. Applications use the refresh token from initial authentication to obtain new access tokens.

To refresh an access token, applications should call the `POST https://www.strava.com/oauth/token` endpoint, specifying `grant_type: refresh_token` and including the application's refresh token for the user as an additional parameter. If the application has an access token for the user that expires in more than one hour, the existing access token will be returned. If the application's access tokens for the user are expired or will expire in one hour (3,600 seconds) or less, a new access token will be returned. In this case, both the newer and older access tokens can be used until they expire.

A refresh token is issued back to the application after all successful requests to the `POST https://www.strava.com/oauth/token` enndpoint. The refresh token may or may not be the same refresh token used to make the request. Applications should persist the refresh token contained in the response, and always use the most recent refresh token for subsequent requests to obtain a new access token. Once a new refresh token is returned, the older refresh token is invalidated immediately.

<table class="parameters">
  <tr>
    <td width="200px">
        <span class="parameter-name">client_id</span>
      <br>
      <span class="parameter-description">
        required integer, in query
      </span>
    </td>
    <td>
        The application’s ID, obtained during registration.
    </td>
  </tr>
  <tr>
    <td width="200px">
      <span class="parameter-name">client_secret</span>
      <br>
      <span class="parameter-description">
        required string, in query
      </span>
    </td>
    <td>
        The application’s secret, obtained during registration.
    </td>
  </tr>
  <tr>
    <td width="200px">
      <span class="parameter-name">grant_type</span>
      <br>
      <span class="parameter-description">
        required string, in query
      </span>
    </td>
    <td>
        The grant type for the request. For initial authentication, must always be "authorization_code".
    </td>
  </tr>
  <tr>
    <td width="200px">
      <span class="parameter-name">refresh_token</span>
      <br>
      <span class="parameter-description">
        required string, in query
      </span>
    </td>
    <td>
        The refresh token obtained during initial authentication.
    </td>
  </tr>
</table>

###### Example Response

    {
      "token_type": "Bearer",
      "access_token": "987654321234567898765432123456789",
      "refresh_token": "1234567898765432112345678987654321",
      "expires_at": 1531385304
    }

## Access the API using an Access Token

Applications use unexpired access tokens to make resource requests to the Strava API on the user's behalf. Access tokens are required for all resource requests, and can be included by specifying the `Authorization: Bearer #{access_token}` header. For instance, using [HTTPie](https://httpie.org/):

```
$ http https://www.strava.com/api/v3/athlete 'Authorization: Bearer 83ebeabdec09f6670863766f792ead24d61fe3f9'
```

## Deauthorization

Applications can revoke access to an athlete’s data. This will invalidate all access tokens that the application has for the athlete, and the application will be removed from the athlete's [apps settings page](https://www.strava.com/settings/apps). All requests made using invalidated tokens will receive a 401 Unauthorized response.

The endpoint is `POST https://www.strava.com/oauth/deauthorize`.

<table class="parameters">
  <tr>
    <td width="200px">
        <span class="parameter-name">access_token</span>
      <br>
      <span class="parameter-description">
        required string, in query
      </span>
    </td>
    <td>
        Responds with the access token submitted with the request.
    </td>
  </tr>
</table>
