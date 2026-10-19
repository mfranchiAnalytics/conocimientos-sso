# Contratos de API de MITTA SSO

## 1. Convenciones

- JSON para requests y responses.
- `Cache-Control: no-store` en respuestas sensibles.
- Errores controlados:

```json
{
  "error": "codigo_error",
  "message": "Descripción segura",
  "details": {}
}
```

- Nunca devolver stack traces, tokens ni secretos.

## 2. API administrativa

Todas las rutas requieren Access Token administrativo válido.

### 2.1 Crear aplicación

```http
POST /admin/sso/applications
Authorization: Bearer ACCESS_TOKEN
Content-Type: application/json
```

Ejemplo:

```json
{
  "applicationId": "mitta-inventario",
  "name": "MITTA Inventario",
  "description": "Sistema de inventario",
  "cognitoClientId": "APP_CLIENT_ID",
  "allowedScopes": ["openid", "email", "profile", "rut"],
  "allowedProviders": ["COGNITO"],
  "environments": {
    "local": {
      "callbackUrl": "http://localhost:4000/auth/callback",
      "logoutUrl": "http://localhost:4000",
      "allowedOrigin": "http://localhost:4000",
      "enabled": true
    }
  }
}
```

Respuesta: `201`, estado inicial `DRAFT`.

### 2.2 Listar aplicaciones

```http
GET /admin/sso/applications?status=ACTIVE&limit=25&cursor=...
```

### 2.3 Consultar aplicación

```http
GET /admin/sso/applications/:applicationId
```

### 2.4 Actualizar aplicación

```http
PUT /admin/sso/applications/:applicationId
```

Usa `version` para bloqueo optimista.

### 2.5 Cambiar estado

```http
PATCH /admin/sso/applications/:applicationId/status
```

```json
{
  "version": 2,
  "status": "ACTIVE",
  "reason": "Configuración validada"
}
```

### 2.6 Archivar

```http
DELETE /admin/sso/applications/:applicationId
```

Archivado lógico, no eliminación física.

### 2.7 Revalidar Cognito

```http
POST /admin/sso/applications/:applicationId/validate-cognito
```

```json
{
  "version": 1
}
```

## 3. Inicialización SSO

```http
POST /sso/initialize
Content-Type: application/json
```

Request:

```json
{
  "applicationId": "mitta-sso-demo",
  "environment": "local",
  "state": "OPAQUE_STATE",
  "nonce": "OPAQUE_NONCE",
  "codeChallenge": "PKCE_CHALLENGE",
  "codeChallengeMethod": "S256",
  "scope": "openid email profile rut"
}
```

Response `201`:

```json
{
  "transactionId": "UUID",
  "transactionKey": "SECRET_TEMPORAL",
  "application": {
    "applicationId": "mitta-sso-demo",
    "name": "MITTA SSO Demo",
    "description": "Aplicación de referencia",
    "cognitoClientId": "PUBLIC_CLIENT_ID",
    "callbackUrl": "http://localhost:3001/auth/callback",
    "logoutUrl": "http://localhost:3001",
    "allowedOrigin": "http://localhost:3001",
    "allowedProviders": ["COGNITO"],
    "pkce": {
      "required": true,
      "allowedMethods": ["S256"]
    }
  },
  "acceptedScopes": ["openid", "email", "profile", "rut"],
  "status": "CREATED",
  "expiresAt": 0,
  "expiresIn": 1800
}
```

`transactionKey` nunca debe incluirse en una URL o log.

## 4. Iniciar autenticación

```http
POST /sso/transactions/:transactionId/start-authentication
Content-Type: application/json
```

```json
{
  "transactionKey": "SECRET_TEMPORAL"
}
```

Response `200`:

```json
{
  "transactionId": "UUID",
  "status": "AUTHENTICATING",
  "version": 2,
  "updatedAt": 0
}
```

## 5. Marcar autenticación completa

```http
POST /sso/transactions/:transactionId/authenticated
Content-Type: application/json
```

```json
{
  "transactionKey": "SECRET_TEMPORAL",
  "userSub": "COGNITO_SUB_OPCIONAL"
}
```

Response `200`:

```json
{
  "transactionId": "UUID",
  "status": "AUTHENTICATED",
  "version": 3,
  "updatedAt": 0
}
```

## 6. Finalize dinámico

```http
POST /sso/finalize
Content-Type: application/json
```

Request reducido:

```json
{
  "transactionId": "UUID",
  "transactionKey": "SECRET_TEMPORAL",
  "idToken": "JWT",
  "accessToken": "JWT",
  "refreshToken": "TOKEN_OPCIONAL",
  "tokenType": "Bearer",
  "expiresIn": 3600
}
```

No enviar:

```text
applicationId
environment
callbackUrl
cognitoClientId
state
nonce
codeChallenge
codeChallengeMethod
scope
tokenExpiresAt
```

Response `201`:

```json
{
  "redirectUrl": "http://localhost:3001/auth/callback?code=...&state=...",
  "expiresIn": 60
}
```

## 7. Exchange

Contrato actual/final esperado:

```http
POST /sso/exchange
Content-Type: application/json
```

```json
{
  "code": "HANDOFF_CODE",
  "codeVerifier": "PKCE_VERIFIER",
  "applicationId": "mitta-sso-demo"
}
```

Response `200`:

```json
{
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "openid email profile rut",
  "id_token": "JWT",
  "access_token": "JWT",
  "refresh_token": "TOKEN_OPCIONAL"
}
```

El exchange atómico definitivo está pendiente.

## 8. Health

```http
GET /sso/health
```

```json
{
  "service": "mitta-sso-bridge",
  "status": "ok"
}
```

## 9. Códigos de error relevantes

```text
unauthorized
invalid_token
forbidden
invalid_request
invalid_scope
application_not_available
initialize_origin_denied
invalid_transaction_id
invalid_transaction_key
sso_transaction_not_found
sso_transaction_expired
invalid_transaction_status
invalid_transaction_transition
transaction_conflict
invalid_cognito_tokens
transaction_user_mismatch
application_configuration_changed
invalid_grant
version_conflict
invalid_status_transition
application_already_exists
```
