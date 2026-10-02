# How SAML works

SAML 2.0 lets a user log in once at an **Identity Provider (IdP)** and then be
signed in to other applications, the **Service Providers (SPs)**, without those
applications ever seeing the user's password.

In this plugin:

| Role | Who | Notes |
| ---- | ---- | ---- |
| IdP | Keycloak (local container) | Holds users and passwords. |
| SP | Open edX LMS | Trusts assertions from the IdP; creates/links the LMS account. |
| User agent | Your browser | Carries every message between the two. |

IdP and SP never talk to each other during a login. All messages travel through
the browser as redirects and form posts. The only direct (server-to-server)
call is fetching the IdP's metadata, done ahead of time.

## Core concepts

- **Entity ID**: a unique name for an IdP or SP. Here the SP entity ID is
  `KEYCLOAK_SP_ENTITY_ID` (default `openedx`) and it is also the **client ID** in Keycloak.
  The IdP entity ID is `<KEYCLOAK_URL>/realms/<KEYCLOAK_REALM>`.
- **Metadata**: an XML document describing an entity: its entity ID, endpoints
  (URLs) and public certificates.
  - IdP metadata: `<KEYCLOAK_URL>/realms/<realm>/protocol/saml/descriptor`
    (fetched by `keycloak-setup`).
  - SP metadata: `<LMS>/auth/saml/metadata.xml`.
- **AuthnRequest**: SP asks the IdP to authenticate the user.
- **Assertion / Response**: signed XML from the IdP stating who the user is,
  plus **attributes** (`firstName`, `lastName`, `email`,
  `attr_user_permanent_id`).
- **ACS (Assertion Consumer Service)**: the SP URL that receives the response.
  Here: `<LMS>/auth/complete/tpa-saml/`.
- **NameID**: the identifier of the user inside the assertion. Open edX uses
  the attribute configured as `attr_user_permanent_id` as the stable user ID,
  not the NameID.
- **Signatures and certificates**: the IdP signs the assertion with its private
  key; the SP verifies it with the public certificate from the IdP metadata.
  This is what makes the assertion trustworthy, since it arrives through the
  untrusted browser.

## SP-initiated login flow (what we test)

```plaintext
Browser                       LMS (SP)                     Keycloak (IdP)
   |  GET /auth/login/tpa-saml/?idp=keycloak                      |
   |------------------------------>|                              |
   |  302 to IdP SSO URL + AuthnRequest                           |
   |<------------------------------|                              |
   |  GET /realms/<realm>/protocol/saml?SAMLRequest=...           |
   |------------------------------------------------------------->|
   |  login form (skipped if a Keycloak session already exists)   |
   |<-------------------------------------------------------------|
   |  username + password                                         |
   |------------------------------------------------------------->|
   |  HTML form with signed SAMLResponse, auto-submitted to ACS   |
   |<-------------------------------------------------------------|
   |  POST /auth/complete/tpa-saml/  (SAMLResponse)               |
   |------------------------------>|                              |
   |                  verify signature, read attributes           |
   |                  find/create LMS user, start session         |
   |  302 to dashboard             |                              |
   |<------------------------------|                              |
```

Steps in the LMS (`third_party_auth` pipeline) after the response arrives:

1. Verify the signature against the certificate in the IdP metadata.
2. Map attributes to LMS fields using the provider config
   (`attr_first_name=firstName`, `attr_last_name=lastName`,
   `attr_email=email`, `attr_user_permanent_id=attr_user_permanent_id`).
3. Look up an existing link (`UserSocialAuth`) by the permanent ID; if none,
   create/link a user (registration form skipped because
   `skip_registration_form` is set for this provider).
4. Log the user in.

*IdP-initiated* login (starting from Keycloak, no AuthnRequest) also exists
but is not what this setup targets.

## How this maps to the plugin

| Piece | Where |
| ------- | ------- |
| Keycloak realm, SAML client, attribute mappers, test user | `templates/keycloak/apps/keycloak/realm.json` |
| Keycloak container | `patches/local-docker-compose-dev-services` |
| LMS SP config (`SAMLConfiguration`) + IdP provider (`SAMLProviderConfig`) + metadata pull | `templates/keycloak/tasks/lms/init`, run with `tutor dev do keycloak-setup` |

The Keycloak user ID is released to the SP as the SAML
attribute `attr_user_permanent_id` by a mapper on the client.

## Things that commonly go wrong

- **Entity ID mismatch**: the SP entity ID in the LMS `SAMLConfiguration` must
  equal the Keycloak client ID, or Keycloak rejects the request.
- **ACS / redirect URI mismatch**: the client must allow the LMS ACS URL
  exactly (scheme, host and port).
- **LMS cannot reach the IdP**: the metadata fetch runs inside the LMS container, so
  the Keycloak hostname must resolve there too (handled by a compose alias).
- **`roles_list` client scope**: it breaks the OneLogin library Open edX uses,
  so the Keycloak client has no default client scopes.
- **Clock skew**: assertions are valid for a short window; large clock
  differences between containers cause "assertion expired" errors.
- **Stale metadata**: after recreating Keycloak its signing keys change; run
  `tutor dev do keycloak-setup` again.

Debugging: decode any `SAMLRequest`/`SAMLResponse` with
<https://www.samltool.io/>; set `debug_mode` on the provider config to log
full XML in the LMS.
