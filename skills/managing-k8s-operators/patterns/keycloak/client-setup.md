# Keycloak: Client Setup

## Contents

- Client types
- Client configuration
- Common patterns
- Client scopes
- Service accounts

## Client Types

### Confidential Client (Backend)

Server-side applications that can securely store a client secret.

```yaml
realm:
  clients:
    - clientId: backend-api
      enabled: true
      protocol: openid-connect
      publicClient: false
      clientAuthenticatorType: client-secret
      secret: "<generated-secret>"
      standardFlowEnabled: false         # no browser login
      serviceAccountsEnabled: true       # machine-to-machine
      directAccessGrantsEnabled: false
      redirectUris: []
      webOrigins: []
```

### Public Client (SPA / Mobile)

Browser or mobile apps that cannot store secrets securely.

```yaml
realm:
  clients:
    - clientId: frontend-spa
      enabled: true
      protocol: openid-connect
      publicClient: true
      standardFlowEnabled: true
      directAccessGrantsEnabled: false
      redirectUris:
        - "https://app.example.com/*"
        - "http://localhost:3000/*"
      webOrigins:
        - "https://app.example.com"
        - "http://localhost:3000"
      attributes:
        pkce.code.challenge.method: S256  # require PKCE
```

Always enable PKCE (`S256`) for public clients — prevents authorization code interception.

### Resource Server (API)

API that only validates tokens, never initiates login. Use a confidential client with all flows disabled:

```yaml
realm:
  clients:
    - clientId: resource-api
      enabled: true
      protocol: openid-connect
      publicClient: false
      standardFlowEnabled: false
      directAccessGrantsEnabled: false
      serviceAccountsEnabled: false
```

The API validates tokens using the realm's JWKS endpoint (`/realms/<realm>/protocol/openid-connect/certs`).

## Client Configuration

### Redirect URIs

```yaml
redirectUris:
  - "https://app.example.com/callback"
  - "https://app.example.com/silent-renew"
```

Avoid wildcards in production (`https://app.example.com/*`). Use exact paths for each callback endpoint.

### Protocol Mappers

Add custom claims to tokens:

```yaml
clients:
  - clientId: backend-api
    protocolMappers:
      # add user's groups to the token
      - name: group-membership
        protocol: openid-connect
        protocolMapper: oidc-group-membership-mapper
        config:
          claim.name: groups
          full.path: "false"
          id.token.claim: "true"
          access.token.claim: "true"
          userinfo.token.claim: "true"
      # add audience claim
      - name: audience
        protocol: openid-connect
        protocolMapper: oidc-audience-mapper
        config:
          included.client.audience: resource-api
          id.token.claim: "false"
          access.token.claim: "true"
```

## Common Patterns

### SPA + API Backend

```yaml
clients:
  # public client for the SPA
  - clientId: frontend-spa
    publicClient: true
    standardFlowEnabled: true
    redirectUris: ["https://app.example.com/*"]
    webOrigins: ["https://app.example.com"]
    defaultClientScopes: ["openid", "profile", "email", "api-access"]
    attributes:
      pkce.code.challenge.method: S256

  # resource server for the API
  - clientId: backend-api
    publicClient: false
    standardFlowEnabled: false
    serviceAccountsEnabled: false
```

The SPA authenticates users and sends the access token to the API. The API validates the token against Keycloak's JWKS endpoint.

### Machine-to-Machine (Client Credentials)

```yaml
clients:
  - clientId: batch-processor
    publicClient: false
    serviceAccountsEnabled: true
    standardFlowEnabled: false
    directAccessGrantsEnabled: false
    secret: "<secret>"
```

```bash
# obtain token
curl -X POST https://keycloak.example.com/realms/my-realm/protocol/openid-connect/token \
  -d "grant_type=client_credentials" \
  -d "client_id=batch-processor" \
  -d "client_secret=<secret>"
```

## Client Scopes

Scopes control which claims appear in tokens.

### Creating Custom Scopes

```yaml
realm:
  clientScopes:
    - name: api-access
      protocol: openid-connect
      description: "Access to the backend API"
      attributes:
        include.in.token.scope: "true"
        display.on.consent.screen: "true"
        consent.screen.text: "Access your data via the API"
      protocolMappers:
        - name: audience
          protocol: openid-connect
          protocolMapper: oidc-audience-mapper
          config:
            included.client.audience: backend-api
            access.token.claim: "true"
```

### Default vs Optional Scopes

```yaml
clients:
  - clientId: frontend-spa
    defaultClientScopes:    # always included in token
      - openid
      - profile
      - email
    optionalClientScopes:   # only if explicitly requested
      - api-access
      - offline_access
```

Optional scopes must be requested in the authorization request: `scope=openid profile api-access`

## Service Accounts

Service accounts enable machine-to-machine auth via client credentials grant.

### Assigning Roles to Service Accounts

```yaml
realm:
  clients:
    - clientId: batch-processor
      serviceAccountsEnabled: true
  # assign realm roles to the service account
  users:
    - username: service-account-batch-processor
      enabled: true
      serviceAccountClientId: batch-processor
      realmRoles:
        - admin
      clientRoles:
        resource-api:
          - manage-data
```

The service account username follows the pattern `service-account-<clientId>`.
