# Runbook de pruebas de MITTA SSO

## 1. Reglas de prueba

- Usar una ventana InPrivate nueva para cada flujo integral.
- No reutilizar callbacks, códigos ni transacciones anteriores.
- No copiar tokens, transactionKey, codeVerifier, state o nonce.
- Consultar DynamoDB con proyecciones seguras.
- Probar una fase antes de continuar con la siguiente.

## 2. Prueba local

Componentes:

```text
Mitta:         http://localhost:3000
Aplicación B:  http://localhost:3001
cognito-api:   http://localhost:3002
```

### Levantar cognito-api

```powershell
cd "D:\Analytics Latam\Mitta\cognito-api"
$env:AWS_PROFILE="Mitta"
$env:AWS_REGION="us-east-1"
npm.cmd run dev
```

### Levantar MITTA

```powershell
cd "D:\Analytics Latam\Mitta\CL.Mitta.Cognito\frontend"
npm.cmd run dev
```

### Levantar Aplicación B

```powershell
cd "D:\Analytics Latam\Mitta\mitta-sso-demo"
npm.cmd run dev
```

### Flujo esperado

```text
localhost:3001
-> localhost:3000/auth/sso
-> localhost:3002/sso/initialize
-> login
-> localhost:3002/sso/finalize
-> localhost:3001/auth/callback
-> localhost:3002/sso/exchange
-> aplicación autenticada
```

## 3. Prueba híbrida

Componentes:

```text
Mitta:         https://develop.d1ttj8ckmgmipf.amplifyapp.com
Aplicación B:  http://localhost:3001
cognito-api:   https://21lkw3c1wg.execute-api.us-east-1.amazonaws.com
```

### Estados HTTP esperados

```text
initialize              201
start-authentication    200
authenticated           200
finalize                201
exchange                200
```

### Verificación en Network

Desde Amplify:

```text
POST /sso/initialize
POST /sso/transactions/:id/start-authentication
POST /sso/transactions/:id/authenticated
POST /sso/finalize
```

Desde localhost:3001:

```text
POST /sso/exchange
```

## 4. Validación del build

### MITTA

```powershell
cd "D:\Analytics Latam\Mitta\CL.Mitta.Cognito\frontend"
npx.cmd tsc --noEmit
Remove-Item -Recurse -Force -ErrorAction SilentlyContinue .next
npm.cmd run build
```

### cognito-api

No tiene build TypeScript. Usar:

```powershell
node --check .\app.js
node --check .\src\routes\sso.js
node --check .\src\services\sso.service.js
node --check .\src\services\sso-transactions.service.js
node --check .\src\services\sso-jwt.service.js
```

## 5. Pruebas de initialize

### Positiva

Esperado:

```text
201
status CREATED
version 1
expiresIn 1800
```

### Negativas

```text
Aplicación desconocida -> 403 application_not_available
Ambiente desconocido -> 403 application_not_available
Scope no permitido -> 400 invalid_scope
PKCE plain -> 400 invalid_request
Challenge inválido -> 400 invalid_request
Origen no permitido -> 403
```

## 6. Pruebas de transiciones

```text
CREATED -> AUTHENTICATING -> AUTHENTICATED
version 1 -> 2 -> 3
```

Negativas:

```text
Clave incorrecta -> 403 invalid_transaction_key
Transición repetida -> 409 invalid_transaction_status
Versión o estado concurrente -> 409 transaction_conflict
```

## 7. Prueba de finalize

Esperado:

```text
AUTHENTICATED -> HANDOFF_READY
version 3 -> 4
handoffCreatedAt con valor
```

Body permitido:

```text
transactionId
transactionKey
idToken
accessToken
refreshToken
tokenType
expiresIn
```

Datos prohibidos en el body:

```text
callbackUrl
cognitoClientId
state
nonce
codeChallenge
scope
```

## 8. Prueba de exchange

Estado actual compatible:

```text
handoff -> EXCHANGED
transaction -> HANDOFF_READY
```

Estado final después de la fase pendiente:

```text
handoff -> EXCHANGED
transaction -> COMPLETED versión 5
```

Pruebas negativas:

```text
codeVerifier incorrecto
code reutilizado
code expirado
applicationId incorrecto
transacción en estado incorrecto
```

## 9. Consultas seguras DynamoDB

### Transacciones

```powershell
aws dynamodb scan `
  --table-name mitta-sso-transactions `
  --region us-east-1 `
  --profile Mitta `
  --query "Items[].{Status:status.S,Version:version.N,CreatedAt:createdAt.N,HandoffCreatedAt:handoffCreatedAt.N}" `
  --output table
```

### Handoffs

```powershell
aws dynamodb scan `
  --table-name mitta-sso-handoffs `
  --region us-east-1 `
  --profile Mitta `
  --query "Items[].{Status:status.S,ApplicationId:applicationId.S,CreatedAt:createdAt.N,ExpiresAt:expiresAt.N,UsedAt:usedAt.N}" `
  --output table
```

## 10. Criterio de prueba integral final

```text
Aplicación dinámica registrada por API
Cognito App Client independiente
initialize 201
estados 1 a 5
finalize dinámico
exchange atómico
sesión final
suspensión efectiva
revocación efectiva
sin cambios ni despliegues centrales
```
