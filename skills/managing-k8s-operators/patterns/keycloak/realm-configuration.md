# Keycloak: Realm Configuration

## Contents

- KeycloakRealmImport CR
- Realm settings
- Realm roles
- Authentication flows
- Realm export and backup

## KeycloakRealmImport CR

The Keycloak operator manages realms via `KeycloakRealmImport`. The CR contains a full realm JSON representation.

```yaml
# v2alpha1 is the current API version — expect changes as it matures
apiVersion: k8s.keycloak.org/v2alpha1
kind: KeycloakRealmImport
metadata:
  name: my-realm
  namespace: keycloak-system
spec:
  keycloakCRName: keycloak  # references the Keycloak CR
  realm:
    realm: my-realm
    enabled: true
    displayName: "My Application"
    registrationAllowed: false
    resetPasswordAllowed: true
    loginWithEmailAllowed: true
    duplicateEmailsAllowed: false
    sslRequired: external
    roles:
      realm:
        - name: admin
          description: "Administrator role"
        - name: user
          description: "Standard user role"
    defaultRoles:
      - user
```

The realm JSON follows the standard Keycloak realm representation format. Export an existing realm from the admin console and use it as the basis for the CR.

### Import Reconciliation

KeycloakRealmImport performs a full realm import on each reconciliation. To avoid overwriting manual changes:
- Define the complete desired state in the CR
- Avoid mixing CR-managed and manually-configured settings in the same realm
- Use a dedicated realm for operator-managed configuration

## Realm Settings

### Login Settings

```yaml
realm:
  registrationAllowed: false        # disable self-registration
  registrationEmailAsUsername: true  # email is the username
  resetPasswordAllowed: true
  loginWithEmailAllowed: true
  verifyEmail: true                 # require email verification
  rememberMe: true
  editUsernameAllowed: false
```

### Token Lifespans

```yaml
realm:
  accessTokenLifespan: 300            # 5 minutes
  accessTokenLifespanForImplicitFlow: 900
  ssoSessionIdleTimeout: 1800         # 30 minutes
  ssoSessionMaxLifespan: 36000        # 10 hours
  offlineSessionIdleTimeout: 2592000  # 30 days
  refreshTokenMaxReuse: 0             # 0 = unlimited reuse
```

Short-lived access tokens (5 min) with longer refresh tokens is the standard pattern. Adjust based on security requirements.

### Password Policy

```yaml
realm:
  passwordPolicy: "length(12) and upperCase(1) and lowerCase(1) and specialChars(1) and digits(1) and notUsername and passwordHistory(5)"
```

Policy elements:
- `length(N)` — minimum length
- `upperCase(N)` / `lowerCase(N)` / `digits(N)` / `specialChars(N)` — character requirements
- `notUsername` — password cannot be the username
- `passwordHistory(N)` — cannot reuse last N passwords
- `hashIterations(N)` — PBKDF2 iterations (default 210000)

### Brute Force Protection

```yaml
realm:
  bruteForceProtected: true
  permanentLockout: false
  maxFailureWaitSeconds: 900          # 15 min max lockout
  minimumQuickLoginWaitSeconds: 60
  waitIncrementSeconds: 60
  quickLoginCheckMilliSeconds: 1000
  maxDeltaTimeSeconds: 43200          # 12 hours
  failureFactor: 5                    # lock after 5 failures
```

## Realm Roles

### Role Definition

```yaml
realm:
  roles:
    realm:
      - name: admin
        description: "Full administrative access"
        composite: true
        composites:
          realm:
            - manage-users
            - manage-clients
      - name: manager
        description: "Team manager"
      - name: user
        description: "Standard user"
  defaultRoles:
    - user
```

Composite roles bundle multiple roles into one. Users assigned `admin` automatically get `manage-users` and `manage-clients`.

### Client Roles

```yaml
realm:
  clients:
    - clientId: my-app
      roles:
        - name: app-admin
          description: "Application admin"
        - name: app-viewer
          description: "Read-only access"
```

## Authentication Flows

### Custom Browser Flow

```yaml
realm:
  authenticationFlows:
    - alias: custom-browser
      description: "Browser flow with conditional OTP"
      providerId: basic-flow
      topLevel: true
      builtIn: false
      authenticationExecutions:
        - authenticator: auth-cookie
          authenticatorFlow: false
          requirement: ALTERNATIVE
          priority: 10
        - authenticator: auth-username-password-form
          authenticatorFlow: false
          requirement: ALTERNATIVE
          priority: 20
  browserFlow: custom-browser  # set as the active browser flow
```

### Required Actions

```yaml
realm:
  requiredActions:
    - alias: CONFIGURE_TOTP
      name: "Configure OTP"
      providerId: CONFIGURE_TOTP
      enabled: true
      defaultAction: false
    - alias: TERMS_AND_CONDITIONS
      name: "Terms and Conditions"
      providerId: TERMS_AND_CONDITIONS
      enabled: true
      defaultAction: true  # all new users must accept
```

## Realm Export and Backup

### Export via Admin CLI

```bash
# exec into the Keycloak pod
kubectl exec -n keycloak-system deploy/keycloak -- \
  /opt/keycloak/bin/kc.sh export \
  --dir /tmp/export \
  --realm my-realm \
  --users skip  # skip user data for portability

# copy export out
kubectl cp keycloak-system/<pod>:/tmp/export/my-realm-realm.json ./realm-export.json
```

### Partial Export (Admin API)

```bash
# port-forward to Keycloak
kubectl port-forward -n keycloak-system svc/keycloak 8443:8443

# get admin token
TOKEN=$(curl -s -k https://localhost:8443/realms/master/protocol/openid-connect/token \
  -d "grant_type=client_credentials&client_id=admin-cli&client_secret=<secret>" \
  | jq -r .access_token)

# export realm (without secrets)
curl -s -k -H "Authorization: Bearer $TOKEN" \
  https://localhost:8443/admin/realms/my-realm > realm-export.json
```

### Backup Strategy

- Store realm JSON in Git as KeycloakRealmImport CRs
- Export periodically for disaster recovery (includes runtime state the CR may not capture)
- Back up the database if users are managed in Keycloak (not federated)