# Infraestructura AWS de MITTA SSO

## 1. Datos generales

- Región: `us-east-1`
- Cuenta AWS: `918475455761`
- Perfil AWS CLI local: `Mitta`
- User Pool: `us-east-1_gPaIQxnif`

> Este documento no debe contener claves, contraseñas, tokens, PAT, Client Secrets ni claves de cifrado.

## 2. Cognito

### User Pool central

El User Pool central administra todos los usuarios del ecosistema MITTA.

### App Clients conocidos

- `mitta-sso-demo`: `2s0bts67dfi7ojv5g0flb1u00l`
- Cliente administrativo: `6u14j7viik0cnk3mvm6g52463o`

### Política para nuevos App Clients

- Tipo public client.
- Sin Client Secret.
- PKCE S256.
- Callbacks exactas por ambiente.
- Logout URLs exactas por ambiente.
- Flujos mínimos requeridos:
  - `ALLOW_USER_PASSWORD_AUTH`
  - `ALLOW_USER_SRP_AUTH`
  - `ALLOW_REFRESH_TOKEN_AUTH`
- Proveedores según la aplicación:
  - `COGNITO`
  - `Google`
  - `SignInWithApple`
- Permisos de lectura requeridos:
  - `sub`
  - `email`
  - `email_verified`
  - `given_name`
  - `family_name`
  - `custom:RUT`

## 3. Amplify

- Aplicación: `Mitta-Cognito`
- App ID: `d1ttj8ckmgmipf`
- Rama de prueba: `develop`
- URL: `https://develop.d1ttj8ckmgmipf.amplifyapp.com`

### Variables públicas de la rama develop

```text
NEXT_PUBLIC_SSO_INITIALIZE_URL=https://21lkw3c1wg.execute-api.us-east-1.amazonaws.com/sso/initialize
NEXT_PUBLIC_SSO_FINALIZE_URL=https://21lkw3c1wg.execute-api.us-east-1.amazonaws.com/sso/finalize
NEXT_PUBLIC_SSO_COGNITO_REGION=us-east-1
```

Las variables se configuraron con AWS CLI porque la consola no mostraba la sección correspondiente.

### Comandos de consulta

```powershell
aws amplify get-branch `
  --app-id d1ttj8ckmgmipf `
  --branch-name develop `
  --region us-east-1 `
  --profile Mitta `
  --query "branch.environmentVariables" `
  --output json
```

Después de modificar variables `NEXT_PUBLIC_*`, es obligatorio reconstruir y desplegar la rama.

## 4. API Gateway

- URL base: `https://21lkw3c1wg.execute-api.us-east-1.amazonaws.com`
- Health: `/sso/health`
- Documentación protegida: `/login-docs`

La ruta `/login-docs` no es el prefijo operativo de la API.

### Rutas operativas

```text
/sso/initialize
/sso/transactions/:transactionId/start-authentication
/sso/transactions/:transactionId/authenticated
/sso/finalize
/sso/exchange
/admin/sso/applications
```

## 5. Lambda cognito-api

Variables relevantes, solo nombres:

```text
AWS_REGION
USER_POOL_ID
CLIENT_ID
SSO_APPLICATIONS_TABLE
SSO_TRANSACTIONS_TABLE
SSO_HANDOFF_TABLE
SSO_TRANSACTION_TTL_SECONDS
SSO_HANDOFF_TTL_SECONDS
SSO_TOKEN_ENCRYPTION_KEY
MITTA_FRONTEND_ORIGIN
SSO_ALLOWED_ORIGINS
SSO_ADMIN_VIEWER_GROUP
SSO_ADMIN_MANAGER_GROUP
SSO_ADMIN_GROUP
```

### Configuración para la prueba híbrida

```text
MITTA_FRONTEND_ORIGIN=https://develop.d1ttj8ckmgmipf.amplifyapp.com
SSO_ALLOWED_ORIGINS=https://develop.d1ttj8ckmgmipf.amplifyapp.com,http://localhost:3001
```

Sin slash final en `MITTA_FRONTEND_ORIGIN`.

## 6. DynamoDB

### 6.1 mitta-sso-applications

Claves:

```text
PK: applicationId
SK: entityKey
```

Índice:

```text
origin-index
PK: originKey
SK: applicationId
```

Elementos:

```text
METADATA
ENV#local
ENV#development
ENV#qa
ENV#production
```

No usa TTL.

### 6.2 mitta-sso-transactions

Clave:

```text
PK: transactionId
```

TTL:

```text
ttl
```

Duración actual:

```text
1800 segundos
```

### 6.3 mitta-sso-handoffs

Clave:

```text
PK: codeHash
```

TTL:

```text
ttl
```

Duración actual:

```text
60 segundos
```

## 7. Aplicación B de prueba

- URL local: `http://localhost:3001`
- Callback: `http://localhost:3001/auth/callback`
- Environment lógico: `local`
- Application ID: `mitta-sso-demo`

### Configuración híbrida local

```text
NEXT_PUBLIC_MITTA_SSO_URL=https://develop.d1ttj8ckmgmipf.amplifyapp.com/auth/sso
NEXT_PUBLIC_MITTA_EXCHANGE_URL=https://21lkw3c1wg.execute-api.us-east-1.amazonaws.com/sso/exchange
NEXT_PUBLIC_MITTA_APPLICATION_ID=mitta-sso-demo
NEXT_PUBLIC_MITTA_SSO_ENVIRONMENT=local
NEXT_PUBLIC_MITTA_SSO_SCOPES=openid email profile rut
NEXT_PUBLIC_APP_CALLBACK_URL=http://localhost:3001/auth/callback
```

## 8. CORS requerido para prueba híbrida

Orígenes:

```text
https://develop.d1ttj8ckmgmipf.amplifyapp.com
http://localhost:3001
```

Métodos mínimos:

```text
GET
POST
OPTIONS
```

Headers mínimos:

```text
Content-Type
Authorization
```

## 9. Comprobaciones operativas

### Health

```powershell
Invoke-RestMethod `
  -Method Get `
  -Uri "https://21lkw3c1wg.execute-api.us-east-1.amazonaws.com/sso/health"
```

### Tabla de transacciones, proyección segura

```powershell
aws dynamodb scan `
  --table-name mitta-sso-transactions `
  --region us-east-1 `
  --profile Mitta `
  --query "Items[].{Status:status.S,Version:version.N,CreatedAt:createdAt.N,HandoffCreatedAt:handoffCreatedAt.N}" `
  --output table
```

### Tabla de handoffs, proyección segura

```powershell
aws dynamodb scan `
  --table-name mitta-sso-handoffs `
  --region us-east-1 `
  --profile Mitta `
  --query "Items[].{Status:status.S,ApplicationId:applicationId.S,CreatedAt:createdAt.N,ExpiresAt:expiresAt.N,UsedAt:usedAt.N}" `
  --output table
```
