# User Account Creation

For public clouds we need to provide a way for users to create accounts and start consuming services.

## Changelog

- v1.0.0 2025-03-03 (@spjmurray): Initial RFC

## OIDC Flow

When a user logs in, we first need to do a full OIDC authorization code flow with a backend IdP in order to acquire an ID token for that user.
At this point we can use the user's email address encoded in the ID token to lookup a global user account.

The global user account is vitally important in the authorization flow as it performs a number of roles:

* Account state i.e. active/suspended at the platform level, this allows platform admins to lock out a bad actor.
* Session management, this enforces a single active access/refresh token per client.
* Access token uniqueness and revocation, only one access token can exist per session and may be revoked as per the oauth2 specification.
* Refresh token uniqueness and single use, only one refresh token can exist per session and can only be used once.
* Active authorization time, used for silent logins so the client can limit the maximum duration, and more generally for monitoring user activity.
* Caching identity information, this allows OIDC token introspection via the `userinfo` endpoint.

Without a global user record, we cannot issue any tokens.

At this point we have two options, for private cloud deployments, we can simply return an error and block the user until explicitly added by an organization admin, or we can create an organization, group and bind the user to that group.
This specification covers the latter option and defines any external interface contracts.

## Onboarding Request

The client provides a callback URI that will be installed in the `oauth2client` record.
When a user does not exist in the system the identity service will pass the following to that URI as GET query parameters:

| Parameter | Required | Description |
|---|---|---|
| state | :white\_check\_mark: | Opaque state related to the authorization flow. This is time limited and single use. |
| callback | :white\_check\_mark: | URI to POST form data back to. |
| email | :white\_check\_mark: | User’s canonical email address.  |
| username | | User’s full name as reported by OIDC. |
| forename | | User’s forename as reported by OIDC. |
| surname | | User’s surname as reported by OIDC. |

That should be enough to perform any KYC checks and be friendly. The request SHOULD be over HTTPS for confidentiality. 

### Example

```
HTTP/1.2 GET /onboard?state=xyz&callback=https%3A%2F%2Fidentity.domain.com/onboard&email=foo%40bar.com&username=foo
Host: ui.domain.com
```

## Onboarding Response

The client needs to respond with a HTTP Post, with `Content-Type: application/x-www-form-urlencoded` to the provided callback containing:

| Parameter | Required | Description |
|---|---|---|
| state | :white\_check\_mark: | Unaltered state as provided by the initial redirect. |
| organization\_name | :white\_check\_mark: | User provided organization name requested by the user, must be a valid DNS name. As uniqueness is handled by GUIDs and all RBAC done via GUIDs, there is no restriction on uniqueness of the name. |
| organization\_description | | User provided description or the Organization |
| organization\_tags | | Client provided, space separated list of tags to apply to the organization, individual elements are defined as `key:value` fields |
| group\_name | :white\_check\_mark: | Client provided group to create. |
| group\_description | | Client provided group description. |
| roles | | Client provided, space separated list of role names to bind the user to via the group. |

The identity service will then create the requested resources for the user.
The user record will remain in the pending state until an optional webhook returns success. 

### Example

```
HTTP/1.2 POST /onboard
Content-Type: application/x-www-form-urlencoded
Host: identity.domain.com

state=xyz&organization_name=foo&organization_tags=foo%3Abar+baz%3Aball&group_name=foo&roles=administrator
```

## Webhook

This provides integration with 3rd party services for billing etc. 

The webhook will contain a JSON payload describing the new user and organization:

```json
{
  “email”: “meow@cat.com”,
  “username”: “Fluffy Pussykins”, 
  “forename”: “Fluffy”, 
  “surname”: “Pussykins”, 
  "organizationName": "meow",
  “organizationID”: “7374-27482-18385”,
  “organizationUserID”: “8462-28596-11630”
}
```

Names are optional depending on available identity information as described for the onboarding Request callback. 

A HTTP bearer token may be provided as additional authentication.
HTTPS SHOULD be used for confidentiality. 

### Quotas and Resource Use

Allowing an anonymous public user onto the platform, with a default quota set, is basically granting free use of resource, and that costs money.
Unless you are philanthropic crazy person, you don't want this!

The webhook gives a perfect opportunity to set quotas to zero, and only once payment has been collected should the quotas be elevated to a suitable level.

## Onboarding Completion

Upon successful account creation the onboarding endpoint will issue an authorization code to the redirect URI of the client.
This can be immediately exchanged for an access token and access to the UI granted.
