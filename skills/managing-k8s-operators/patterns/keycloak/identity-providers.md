# Keycloak: Identity Providers

## Contents

- OIDC provider
- SAML provider
- Social providers
- Provider mappers
- Brokering patterns

## OIDC Provider

Configure an external OIDC identity provider (e.g., corporate Azure AD, Okta, another Keycloak). These examples show realm JSON structure — in practice, store secrets via sealed-secrets or external-secrets, not inline.

```yaml
realm:
  identityProviders:
    - alias: corporate-idp
      providerId: oidc
      enabled: true
      trustEmail: true
      firstBrokerLoginFlowAlias: first broker login
      config:
        authorizationUrl: "https://idp.example.com/authorize"
        tokenUrl: "https://idp.example.com/token"
        userInfoUrl: "https://idp.example.com/userinfo"
        clientId: keycloak-client
        clientSecret: "<secret>"
        defaultScope: "openid profile email"
        syncMode: FORCE
        validateSignature: "true"
        useJwksUrl: "true"
        jwksUrl: "https://idp.example.com/.well-known/jwks.json"
```

Key settings:
- `trustEmail: true` — skip email verification for users from this IdP
- `syncMode: FORCE` — update user attributes on every login (vs `IMPORT` for first-login only)
- `firstBrokerLoginFlowAlias` — flow to run when user logs in via this IdP for the first time

### OpenID Connect Discovery

If the IdP supports OIDC discovery, you can use the discovery URL instead of individual endpoints:

```yaml
config:
  discoveryUrl: "https://idp.example.com/.well-known/openid-configuration"
  clientId: keycloak-client
  clientSecret: "<secret>"
```

## SAML Provider

```yaml
realm:
  identityProviders:
    - alias: saml-idp
      providerId: saml
      enabled: true
      trustEmail: true
      config:
        singleSignOnServiceUrl: "https://idp.example.com/saml/sso"
        singleLogoutServiceUrl: "https://idp.example.com/saml/slo"
        nameIDPolicyFormat: "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress"
        postBindingAuthnRequest: "true"
        postBindingResponse: "true"
        wantAuthnRequestsSigned: "true"
        validateSignature: "true"
        signingCertificate: "<base64-encoded-cert>"
```

### IdP Metadata Import

Instead of configuring individual URLs, import SAML metadata:

```yaml
config:
  importFromUrl: "https://idp.example.com/saml/metadata"
  # or paste metadata XML inline:
  # importFromUrl: ""
  # samlXmlMetadata: "<EntityDescriptor>...</EntityDescriptor>"
```

### Assertion Mapping

```yaml
realm:
  identityProviderMappers:
    - name: email-mapper
      identityProviderAlias: saml-idp
      identityProviderMapper: saml-user-attribute-idp-mapper
      config:
        syncMode: FORCE
        user.attribute: email
        attribute.name: "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress"
    - name: first-name-mapper
      identityProviderAlias: saml-idp
      identityProviderMapper: saml-user-attribute-idp-mapper
      config:
        syncMode: FORCE
        user.attribute: firstName
        attribute.name: "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname"
```

## Social Providers

### Google

```yaml
identityProviders:
  - alias: google
    providerId: google
    enabled: true
    trustEmail: true
    config:
      clientId: "<google-oauth-client-id>"
      clientSecret: "<google-oauth-client-secret>"
      defaultScope: "openid profile email"
      hostedDomain: "example.com"  # restrict to organization domain
```

### GitHub

```yaml
identityProviders:
  - alias: github
    providerId: github
    enabled: true
    config:
      clientId: "<github-oauth-app-client-id>"
      clientSecret: "<github-oauth-app-client-secret>"
      defaultScope: "user:email"
```

### Microsoft / Azure AD

```yaml
identityProviders:
  - alias: microsoft
    providerId: microsoft
    enabled: true
    trustEmail: true
    config:
      clientId: "<azure-ad-app-client-id>"
      clientSecret: "<azure-ad-app-client-secret>"
      defaultScope: "openid profile email"
      tenantId: "<azure-tenant-id>"  # restrict to specific tenant
```

## Provider Mappers

Mappers transform IdP claims/attributes into Keycloak user attributes and roles.

### Attribute Importer

```yaml
identityProviderMappers:
  - name: department-mapper
    identityProviderAlias: corporate-idp
    identityProviderMapper: oidc-user-attribute-idp-mapper
    config:
      syncMode: FORCE
      claim: department
      user.attribute: department
```

### Hardcoded Role Mapper

Assign a role to all users from a specific IdP:

```yaml
identityProviderMappers:
  - name: default-role
    identityProviderAlias: corporate-idp
    identityProviderMapper: hardcoded-role-idp-mapper
    config:
      syncMode: INHERIT
      role: user
```

### Claim-Based Role Mapper

```yaml
identityProviderMappers:
  - name: admin-role-mapper
    identityProviderAlias: corporate-idp
    identityProviderMapper: oidc-role-idp-mapper
    config:
      syncMode: FORCE
      claim: groups
      claim.value: platform-admins
      role: admin
```

### Username Template Mapper

```yaml
identityProviderMappers:
  - name: username-mapper
    identityProviderAlias: corporate-idp
    identityProviderMapper: oidc-username-idp-mapper
    config:
      syncMode: FORCE
      template: "${CLAIM.preferred_username}@corporate"
```

## Brokering Patterns

### First Login Flow

Controls what happens when a user logs in via an IdP for the first time:

- **Auto-create user** — creates a local Keycloak user automatically
- **Link existing account** — matches by email and links to existing user
- **Review profile** — asks user to fill in missing profile fields

The default `first broker login` flow handles most cases. Customize for stricter account linking.

### Account Linking

Allow users to link multiple IdPs to one Keycloak account:

```yaml
realm:
  identityProviders:
    - alias: corporate-idp
      config:
        syncMode: FORCE
      firstBrokerLoginFlowAlias: first broker login
      postBrokerLoginFlowAlias: ""  # optional post-login flow
```

### Forced Identity Provider (kc_idp_hint)

Skip the Keycloak login page and redirect directly to an IdP:

```
https://keycloak.example.com/realms/my-realm/protocol/openid-connect/auth?
  client_id=my-app&
  redirect_uri=https://app.example.com/callback&
  response_type=code&
  scope=openid&
  kc_idp_hint=corporate-idp
```

Configure default IdP in realm settings to auto-redirect all login requests to a specific provider.