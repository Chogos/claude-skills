# Keycloak: Customization

## Contents

- Custom themes
- Custom providers (SPIs)
- Custom image
- Configuration via environment variables

## Custom Themes

Themes control the look and feel of login, account, email, and admin pages.

### Theme Directory Structure

```text
themes/
  my-theme/
    login/
      theme.properties
      resources/
        css/
          custom.css
        img/
          logo.png
      login.ftl
      template.ftl
    account/
      theme.properties
    email/
      theme.properties
      messages/
        messages_en.properties
```

### theme.properties

```properties
# inherit from the default keycloak theme
parent=keycloak
import=common/keycloak

# override styles
styles=css/login.css css/custom.css
```

### Deploying Themes via ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: keycloak-theme
  namespace: keycloak-system
data:
  theme.properties: |
    parent=keycloak
    import=common/keycloak
    styles=css/login.css css/custom.css
  custom.css: |
    .login-pf body { background-color: #f0f0f0; }
    #kc-header-wrapper { font-size: 24px; }
```

Mount into the Keycloak pod:

```yaml
spec:
  unsupported:
    podTemplate:
      spec:
        volumes:
          - name: theme
            configMap:
              name: keycloak-theme
        containers:
          - volumeMounts:
              - name: theme
                mountPath: /opt/keycloak/themes/my-theme/login
```

Override specific FreeMarker `.ftl` templates by placing them in the theme directory. Only override templates you need to change — inherit the rest from the parent theme.

## Custom Providers (SPIs)

Keycloak's Service Provider Interface (SPI) allows extending functionality with custom Java code.

### Common SPIs

| SPI | Purpose | Example |
|-----|---------|---------|
| Authenticator | Custom authentication step | CAPTCHA, hardware token |
| Event Listener | React to login/admin events | Audit logging, webhook |
| User Storage | Federate users from external store | Legacy database, LDAP extension |
| Protocol Mapper | Custom token claims | Dynamic claim from external API |
| Required Action | Post-login action | Custom terms acceptance |

### Building a Custom SPI

```java
public class AuditLogFactory implements EventListenerProviderFactory {
    @Override
    public String getId() { return "audit-log"; }

    @Override
    public EventListenerProvider create(KeycloakSession session) {
        return new AuditLogProvider(session);
    }
}
```

Package as a JAR with `META-INF/services/org.keycloak.events.EventListenerProviderFactory` pointing to the factory class.

### Deploying SPIs

**Via init container:**

```yaml
spec:
  unsupported:
    podTemplate:
      spec:
        initContainers:
          - name: spi-provider
            image: registry.example.com/keycloak-spis:1.0
            command: ["cp", "/providers/audit-log.jar", "/deployments/"]
            volumeMounts:
              - name: providers
                mountPath: /deployments
        containers:
          - volumeMounts:
              - name: providers
                mountPath: /opt/keycloak/providers
        volumes:
          - name: providers
            emptyDir: {}
```

Prefer baking providers into a custom image for production (see below).

## Custom Image

Build a custom Keycloak image with themes and providers baked in:

```dockerfile
FROM quay.io/keycloak/keycloak:26.0 as builder

COPY providers/ /opt/keycloak/providers/
COPY themes/ /opt/keycloak/themes/

RUN /opt/keycloak/bin/kc.sh build

FROM quay.io/keycloak/keycloak:26.0
COPY --from=builder /opt/keycloak/ /opt/keycloak/

ENTRYPOINT ["/opt/keycloak/bin/kc.sh"]
```

Reference in the Keycloak CR:

```yaml
apiVersion: k8s.keycloak.org/v2alpha1
kind: Keycloak
metadata:
  name: keycloak
spec:
  instances: 3
  image: registry.example.com/keycloak-custom:26.0
  db:
    vendor: postgres
    host: postgres.db.svc
    usernameSecret:
      name: keycloak-db
      key: username
    passwordSecret:
      name: keycloak-db
      key: password
  hostname:
    hostname: keycloak.example.com
  http:
    tlsSecret: keycloak-tls
```

## Configuration via Environment Variables

Keycloak uses `KC_` prefixed environment variables. Available features vary by version — run `kc.sh show-config` to see current options.

```yaml
spec:
  unsupported:
    podTemplate:
      spec:
        containers:
          - env:
              # reverse proxy headers (replaces deprecated KC_PROXY)
              - name: KC_PROXY_HEADERS
                value: xforwarded
              - name: KC_HTTP_ENABLED
                value: "true"
              # database pool
              - name: KC_DB_POOL_MIN_SIZE
                value: "5"
              - name: KC_DB_POOL_MAX_SIZE
                value: "20"
              # logging
              - name: KC_LOG_LEVEL
                value: "INFO"
              # features
              - name: KC_FEATURES
                value: "organization,passkeys"
              # cache
              - name: KC_CACHE
                value: ispn
              - name: KC_CACHE_STACK
                value: kubernetes
```

Proxy configuration:

- `KC_PROXY_HEADERS: xforwarded` — trust `X-Forwarded-*` headers from reverse proxy
- `KC_HTTP_ENABLED: "true"` — enable HTTP when behind a TLS-terminating proxy
- For TLS passthrough, configure `KC_HTTPS_*` env vars instead
